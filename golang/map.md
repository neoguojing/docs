# runtime/map.go
## 总结
- map的并发写检测：  h.flags&hashWriting != 0  通过hmap的标记判断是否有并行写的情况，是则抛出异常
- make(map[string]int)：不带大小的map在第一个写入时才创建bucket
- 每个bucket包含8个k/v
- 先k后v的存储法的好处: 相同类型的数据存一起，避免大量的pad
- 针对不同的key类型：字符串，整形，编译器会调用不同的访问函数，有hash比较的优化
- reflexive: 反射性的，k==k；在计算机里浮点数的比较是在一定范围能相等则表示相等，key类型是float32\float64\complex64\complex64\interface的或包含该类型的都是反射性key。
- 数据存储在buckets数组中，bucket本身是一个type类型；每个bucket最多包含8对k/v
- hash值的低2^B-1位，用于选择bucket，高8bit用于确定bucket内部k/v的位置;tophash的值需大于5
- bucket值超过8，值放入overflow列表中
- map增长，会新建一个2倍大小的bucket数组，动态增量的从老bucket复制过去
- evacuated：即map大小变更
- 装载因子：6.5*2^B，大于这个值，则需要进行两倍扩容；
- 掩码： 2^B-1
- bucket选择：偏移hash & 2^B-1 * maptype.bucketsize
- elem选择： 选择hash的高8bit，与hmap的tophash逐一比较，相等，找到key的位置，比较key的值，相等，则偏移找到elem
- 每次插入：都需要遍历8（选定buckcet中的8个k/v）+overflow*8个（？？？）
- map的遍历随机，是因为mapiterinit中设置了随机的起始bucket
- map创建：若B大于4，需要创建2^(B-4)个overflow bucket
- b.tophash[i] == emptyRest：表面后面的值都为空，无需遍历
- 删除key，只是将tophash的位置设置为emptyOne；并会从后往前，将最前面的一个tophash设置为emptyRest
### map扩容
#### 触发
- set时，即mapasign
#### 扩容条件
- 存储的k/v超过装载因子 : 实施2倍扩容；实际元素个数大于6.5*2^B；单bucket的元素个数超过6.5
- 溢出bucket的个数大于2^B B<=15 ： 实施等量扩容；溢出bucket的数量过多
#### 何时扩容
- 执行set的时候
- 执行delete的时候
#### 如何扩容
- 根据oldbucket 索引，定位到oldbucket；遍历bucket内的entry以及ovf，根据hash值随机的迁移的新map的x和y区域
- x区域和oldbucket的索引一致，即相对位置不变
- y区域是old索引+old所有元素的个数，做偏移
- k/v的前后顺序保持不变，但是空元素会被忽略
#### 扩容的特征
- oldbuckets ！= nil
- 迁移完成会设置oldbuckets = nil oldoverflow=nil
## 架构
```
// map头部
type hmap struct {
	count     int // # live cells == size of map
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // overflow bucket的个数
	hash0     uint32 // hash seed
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // 已经迁移的bucket个数

	extra *mapextra // optional fields
}
<!-- 数据存储，bucket的实际实现 -->
type bmap struct {
	tophash [bucketCnt]uint8 //tophash[0] < minTopHash 则表示该bucket在迁移过程中
	//紧跟8key+8value
	//紧跟overflow指针
}

<!-- reflect/type.go Map的runtime实现 -->
type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
}

type mapextra struct {
	overflow    *[]*bmap //存储所有buckets overflow指针
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

<!--  迁移目的地-->
type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

<!-- 遍历结构体 -->
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end 
	elem        unsafe.Pointer // Must be in second position 
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at；是一个整形表示偏移
	offset      uint8          // bucket内部值遍历的索引
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr   //
	checkBucket uintptr
}
```
- bucketCnt: 8
- loadFactorNum = 13
- loadFactorDen = 2
- 装载因子：6.5，触发map增长
- maxKeySize  = 128 字节
- maxElemSize = 128
- minTopHash : 5
- emptyRest： 0 ，档期cell为空，且之后的entry和overflow都为空
- evacuatedX     = 2 // k/v被疏散到新map的第一半
- evacuatedY     = 3 //k/v被疏散到新map的第二半
- evacuatedEmpty = 4 // 迁移完成
- minTopHash     = 5 // minimum tophash for a normal filled cell.
- iterator     = 1 // there may be an iterator using buckets
- oldIterator  = 2 // there may be an iterator using oldbuckets
## 函数
- makemap_small： make(map[k]v)和make(map[k]v, <8)
- > 创建一个hmap，返回
- makemap : make(map[k]v, hint)
- > 判断是否越界
- > 若h==nil，则创建hmap，初始化hash0
- > 判断是否超载，是则B++,知道满足需要，设置B
- > 若B不为0，则新建2^B个bucket，并设置h.extra.nextOverflow（下一个空闲的overflow）
- makeBucketArray：创建bucket和overflow
- > 计算bucket个数2^B
- > 若b>=4,则需要2^(B-4)的额外空间用于overflow，计算总的bucket个数
- > 调用newarray分配bucket类型的空间
- > 若有overflow分配，返回overflow的起始指针，设置最后一个bucket
- overLoadFactor 是超过装载因子：map的size>8,且size>13*2^B/2 .实际元素个数大于6.5*2^B
- mapaccess1：返回h[key]的指针
- > maptype.hasher：计算key的hash值；hash&（2^b-1[掩码]）得到lowhash,计算bucket的位置hash&（2^b-1[掩码]* maptype.bucketsize
- > 若hmap.oldbuckets不为nil,若不是等大小grow，则掩码需要除以2，获取oldbucket的位置，若不在迁移过程中，则设置待遍历的bucket为oldbucket；
- > 调用tophash，hash值左移56bit，获得一个8bit的tophash，tohash必须>5
- > bucketloop ：遍历bucket和b.overflow:
- > 遍历8个k/v：
- > 若tophash不相等，若 b.tophash[i] == emptyRest，则跳到bucketloop，否则遍历下一个kv
- > 否则tophash相等，直接找到第i个key的位置（i*maptype.keysize）
- > 若key是指针，则需要提取指针的值
- > 若key比较相等，则偏移i*maptype.keysize + i*maptype.elemsize
- > elem为指针则提取指针的值，返回elem
- > 未找到则返回空
- mapaccess2：会返回值以及true，false
- mapaccessK： 返回k/v
- bmap.overflow: bucketsize-8，获得最后一个指针的位置，即overflow的指针的位置
- mapassign：返回elem将要保存的位置
- > 计算key的hash值
- > hmap.buckets未nil，则创建一个bucket
- > again：计算bucket的lowhash
- > 若oldbucket!= nil,表示正在迁移，调用growWork做迁移
- > 否则根据lowhash，计算目标bucket的位置
- > 计算tophash
- > bucketloop for：
- > 遍历bucket内8个k
- > 若tophash不相等，且bmap.tophash[i] <=emptyOne,则保存tophash、k、v的位置信息，若tophash[i]等于emptyRest,跳出所有循环，否则 continue
- > 否则表示找到一个tophash相等的，获取key的位置，并比较key：
- > key不相等，则continue
- > key相等，则调用typedmemmove，更新key的值,获取elem的位置，跳转到done ----更新值路径
- > 遍历完8，仍没有done，则遍历overflow，为nil则break
- > 退出循环
- > 若装载因子超标或bucket过多，则hashGrow，扩容，goto again                  --扩容路径
- > 若bucket和overflow满了，则newoverflow，分配一个新的bucket，保存相关地址   -- overflow增加路径
- > 为key、elem分配空间，将key的值放入对应位置，设置tophash的值，增加计数      -- 设置值路径
- > done：返回elem的位置
- growWork: 调用evacuate，从当前set或者delete的bucket位置开始执行
- evacuate: 从索引位置开始迁移
- > 计算oldbucket中bucket的指针，计算oldbucket的bucket的个数
- > 若当前bucket未迁移：
- > 计算oldbuket对应的b，在hmap.buckets中的目的地址；在扩容2倍的情况下，迁移地址分为两部分，x:索引等于oldbucket，y:索引位置加上oldmap的bucket个数
- > 遍历old bucket以及overflow中的entry：
- > tophash为空，则直接设置b.tophash[i] = evacuatedEmpty，continue
- > 非等量扩容：非反射性key，重新计算tophash；通过top或者hash&（oldmap的大小） != 0 来设置是否迁移到y的标记
- > 设置oldbucket.tophash[i]=evacuatedX + useY，设置迁移状态
- > 获取迁移地址evacDst，若dst.i == 8，表示bucket满了，创建ovf
- > 设置dst.b.tophash为新的top值，copy k和elem的值
- > 增加dst 索引 k 和v的索引
- > 遍历结束
- > 清空ovf，辅助gc
- > 若当前bucket索引等于h.nevacuate ，则advanceEvacuationMark：增加nevacuate，设置结束标记等
- advanceEvacuationMark：增加h.nevacuate 计数，若所有oldbucket均迁移，则清空h.oldbuckets ，h.extra.oldoverflow = nil，清空标记
- newoverflow：创建ovf
- > 尝试从空闲overflow获取一个bucket，更新空闲列表指针；
- > 否则分配一个新的
- > 增加ovf计数
- > 保存ovf指针到bmap
- evacuated： 判断是否迁移完毕，通过bucket.tophash[0]判读，值大于emptyOne，小于5
- overLoadFactor: 元素个数大于bucket个数，且> 6.5 * 2^B,返回true，表示要扩容
- tooManyOverflowBuckets： noverflow >= uint16(1)<<(B&15)
- hashGrow: 
- > 若overLoadFactor返回false，执行等量扩容sameSizeGrow,设置标记
- > makeBucketArray 创建新的空间，等量扩容创建等量空间，否则扩大2倍
- > 设置hmap参数，绑定oldbuckets和buckets
- >hmap 溢出bucket不为nil，设置h.extra.oldoverflow = h.extra.overflow,h.extra.nextOverflow = nextOverflow
- typedmemmove
- mapdelete： 删除key
- > 计算hash值，设置hashWriting 写标志
- > 计算bucket位置，
- > 若在扩容，则执行growWork
- > 计算bucket指针和tophash
- >search： 遍历bucket和overflow中的元素
- > tophash不相等，则continue,若b.tophash[i] == emptyRest 则break search（跳出所有循环）
- > 若tophash相等，获取key的指针
- > key不相等，指针则设置为nil，否则清空k的值
- > 获取elem指针，指针值则设置为nil，否则清空elem的值
- > 设置tophash值为emptyOne
- > 若i为bucket最后一个entry，且overflow的tophash[0] != emptyRest，则跳转到notLast
- > 否则下一个tophash!=emptyRest,跳转到noLast
- > 循环：此时i已经为最后一个entry ，尝试从后往前设置所有emptyOne的tophash为emptyRest
- > 设置tophash[i] = emptyRest
- > i--，若b.tophash[i] != emptyOne，则退出当前循环
- > 若i==0，
- > noLast：count-1，若count为0，则重置hash0，跳出所有循环
- > 清除写标志
- mapiterinit： 初始化hiter结构体,设置随机的起始位置，包括bucket以及bucket内的offset，并调用mapiternext
- mapiternext：
- next：
- > 若it.bucket == it.startBucket且it.wrapped（从结尾反转），则表示遍历结束，设置it的key和elem，返回
- > 若正在扩容，且B没有变化，表示没有迁移完毕，则通过it.bucket，从oldbucket获取起始地址
- > 否则，从当前buckets获取起始地址
- > it.bucket++,若等于最后一个bucket的索引，则设置 it.bucket=0，it.wrapped = true
- > 遍历当前buckets
- > tophash为空，则继续
- > 获取key的地址和elem的地址
- > 若map在扩容，continue，忽略即将迁移的key？？？
- > 若b.tophash[offi] 不是疏散状态，则设置it的key和elem
- > 否则表示key已经迁移完毕，调用mapaccessK，获取当前k/v，并设置it，k为空表示已经删除；此处需要处理k被删除更新等
- > 重置bucket bptr i等值，返回
- > 获取overflow的起始地址，跳转到next
- mapclear： 清空map


## sync.Map
```
type Map struct {
    // 当涉及到脏数据(dirty)操作时候，需要使用这个锁
    mu Mutex
    
    // read是一个只读数据结构，包含一个map结构，
    // 读不需要加锁，只需要通过 atomic 加载最新的指正即可
    read atomic.Value // readOnly
    
    // dirty 包含部分map的键值对，如果操作需要mutex获取锁
    // 最后dirty中的元素会被全部提升到read里的map去
    dirty map[interface{}]*entry
    
    // misses是一个计数器，用于记录read中没有的数据而在dirty中有的数据的数量。
    // 也就是说如果read不包含这个数据，会从dirty中读取，并misses+1
    // 当misses的数量等于dirty的长度，就会将dirty中的数据迁移到read中
    misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    // m包含所有只读数据，不会进行任何的数据增加和删除操作 
    // 但是可以修改entry的指针因为这个不会导致map的元素移动
    m       map[interface{}]*entry
    
    // 标志位，如果为true则表明当前read只读map的数据不完整，dirty map中包含部分数据
    amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
    p unsafe.Pointer // *interface{}
}
```

### load ： 迁移dirty到read
- 先从read读取数据，存在则返回
- 否则，加锁二次读取read（两次确认）；
- 依然不存在，则从dirty读取；
- 无论dirty中是否存在，都更新miss值；miss值等于dirty的长度，则迁移dirty到read；迁移完dirty等于nil和miss等于0
### store 更新dirty；同时存在的情况需要都更新
- read中存在，则尝试直接跟新值；
- 不存在或者更新失败，则加锁：
- read中再检查，存在：则更新read值，若未删除则需要更新到dirty
- read中不存在，dirty中存在，则更新dirty
- read和dirty中都不存在，则将值更新到dirty；若dirty中数据数据没有新值，则从read复制未删除数据；

### delete  在read中软删除
- 若key在read中，则标记为expunged，软删除
- 不在read中，加锁，再检查一次；不存在，则在dirty中直接删除；
