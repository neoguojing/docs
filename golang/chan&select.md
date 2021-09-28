# chan和select

## channel

### 概览
- chan中元素大小不能大于2^16()64k
- hchanSize： 96
### 函数
- makechan：
- > 对于长度为0的chan，调用mallocgc分配hchanSize的空间；c.buf指向nil
- > elem中没有指针，则mallocgc分配hchanSize+elem.size*size的连续空间；设置c.buf指向hchanSize偏移后的地址
- > 否则：new(hchan，使用mallocgc为c.buf分配空间
- > 设置elem等相关参数

## select
