### sync.waitGroup

***

#### 0、结构体
```
type WaitGroup struct {
	noCopy noCopy       // sync包下的特殊标记，vet检查时若有拷贝会报错

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage for the sema.
	state1 [3]uint32    // 存放任务计数器和等待者计数器，waiter-等待者计数，counter-任务计数，sema-信号量
                        //          state[0]   state[1]   state[2]
                        // 64       waiter     counter    sema
                        // 32       sema       waiter     counter
}

// noCopy may be embedded into structs which must not be copied after the first use.
// See https://golang.org/issues/8005#issuecomment-190753527 for details.
type noCopy struct{}
```

#### 1、添加
```
// Add adds delta, which may be negative, to the WaitGroup counter.
// If the counter becomes zero, all goroutines blocked on Wait are released.
// If the counter goes negative, Add panics.
//
// Note that calls with a positive delta that occur when the counter is zero
// must happen before a Wait. Calls with a negative delta, or calls with a
// positive delta that start when the counter is greater than zero, may happen
// at any time.
// Typically this means the calls to Add should execute before the statement
// creating the goroutine or other event to be waited for.
// If a WaitGroup is reused to wait for several independent sets of events,
// new Add calls must happen after all previous Wait calls have returned.
// See the WaitGroup example.
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state() // 获取wg状态
	if race.Enabled {           // 数据竞态检测，默认false
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}
	state := atomic.AddUint64(statep, uint64(delta)<<32)   // 原子操作，为计数器加上delta的值
	v := int32(state >> 32)                                // 获取任务计数器值（高32位）
	w := uint32(state)                                     // 获取等待者计数器值（低32位）
	if race.Enabled && delta > 0 && v == int32(delta) {
		// The first increment must be synchronized with Wait.
		// Need to model this as a read, because there can be
		// several concurrent wg.counter transitions from 0.
		race.Read(unsafe.Pointer(semap))
	}
	if v < 0 {                                             // 任务计数器不可为负数
		panic("sync: negative WaitGroup counter")
	}
	if w != 0 && delta > 0 && v == int32(delta) {          // 在此次add之前，已调用过wait方法
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	if v > 0 || w == 0 {                                   // 无等待者或任务未完成，直接返回
		return
	}
	// This goroutine has set counter to 0 when waiters > 0.
	// Now there can't be concurrent mutations of state:
	// - Adds must not happen concurrently with Wait,
	// - Wait does not increment waiters if it sees counter == 0.
	// Still do a cheap sanity check to detect WaitGroup misuse.
	if *statep != state {                                  // 有等待者但数据仍在变更
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
	*statep = 0
	for ; w != 0; w-- {                                    // 重置状态，并向所有等待者发出信号，告知任务已完成
		runtime_Semrelease(semap, false, 0)
	}
}

// state returns pointers to the state and sema fields stored within wg.state1.
// 取出state1中存储的状态，statep为计数器状态，semap为信号量
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```

#### 2、删除
```
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

#### 3、等待
```
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()                 // 获取wg状态
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)      // 用atomic的LoadUint64来保证写操作已完成
		v := int32(state >> 32)                 // 获取任务计数器值（高32位）
		w := uint32(state)                      // 获取等待者计数器值（低32位）
		if v == 0 {                             // 没有任务直接return
			// Counter is 0, no need to wait.
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// Increment waiters count.
		if atomic.CompareAndSwapUint64(statep, state, state+1) {    // 使用cas操作，若不等，说明已被修改了状态，则等待者+1后再次进入for循环
			if race.Enabled && w == 0 {
				// Wait must be synchronized with the first Add.
				// Need to model this is as a write to race with the read in Add.
				// As a consequence, can do the write only for the first waiter,
				// otherwise concurrent Waits will race with each other.
				race.Write(unsafe.Pointer(semap))
			}
			runtime_Semacquire(semap)                               // 等待信号量
			if *statep != 0 {                                       // 收到信号但状态不是0，证明wait之后又被调用了add，则panic
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
```

#### 4、总结
waitGroup需要指针传递。
调用wait方法之后，等待期间不可再add，否则会panic。但任务完成，waiter收到信号之后，可以再次调用add方法。
wg可以多次调用wait方法，任务完成后会通知所有waiter。
通过load和cas（Compare-and-Swap）操作+循环来避免锁。