# runtime/map.go
## 概念
- 数据存储在buckets数组中，bucket本身时一个type类型；每个bucket最多包含8对k/v
- hash值的低2^B-1位，用于选择bucket，高8bit用于确定bucket内部k/v的位置
- bucket值超过8，值放入extra bucket列表中
- map增长，会新建一个2倍大小的bucket数组，动态增量的从老bucket复制过去
- evacuated：即map大小变更
- 装载因子以bucket的个数计算，而不是以elem
- 掩码： 2^B-1
- bucket选择：偏移hash & 2^B-1 * maptype.bucketsize
- elem选择： 选择hash的搞8bit，与hmap的tophash逐一比较，相等，找到key的位置，比较key的值，相等，则偏移找到elem
- 每次插入：都需要遍历8+overflow*8个
## 架构
```
// map头部
type hmap struct {
	count     int // # live cells == size of map
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

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

```
- bucketCnt: 8
- loadFactorNum = 13
- loadFactorDen = 2
- 装载因子：6.5，触发map增长
- maxKeySize  = 128 字节
- maxElemSize = 128
- minTopHash : 5
- emptyRest： 0 ，档期cell为空，且之后的entry和overflow都为空
## 函数
- evacuated：判断是否在迁移
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
- growWork
- overLoadFactor
- tooManyOverflowBuckets
- hashGrow
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