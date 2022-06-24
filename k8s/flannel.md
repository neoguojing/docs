# flannel

- pod内部，通过loop网口相互访问
- pod到docker0： 使用路由表转发
- docker0-flannel0：使用路由表转发
- 跨设备转发：
- > 目的ip为其他node的pod的ip
- > 目的mac为其他node的flannel0的mac地址
- > mac地址在arp cache中查找，找不到，则内核向用户空间发送L3 miss事件；flanneld会查询etcd获取对应node的flannel0的mac地址;内层数据包封装：dst ip为对端pod ip；dst mac为对端flanal网卡地址
- > 通过fdb（转发表）获取对端node的ip和mac，不存在则触发L2 miss，flanneld会查询etcd获取对端node ip和mac；外层封装：dst分别为node ip和mac
- eth0检测到是vxLan包，则发送到flannel0网卡
- flannel0-docker0：查询路由表
- docker0-pod：通过arp查看ip对应的mac，进行二层转发
