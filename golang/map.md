# runtime/map.go
## 概念
- 数据存储在buckets数组中，bucket本身时一个type类型；每个bucket最多包含8对k/v
- hash值的低8bit用于选择bucket，高8bit用于确定bucket内部k/v的位置
- bucket值超过8，值放入extra bucket列表中
- map增长，会新建一个2倍大小的bucket数组，动态增量的从老bucket复制过去
- evacuated：即map大小变更
- 装载因子以bucket的个数计算，而不是以elem
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
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
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
## 函数
- makemap_small： make(map[k]v)和make(map[k]v, <8)
- > 创建一个hmap，返回
- makemap : make(map[k]v, hint)
- > 判断是否越界
- > 若h==nil，则创建hmap，初始化hash0
- > 判断是否超载，是则B++,知道满足需要，设置B
- > 若B不为0，则新建2^B个bucket，并设置h.extra.nextOverflow
- makeBucketArray：创建bucket和overflow
- > 计算bucket个数2^B
- > 若b>=4,则需要2^(B-4)的额外空间用于overflow，计算总的bucket个数
- > 调用newarray分配bucket类型的空间
- > 若有overflow分配，
- overLoadFactor 是超过装载因子：map的size>8,且size>13*2^B/2 .实际元素个数大于6.5*2^B
