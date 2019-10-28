### sync.map

***
#### 1、结构体
```
type Map struct {
    mu Mutex    //互斥锁，用于锁定dirty map

    read atomic.Value //优先读map，支持原子操作

    dirty map[interface{}]*entry // 当前最新map，允许读写

    misses int // 记录读取read map失败的次数，当misses等于dirty的长度时，会将dirty复制到read
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 如果数据在dirty中但没有在read中，该值为true，作为修改标识
}

type entry struct {
    // nil: 被删除，且Map.dirty == nil
    // expunged: 被删除，且Map.dirty != nil
    // 其他: 表示存着真实数据
    p unsafe.Pointer // *interface{}
}
```

#### 2、原理
通过空间换时间即冗余两个数据结构(read/dirty)来实现。将读写分离到不同的map，read map提供并发读和已存元素的原子写，dirty map负责读写。
故而read map可以在不加锁的情况下进行并发读，当read map中未读取到时，再加锁进行读取，并累计未命中数，当未命中数大于等于dirty map长度时，
用dirty map覆盖read map。两个map的底层数据指针仍指向同一份值。

#### 3、查询
```
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {

    // 查询read map
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // read map中未读到且dirty map中有新数据，则查询dirty map
    if !ok && read.amended {
    
        // 为dirty map加锁
        m.mu.Lock()
        
        // 再次查询read map，主要防止在加锁的过程中,dirty map转换成read map,导致读取不到数据
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
        
            // 查询dirty map
            e, ok = m.dirty[key]
            
            // 不论元素是否存在，均需记录miss数，以便dirty map升级为read map
            m.missLocked()
        }
        
        // 解锁
        m.mu.Unlock()
    }
    
    // 元素不存在则返回
    if !ok {
        return nil, false
    }
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    
    // 判断dirty map是否可升级为read map
    if m.misses < len(m.dirty) {
        return
    }
    
    // dirty map升级为read map
    m.read.Store(readOnly{m: m.dirty})
    
    // 清空dirty map
    m.dirty = nil
    
    // 重置misses
    m.misses = 0
}
```

#### 4、删除
```
func (m *Map) Delete(key interface{}) {

    // 查询read map
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // read map中未读到且dirty map中有新数据，则查询dirty map，当read与dirty不同时amended为true即dirty中有read没有的新数据
    if !ok && read.amended {
        m.mu.Lock()
        
        // 再次查询read map，主要防止在加锁的过程中,dirty map转换成read map,导致读取不到数据
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
        
            // 直接删除
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    
    if ok {
    
        // 若read map中存在该key，则将其标记为nil（采用标记的方式删除！）
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        
        // 原子操作
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```

#### 5、新增/修改
```
func (m *Map) Store(key, value interface{}) {

    // 查询read map，若能查到且entry未被标记删除，则尝试更新
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    m.mu.Lock()
    
    // 再次查询read map
    read, _ = m.read.Load().(readOnly)
    
    // read map中存在key
    if e, ok := read.m[key]; ok {
    
        // 若entry被标记expunge，则表明dirty没有key，可添加并更新entry
        if e.unexpungeLocked() {
        
            // 元素之前被删除，意味着有非nil不包含元素的dirty
            m.dirty[key] = e
        }
        
        // 更新read map 元素值
        e.storeLocked(&value)
        
    // dirty map中存在key
    } else if e, ok := m.dirty[key]; ok {
    
        // 更新dirty map
        e.storeLocked(&value)
        
    // read与dirty均不存在
    } else {
    
        // read.amended==false即read与dirty相同，将read map复制一份到dirty map
        if !read.amended {
        
            // 将read中未删除的数据加入到dirty中
            m.dirtyLocked()
            
            // 设置read.amended==true
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        
        // 更新dirty map
        m.dirty[key] = newEntry(value)
    }
    
    // 解锁。因m.dirtyLocked()中有写入操作，故这么大的锁范围是有必要的
    m.mu.Unlock()
}

func (e *entry) tryStore(i *interface{}) bool {

    // 获取对应Key的元素，判断是否标识为删除
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
    
        // cas尝试写入新值
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        
        // 判断是否标识为删除
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// 将read map中标记为expunge的改为nil
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
    
        // 过滤标记为nil或者expunged的，其余复制进dirty map
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

// 判断entry是否被标记删除，并且将标记为nil的entry更新为expunge
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    
    for p == nil {
    
        // 将标记为nil的数据改为expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}



// 更新entry
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

```

#### 6、总结
sync.map不适合同时存在大量读写的场景，大量写会导致read map查不到数据从而加锁读取dirty map，进而是dirty map升级为read map，最终导致性能下降。
更适合append-only或大量读少量写场景。
