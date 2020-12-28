### map

***

#### 0、概述
map底层由hash表实现，数据存储在由桶组成的有序数组中，每个桶最多存放8个键值对，key的hash值(32位)的低位用于定位数组中的桶，高8位用于定位桶中的键值对。

一旦某个桶中键值对数量超过8则触发overflow，通过extra字段对应的mapextra的overflow字段进行拓展，即申请一个新的bmap挂在当前bmap的后面形成链表，
所以其基本结构为由桶组成的数组+溢出桶链表+由键-值或状态组成的桶。优先用预分配的overflow bucket，若已用完则malloc一个新的。
需要注意的是，map的删除操作仅将对应的tophash设置为empty，并不会释放内存，故在未进行扩容(等量/增量扩容)的情况下，内存只会越用越多。

当(元素个数/bucket)>=6.5时，map会进行扩容，bucket数量*2，hash表扩容后需将老数据迁移到新表上，但迁移并非一次性完成，而是在insert和remove时完成，
同时为了避免bucket一直访问不到导致扩容无法完成，会进行一次顺序扩容，每次因写操作迁移对应bucket后，还会按顺序迁移未被迁移的bucket，故最差情况是n次写操作迁移完大小为n的map。
旧bucket迁移完成后，会被标记为已删除，当所有旧bucket均迁移完成后，内存才会被释放。

#### 1、hash算法
map的hash函数有多个，不同key类型有不同的hash算法，用来提高hash效率。hash算法用以寻找key应存入的bucket。
```
cmd/compile/internal/gc/reflect.go
func dtypesym(t *types.Type) *obj.LSym {
  switch t.Etype {
    ...
  case TMAP:
    ...

    // 依据key构建hash函数
    hasher := genhash(t.Key())
    ...
  }
}

cmd/compile/internal/gc/alg.go
func genhash(t *types.Type) *obj.LSym {
  switch algtype(t) {
  ...

  // 具体针对interface调用interhash
  case AINTER:
    return sysClosure("interhash")
  ...
  }
}

runtime/alg.go
func interhash(p unsafe.Pointer, h uintptr) uintptr {
    //获取interface p的实际类型t
	a := (*iface)(p)
	tab := a.tab
	if tab == nil {
		return h
	}
	t := tab._type
	fn := t.alg.hash

    // map中的key必须为可比较的类型
	if fn == nil {
		panic(errorString("hash of unhashable type " + t.string()))
	}
	...
}
```

#### 2、结构体
```
runtime/map.go
type hmap struct {
	count     int   // 元素数量
	
	flags     uint8 // 状态标识，主要是 goroutine 写入和扩容机制的相关状态控制，并发读写的判断条件之一
	
	B         uint8 // bucket个数，loadFactor*2^B(loadFactor=6.5)
	
	noverflow uint16 // 溢出的bucket个数
	
	hash0     uint32 // hash种子(每个map不一样，减少哈希碰撞几率)

	buckets    unsafe.Pointer // buckets数组指针
	
	oldbuckets unsafe.Pointer // 扩容时才使用，指向旧buckets的指针
	
	nevacuate  uintptr        // 扩容时copy到新table的buckets数

	extra *mapextra
}
```
hmap为由若干个bucket(即bmap)组成的数组，每个bucket存放不超过8个键值对。
hmap.B作为2的指数来表示bucket数目，之所以用2作为底，是为了方便定位bucket和进行扩容。
定位时，bucket的hash%n可转变为hash&(n-1)，进一步优化为位运算hash&(1<<B-1)((1<<B-1)即源码中BucketMask)。
扩容时，hmap.B+=1即扩容为2倍。

```
runtime/map.go
type mapextra struct {
	// If both key and value do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and value do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap // 申请的空的bucket，解决冲突时使用，用完变为nil
}
```
当map的key和value均不包含指针时，会被标记为无指针map，在gc时能避免被扫描。overflow和oldoverflow在无指针map中被使用，用于存储hmap中bucket的数据。

```
runtime/map.go
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.

    // 记录桶内8个单元的高8位hash值，或标记空桶状态，用于快速定位key
    // emptyRest      = 0 // 此单元为空，且更高索引的单元也为空
    // emptyOne       = 1 // 此单元为空
    // evacuatedX     = 2 // 用于表示扩容迁移到新桶前半段区间
    // evacuatedY     = 3 // 用于表示扩容迁移到新桶后半段区间
    // evacuatedEmpty = 4 // 用于表示此单元已迁移
    // minTopHash     = 5 // 最小的空桶标记值，小于其则是空桶标志
}

cmd/compile/internal/gc/reflect.go
type bmap struct {
	topbits  [8]uint8    // 存储tophash
	keys     [8]keytype  // 存储key
	elems    [8]elemtype // 存储value
	overflow otyp        // otyp 类型为指针*Type，若keytype及elemtype不含指针，则为uintptr，此时bmap整体不含指针，gc时不会scan此map
}

iterator     = 1 // there may be an iterator using buckets 迭代器buckets桶的标志位，为1表示正在使用buckets
oldIterator  = 2 // there may be an iterator using oldbuckets 迭代器oldbuckets桶的标志位 ，为1表示正在使用oldbuckets
hashWriting  = 4 // a goroutine is writing to the map 并发写标志位，为1表示有goroutine正在写map
sameSizeGrow = 8 // the current map growth is to a new map of the same size 等量扩容标志，表示申请的桶数量和原来一样
```
hash值的高8位存储在bucket中的tophash中，这样在查找时不用做全等判断，可以加快查询速度。bucket中存放的数据格式为key1key2...keynval1val2...valn，
这种方式能在key与value长度不同时节省padding空间。当size大于128(MAXKEYSIZE/MAXELEMSIZE)时，类型会被转为指针(indirectkey、indirectelem)。

#### 3、初始化
map初始化有两种情况，当不指定大小或指定的大小不大于8时，系统会调用makemap_small，此时直接在堆上初始化hmap和hash种子，并不初始化buckets。
当指定大小大于8时，系统会调用makemap，除了初始化hmap和hash种子外，还要根据overLoadFactor循环增加h.B，获取 hint/(1<<h.B) 最接近6.5的h.B，再预分配hashtable的bucket数组，
当h.B大于4时，多分配1<<(h.B-4)个bucket，用于可能的overflow，并将h.nextOverflow设置为第一个可用的overflow bucket，
将最后一个overflow bucket指向h.buckets(方便后续判断已无overflow bucket)。
overLoadFactor=6.5由测试得出，overLoadFactor指map平均每个bucket能装载的键值对个数，太大会导致哈希冲突的地方溢出bucket过多，太小会导致浪费空间。

```
runtime/map.go
// makemap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
func makemap_small() *hmap {
	h := new(hmap)
	h.hash0 = fastrand()
	return h
}

// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}

    // 生成hash种子
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.

    // B!=0时初始化桶指针buckets
	if h.B != 0 {
		var nextOverflow *bmap

        // 初始化桶指针buckets并分配空间
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)

            // 设置溢出桶
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

func overLoadFactor(count int, B uint8) bool {
    // 元素数量>8 && count>bucket数量*6.5
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
先根据bucketsize和hint，计算需要分配的内存大小及是否会overflow，若溢出或申请的内存大于最大可申请内存时，将hint置为0，不初始化buckets；接着和makemap_small一样，初始化一个随机种子；
然后计算B，在overLoadFactor中，若hint小于等于8，则不再赋值B，直接不初始化数据，若大于8，则增加B，使其刚好满足hint<6.5*(1<<B)；
最后，申请bucket数组，赋值给buckets，若有多的，则赋值给extra.nextOverflow。

// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    // base指用户预期的桶数量；nbuckets指实际分配的桶数量，>=base，会追加溢出桶
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
        // 申请内存，结构为数组，元素为bucket，因bmap内部由tophash/8个key/8个value/1个指针组成，tophash是数组，大小为8，类型为uint8，故总大小为8*1=8字节，
        // key/value类型为string，16字节，总大小为8*16+8*16=256字节，指针在64为系统上占8字节，故一个bucket总大小272字节。
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

#### 4、新增/修改
hmap指针传递的方式决定了map在使用前必须初始化，也不支持并发读写。
```
runtime/map.go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}

    // 竞争检查
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}

    // 并发写
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

    // hash计算
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic, in which case we have not actually done a write.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    // 用hash值低位定位数组下标便宜量
	bucket := hash & bucketMask(h.B)
	
	// 正在扩容且bucket还未迁移完成，则先迁移
	if h.growing() {
		growWork(t, h, bucket)
	}

    // 直接通过指针的偏移定位桶的位置
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))

    // 取hash值高8位定位键值对
	top := tophash(hash)

    // inserti-tophash的插入位置，insertk-key的插入位置，val-value的插入位置
	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
		    // 对比tophash，不相等则continue
			if b.tophash[i] != top {
                // 找到空位，先记录下位置，遍历完才可知是否插入此位置
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				if b.tophash[i] == emptyRest {
                    // 遍历完整个溢出链表，结束循环
					break bucketloop
				}
				continue
			}

			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			
			// 对比key
			if !alg.equal(key, k) {
				continue
			}
			
			// already have a mapping for key. Update it.
			// 已存在，则更新
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
            // 遍历完整个溢出链表，没找到插入位置，结束循环，后续追加溢出桶
			break
		}

        // 继续遍历下一个溢出桶
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// 未找到则判断是否需要扩容，需要扩容且非正在扩容，则进行扩容；若不需要扩容但没有空solt，则分配一个overflow bucket挂在链表尾部，这个bucket的第一个solt存放数据

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

    // inserti == nil说明未找到空位，桶已满，需新增一个溢出桶
	if inserti == nil {
        // 分配新溢出桶，优先用预留的，已用完则分配一个新的
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

    // 当key或value类型大小超过阈值，桶仅存储其指针，此处分配空间并取指针
	// store new key/value at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}

    // 插入key/tophash
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}

    // 释放hashWriting标志位
	h.flags &^= hashWriting
	if t.indirectvalue() {
		val = *((*unsafe.Pointer)(val))
	}

    // 返回value可插入位置的指针，需注意此时value还未插入
	return val
}
mapassign()只插入tophash和key并返回val指针，编译器会在调用mapassign()后用汇编往val插入value。
```

#### 5、扩容
触发条件：
* 元素数量过多，大于6.5*(1<<h.B)
* 溢出桶数量太多，超过1<<h.B。

扩容分为增量扩容和等量扩容，当元素数量超过6.5*(1<<h.B)时为增量扩容，分配的桶数量为原来的两倍(1<<(h.B+1)，当溢出桶数量超过1<<h.B时为等量扩容，分配的桶数量和原来相等(1<<h.B)，
扩容由hashGrow实现，主要是预分配桶空间，并不进行数据迁移。
```
runtime/map.go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}

// 仅扩容不迁移
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)

    // 若count大于bucket个数*负载因子，则bigger=1，此时会进行增量扩容，否则等量扩容
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil) // 分配桶空间

	flags := h.flags &^ (iterator | oldIterator) // 将buckets和oldbuckets迭代标志置0
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0 // 搬迁状态为0表示未进行迁移
	h.noverflow = 0

    // 当key/value不是指针时，用extramap中的指针存储溢出桶，而不用bmap中的overflow。overflow表示hmap结构buckets中的溢出桶，oldoverflow表示hmap中oldbuckets中的溢出桶，nextoverflow预分配溢出桶空间
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}

// 迁移，为了防止有bucket一直没机会被访问，此处会迁移两个bucket，入参bucket和h.nevacuate
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

// 迁移具体逻辑
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))) // 定位oldbucket
	newbit := h.noldbuckets() // 与原来旧桶分配的容量相等
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0] // 等量扩容或增量扩容的前一部分
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset) // key的地址
		x.v = add(x.k, bucketCnt*uintptr(t.keysize)) // value的地址

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1] // 若为增量扩容，需要后一部分，即增长的空间
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize))) // 后一部分桶的索引
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) { // 遍历最后一个bmap及溢出桶
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) { // 遍历桶中元素
				top := b.tophash[i] // 获取tophash
				if isEmpty(top) { // 如果tophash为空，标记为已被搬迁状态
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash { // tophash为hash+minTopHash
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8 // useY用来判断是落在oldbucket还是newbit
				if !h.sameSizeGrow() { // 如果为增量扩容，h.B增大1，桶的位置发生变化
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/value to bucket x or bucket y).
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.alg.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY 搬迁为X或者Y状态
				dst := &xy[useY]                 // evacuation destination useY=0表示搬迁到前半部分，否则到后半部分

				if dst.i == bucketCnt { // 当桶中元素数量达到最大8时，需要溢出桶
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer key为指针时，复制指针
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue() {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v) // value为指针时，复制指针
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
                // 进行下一个元素的搬迁
				dst.i++
				// These updates might push these pointers past the end of the
				// key or value arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// Unlink the overflow buckets & clear key/value to help GC.
        // 遍历完桶后，如果没有其他goroutine使用该桶，就把该桶清空
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}

// 确定桶的搬迁进度，如果搬迁完成进行后续操作
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) { // 如果搬迁没有完成将搬迁进度nevacuate加1
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil // 搬迁完成，将oldbuckets置nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil // 溢出桶置为nil
		}
		h.flags &^= sameSizeGrow // 等量扩容位设为0
	}
}
```

#### 6、查询和迭代
map查询有3种情况，区别是返回参数不同。计算出key的hash值后定位到bucket，若处于扩容中，尝试去旧bucket中取值，此时需考虑扩容前bucket大小是否为现在的一半且所指向的bucket未迁移，
然后按bucket->overflow链表的顺序遍历。
```
mapaccess1: m[k]
mapaccess2: v,ok = m[i]
mapaccessk: 在map遍历时若grow已发生，key可能有更新，需用此函数重新获取k/v

runtime/map.go
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the elem type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 竞争检查
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
    
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}

    // 检测是否并发写，因map非并发安全
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
    
    // 计算key的hash
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

    // 若旧bucket未被迁移完，则查询旧bucket
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}

    // 先比对tophash，相等再比key，当前bucket查不到，查询overflow bucket
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}

func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess2)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0]), false
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}

// returns both key and elem. Used by map iterator
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {
	if h == nil || h.count == 0 {
		return nil, nil
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return k, e
			}
		}
	}
	return nil, nil
}
```

map迭代通过hiter结构和对应的两个辅助函数实现，hiter结构由编译器在调用辅助函数前创建并传入，每次迭代结果也由hiter结构传回。
* it.startBucket：开始的桶
* it.offset：桶中的偏移量
* it.bptr：当前遍历的桶
* it.i：it.bptr已经遍历的键值对数量，i初始为0，当i=8时表示这个桶遍历完了，将it.bptr移向下一个桶
* it.key：每次迭代的key
* it.value：每次迭代的value
```
// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
	}

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}
```
mapiterinit函数决定开始迭代位置。之所以不是从hash数组头部开始，是因为hash表中数据每次插入的位置是变化的
（更直接的原因是实现上，hash种子是随机的，因此相同的数据在不同的map中其hash值是不同的；而即使同一map中，数据删除再添加的位置也有可能不同，
因为同一个桶及溢出链表中数据的位置不分先后），所以为了防止用户错误的依赖每次迭代的顺序，map作者干脆让相同的map每次迭代的顺序也是随机的。

从hash数组中第it.startBucket个桶开始，先遍历hash桶及其溢出链表，之后hash数组偏移量+1，继续前一步动作。遍历每一个桶，无论是hash桶还是溢出桶，都从it.offset偏移量开始，
当迭代器经过一轮循环回到it.startBucket的位置，则结束遍历。

```
func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket
	alg := t.key.alg

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || alg.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := alg.hash(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

#### 7、删除
当key/value过大时，hash表里存储的是指针，此时为软删除，将指针置为nil，数据由gc删除。定位key位置的方式是查找tophash，所以删除操作对tophash的处理是关键：
首先将对应位置的tophash[i]置为emptyOne，表示该位置已被删除，若tophash[i]不是整个链表的最后一个，则只置emptyOne标志，删除但未释放，后续插入操作不能使用此位置，
若tophash[i]是链表最后一个有效节点，则把链表最后面的所有为emptyOne的位置置为emptyRest。置为emptyRest的位置可以在后续的插入操作中被使用，这种删除方式，以少量空间来避免桶链表和桶内的数据移动。
```
runtime/map.go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
    	callerpc := getcallerpc()
    	pc := funcPC(mapdelete)
    	racewritepc(unsafe.Pointer(h), callerpc, pc)
    	raceReadObjectPC(t.key, key, callerpc, pc)
    }
    if msanenabled && h != nil {
    	msanread(key, t.key.size)
    }
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	
	// 若正在扩容且bucket未迁移完成，则迁移bucket
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
		
		    // 对比高8位
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			
			// 对比key
			if !alg.equal(key, k2) {
				continue
			}
			
			// Only clear key if there are pointers in it.
			// 若key、value包含指针，则将其值为nil，方便gc回收
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.kind&kindNoPointers == 0 {
				memclrHasPointers(k, t.key.size)
			}
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			if t.indirectvalue() {
				*(*unsafe.Pointer)(v) = nil
			} else if t.elem.kind&kindNoPointers == 0 {
				memclrHasPointers(v, t.elem.size)
			} else {
				memclrNoHeapPointers(v, t.elem.size)
			}

            // 标记删除，emptyOne标记的位置暂时不能被插入新元素
			b.tophash[i] = emptyOne

            // 若b.tophash[i]不是最后一个元素则先占坑
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
                // 若b.tophash[i]是最后一个元素，则将末尾的emptyOne全置为emptyRest
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```