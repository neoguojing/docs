
# network plugin
## flannel

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

## calico
- Felix运行在每一台Host的agent进程，主要负责网络接口管理和监听、路由、ARP管理、ACL管理和同步、状态上报等。Felix会监听Etcd中心的存储，从它获取事件，比如说用户在这台机器上加了一个IP，或者是创建了一个容器等，用户创建Pod后，Felix负责将其网卡、IP、MAC都设置好，然后在内核的路由表里面写一条，注明这个IP应该到这张网卡。同样，用户如果制定了隔离策略，Felix同样会将该策略创建到ACL中，以实现隔离。
- Confd是负责存储集群etcd生成的Calico配置信息，提供给BIRD层运行时使用。
- BIRD（BIRD Internet Routing Daemon）是核心组件，Calico中的BIRD特指BIRD Client和BIRD Route Reflector，负责主动读取Felix在本机上设置的路由信息，并通过BGP广播协议在数据中心中进行分发路由
- BGP Client（BIRD）：Calico为每一台Host部署一个BGP Client，使用BIRD实现，BIRD是一个单独的持续发展的项目，实现了众多动态路由协议比如BGP、OSPF、RIP等。在Calico的角色是监听Host上由Felix注入的路由信息，然后通过BGP协议广播告诉剩余Host节点，从而实现网络互通。BIRD是一个标准的路由程序，它会从内核里面获取哪一些IP的路由发生了变化，然后通过标准BGP的路由协议扩散到整个其他的宿主机上，让外界都知道这个IP在这里，你们路由的时候得到这里来。
- .BGP Route Reflector：在大型网络规模中，如果仅仅使用BGP client形成mesh全网互联的方案就会导致规模限制，因为所有节点之间俩俩互联，需要N^2个连接，为了解决这个规模问题，可以采用BGP的Router Reflector的方法，使所有BGP Client仅与特定RR节点互联并做路由同步，从而大大减少连接数
- IPIP模式：
- BGP模式：
