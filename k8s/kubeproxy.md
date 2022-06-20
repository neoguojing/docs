# kubeproxy
> 因为kube-proxy是以daemonSet的形式部署在所有节点上的，所以每个节点都会有相同的iptable规则，当任何一个节点上的pod访问service时，其实都是可以在该pod所在的node的的iptable中找到对应的service规则从而找到service所代理的pod的，而对于node而言，寄宿在自己上的pod的发出的流量就是从本机的某进程出去的流量。

- 目前仅支持 TCP 和 UDP，不支持 HTTP 路由，并且也没有健康检查机制。这些可以通过自定义 Ingress Controller 的方法来解决。
- 负责转发流量给service对应的ip：port，并实现部分负载均衡的作用
- 它监听 API server 中 service 和 endpoint 的变化情况，并通过 iptables 等来为服务配置负载均衡
- kube-proxy 可以直接运行在物理机上，也可以以 static pod 或者 daemonset 的方式运行
- 替代品有：cluster DNS
- watch apiserver，当监听到pod 或service变化时，修改本地的iptables规则或ipvs规则；
- dns使用的是：coredns
- dummy网卡：ip link add kube-ipvs0 type dummy 类似loopback 接口，但是可以创建任意多个
- DefaultDummyDevice = "kube-ipvs0"：1.主机内通信；2.将service ip绑定到该设备，防止物理接口变动影响服务；
- > 实际不会接收报文，可以看到它的网卡状态是 DOWN，主要用于绑 ipvs 规则的 VIP
- > kube-ipvs0 这个网卡将 Service 相关的 VIP 绑在上面以便让报文进入 INPUT 进而被 ipvs 转发
- > 当 IP 被绑到 kube-ipvs0 上，内核会自动将上面的 IP 写入 local 路由
- hairpin mode：bridge不允许包从收到包的端口发出，比如bridge从一个端口收到一个广播报文后，会将其广播到所有其他端口。bridge的某个端口打开hairpin mode后允许从这个端口收到的包仍然从这个端口发出；brctl hairpin <bridge> <port> {on|off} turn hairpin on/off
- > 解决网络同一个网络命名空间内的通信问题，从同一个端口进入，同一个端口出去
## 通用
- Netlink套接字是用以实现用户进程与内核进程通信的一种特殊的进程间通信(IPC) ,也是网络应用程序与内核通信的最常用的接口。
- unix.Socket(unix.AF_NETLINK, unix.SOCK_RAW|unix.SOCK_CLOEXEC,unix.NETLINK_GENERIC)
##  proxy mode
- ProxyModeUserspace   ProxyMode = "userspace" ： 即将被废弃
-	ProxyModeIPTables    ProxyMode = "iptables" ： 更快
-	ProxyModeIPVS        ProxyMode = "ipvs" ： 更快
-	ProxyModeKernelspace ProxyMode = "kernelspace"
### winuserspace：
- 仅在windows使用
### userspace
- 应用发往Service的请求会通过iptable规则转发给kube-proxy，
- kube-proxy再转发到Service所代理的后端Pod
- 发往Service的请求会先进入内核空间的iptable，再回到用户空间由kube-proxy代理转发。内核空间和用户空间来回地切换成为了该模式的主要性能问题。但由于发往后端Pod的请求是由kube-proxy代理转发的，请求失败时，是可以让kube-proxy重试的
### iptables 只支持概率轮询的负载均衡
- https://www.zsythink.net/archives/1199
- 整个过程全部发生在内核空间，提高了转发性能
- ptable的规则是基于链表实现的，规则数量随着Service数量的增加线性增加，查找时间复杂度为O(n)。当Service数量到达一定量级时，CPU消耗和延迟增加显著。
- https://zhuanlan.zhihu.com/p/196393839
- 表：承载链条，实现不同功能,按照优先级排列如下,通过-t参数指定
- > security表：为包指定SELinux 标记
- > raw：实现数据跟踪；只有两个链条PREROUTING，OUTPUT
- > mangle: 包修改;5个链条均有
- > nat : 网络地址转换；除了除了foward的其他链均可
- > filter ： 包过滤，防火墙的主要功能实现；input，output，FORWARD
- 用-t 执行表，不指定默认使用filter表
- 链条：PREROUTING，INPUT ， FORWARD ，OUTPUT，POSTROUTING，使用-A（末尾）/-I（在最开始）等指定
- 自定义chain进行规则分类：iptable -t nat -N KUBE-MARK-DROP
- 规则：-d/-s 执行ip/域名；-dport/-sport指定端口，-j指定ACCEPT/DROP，-i执行接口,-p 指定协议；规则按照从上到下的优先级，最上面规则匹配到，则下层规则失效；
- SNAT：对ip报文的源地址做转换；家里网络访问公网资源，经过路由器时，内网地址192.XXX被转换为公网地址；
- DNAT：对IP报文的目的地址做转换转换公网地址为内网地址
- MARK功能可以用于标记网络数据包，用于标记数据包。在一些不同的table或者chain之间需要协同处理某一个数据包时尤其有用
- iptables restore：回复备份的配置
#### 操作
- return:返回上一个chain，然后执行下一条规则
- accept：
- MASQUERADE：地址伪装；自动读取网卡地址，实现自动化的SNAT转换
### ipset
- 集合类型：hash-可自动扩容（最大65535）；bitmap，list固定大小
- 数据类型：ip-支持ip地址和地址段，net：网络地址；ip,port
  
```
bitmap:ip
bitmap:ip,mac
bitmap:port
hash:mac
hash:ip,mac
hash:net,net
hash:net,port
hash:ip,port,ip
hash:ip,mark
hash:net,port,net
hash:net,iface
list:set
```
  
- 采用增量式更新，并可以保证 service 更新期间连接保持不断开
- linux 命令，建立资源集合，如IP
- iptable -m set --match-set 使用set
- -m 也可以指定为ipvs
- KUBE-CLUSTER-IP All service IP + port Mark-Masq for cases that masquerade-all=true or clusterCIDR specified
- KUBE-LOOP-BACK All service IP + port + IP masquerade for solving hairpin purpose
- KUBE-EXTERNAL-IP service external IP + port masquerade for packages to external IPs
- KUBE-LOAD-BALANCER load balancer ingress IP + port masquerade for packages to load balancer type service
- KUBE-LOAD-BALANCER-LOCAL LB ingress IP + port with externalTrafficPolicy=local accept packages to load balancer with externalTrafficPolicy=local
- KUBE-LOAD-BALANCER-FW load balancer ingress IP + port with loadBalancerSourceRanges package filter for load balancer with loadBalancerSourceRanges specified
- KUBE-LOAD-BALANCER-SOURCE-CIDR load balancer ingress IP + port + source CIDR package filter for load balancer with loadBalancerSourceRanges specified
- KUBE-NODE-PORT-TCP nodeport type service TCP port masquerade for packets to nodePort(TCP)
- KUBE-NODE-PORT-LOCAL-TCP nodeport type service TCP port with externalTrafficPolicy=local accept packages to nodeport service with externalTrafficPolicy=local
- KUBE-NODE-PORT-UDP nodeport type service UDP port masquerade for packets to nodePort(UDP) 
- KUBE-NODE-PORT-LOCAL-UDP nodeport type service UDP port withexternalTrafficPolicy=local accept packages to nodeport service withexternalTrafficPolicy=local
### ipvs 工作在INPUT，只做DNAT，4层负载均衡，ingress负责7层转发
- ipvsadm用户态命令行
- ipvs工作在内核态
- https://zhuanlan.zhihu.com/p/94418251
- IPVS 模式也会使用 iptables 来执行 SNAT 和 IP 伪装（MASQUERADE）以及包过滤，并使用 ipset 来简化 iptables 规则的管理
- https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md 
- 基于lvs：linux虚拟服务器，丰富的负载均衡功能，以及直接再内核态转发数据的功能
- 核心概念：DS-负载均衡节点，对应的IP是DIP；VIP：一般是指DS的虚拟IP；
- 核心流程：流量进入DS，经过PREROUTING，因为是本机的VIP，所以到达INPUT；在INPUT中，使用负载均衡技术选择一个IP，修改目的地址为后端服务IP和端口；经过POSTROUTING转发到后端服务；后端服务处理完后会经过POSTROUTING->FORWARD->PREROUTING转发给client；
- 三种模式：
- > NAT：ip层的修改，流量大时损耗大；入口流量和出口流量都要经过负载均衡节点；k8s需要使用这个模式做端口映射
- > DR：修改MAC地址，二层转发，要求一个以太网，效率最高；LB端口和RS端口要一致；无法做端口映射
- > tun：隧道在ip层嵌套一层；
- 负载均衡算法：轮询；加权轮询；最少连接；加权最少连接；
- 操作使用ipvsadm，用户态管理ipvs的命令行：功能包括：1.创建VIP；2.为VIP添加后端服务地址
- Service就是一个负载均衡实例，而server就是后端member,ipvs术语中叫做real server，简称RS
- /proc/pid/task/tid/ns/net : 网络空间文件地址
- ipvs的负载是直接运行在内核态的，因此不会出现监听端口
#### 工作流程
- 首先创建kube-ipvs0虚拟网卡，将所有的service ip（cluster ip）绑定道该网卡，是虚拟ip
- 
#### 前置条件 /proc/sys
- net/ipv4/conf/all/route_localnet：启动127/8地址作为源或者目的地址
- net/bridge/bridge-nf-call-iptables：二层的网桥在转发包时也会被iptables的FORWARD（三层）规则所过滤,解决linux网桥（二层）回包无法使用conntrack（三层）的问题
- net.ipv4.vs.conntrack：开启conntrack
- net/ipv4/vs/conn_reuse_mode= 0：设置端口可以重用 
- net/ipv4/vs/expire_nodest_conn=1：后端服务不可用，则直接结束调连接；
- net/ipv4/vs/expire_quiescent_template=1:调度的后端服务器权重为0会立即使持久连接过期，并被发送到新的服务器上
- net/ipv4/ip_forward=1：
- 如下规则：ipvsDR模式，需要在lo上设置虚拟ip，此时需要隐藏网卡，因此设置如下规则
- net/ipv4/conf/all/arp_ignore=1:只响应目的IP地址为接收网卡上的本地地址的arp请求
- net/ipv4/conf/all/arp_announce=2:忽略IP数据包的源IP地址，选择该发送网卡上最合适的本地地址作为arp请求的源IP地址
- 
#### ipvs的优雅关闭问题：短连接，高并发的情况下的延时问题：

### conntrack 工作在三层网络
- 连接跟踪表：记录每个连接的源ip，目的ip和端口的等信息；回报时匹配规则进行自动snat
- 解决问题：当snat做转发发送到公网；然后返回时，路由器如何知道发给哪个PC？
- /proc/sys/net/netfilter/nf_conntrack_max：设置最大记录的连接数个数
- /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established：设置建立连接的超时时间
- /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_close_wait：设置连接关闭等待时间
### kernelspace

## kube控制链
![来源于cilium iptable规则](https://github.com/cilium/k8s-iptables-diagram/blob/master/kubernetes_iptables.svg)
![ipvs rules]( https://github.com/neoguojing/docs/blob/main/k8s/controller-kubelet-iptable-rule.drawio.png) 

### 规则解释
- 转发规则：匹配到%#08x/%#08x标记，则接受；否则匹配conn，存储连接状态则接受
- PREROUTINGG和OUTPUT会启用KUBE-SERVICE规则链：
- > 匹配到KUBE-LOAD-BALANCER集合(dst,dst)则转到KUBE-LOAD-BALANCER链 // 对于load balancer type service
- > 匹配到KUBE-CLUSTER-IP集合(dst,dst)则转到标记链打标
- > 匹配到KUBE-EXTERNAL-IP集合则转到标记链打标   //指定external IPs的service才有
- > 目的地址是本地主机地址的，跳转到KUBE-NODE-PORT链
- POSTROUTING启用KUBE-POSTROUTING规则：
- > 匹配到KUBE-LOOP-BACK集合（dst，dst，src）则执行MASQ //处理hairpin模式
- > 未打标的包返回上一个链条处理,标记了0x4000/0x4000的数据包继续向下执行标记和snat
- > 对标记进行xor操作
- > 进行snat，且端口随机化
- KUBE-LOAD-BALANCER：仅对于load balancer type service
- > 匹配到KUBE-LOAD-BALANCER-FW集合则跳转到KUBE-FIREWALL链条
- > 匹配到KUBE-LOAD-BALANCER-LOCAL集合则返回到KUBE-SERVICE链处理  //仅externalTrafficPolicy=local才出现
- > 跳转到打标链
- KUBE-FIREWALL：
- > 匹配KUBE-LOAD-BALANCER-SOURCE-CIDR则返回KUBE-LOAD-BALANCER链； //指定了loadBalancerSourceRanges 的服务
- > 匹配KUBE-LOAD-BALANCER-SOURCE-IP则返回KUBE-LOAD-BALANCER链；
- > 跳转到标记drop链条
- KUBE-NODE-PORT:
- > 匹配到KUBE-NODE-PORT-LOCAL-TCP集合则返回KUBE-SERVICE继续处理 //仅externalTrafficPolicy=local才出现
- > 匹配到KUBE-NODE-PORT-TCP集合则跳转到标记链
- > UDP同上
- KUBE-MARK-DROP
- > 被标记了0x8000/0x8000的数据包都会被直接DROP
### syncProxyRules
- 获取本地ip地址集合
- 统计无用的service集合
- createAndLinkeKubeChain：创建和链接规则链
- > iptable -t nat -N KUBE-MARK-DROP  //创建丢弃自定义链
- > iptable -t nat -N KUBE-SERVICES
- > iptable -t nat -N KUBE-POSTROUTING
- > iptable -t nat -N KUBE-FIREWALL
- > iptable -t nat -N KUBE-NODE-PORT
- > iptable -t nat -N KUBE-LOAD-BALANCER
- > iptable -t nat -N KUBE-MARK-MASQ
- > iptable -t filter -N KUBE-FORWARD 
- 创建和获取kube-ipvs0
- 依据ipsetInfo所有的ipset
- 判断是否有nodeport创建，有则找到所有nodeip
- 遍历proxier.serviceMap(svcName,svc)，构建service规则
- > 遍历endpointsMap[svcName]，将endpoint放入KUBE-LOOP-BACK的ipset
- > 将svc信息放入KUBE-CLUSTER-IP
- > 调用syncService构建虚拟服务,成功则调用syncEndpoint构建真实服务
- > 遍历svcInfo.ExternalIPStrings，若只是本地endpoint，则放入KUBE-EXTERNAL-IP-LOCAL集合，否则放入KUBE-EXTERNAL-IP（需要经过filter链做转发）,调用syncService和syncEndpoint构建ipvs服务
- > 遍历svcInfo.LoadBalancerIPStrings，同上，放入KUBE-LOAD-BALANCER或者KUBE-LOAD-BALANCER-LOCAL或者KUBE-LOAD-BALANCER-FW或者KUBE-LOAD-BALANCER-SOURCE-CIDR集合并构建ipvs服务
- > 遍历nodepoint集合，添加到nodeport相关ipset，并构建ipvs服务
- 调用ipset命令同步ipset配置，重新设置相关的集合
- writeIptablesRules：刷写基于ipset的规则
- > iptable -t nat -I OUTPUT -j KUBE-SERVICES   -m comment --comment  //
- > iptable -t nat -I PREROUTING -j KUBE-SERVICES   -m comment --comment
- > iptable -t nat -I POSTROUTING -j KUBE-POSTROUTING   -m comment --comment
- > iptable -t filter -I FORWARD -j KUBE-FORWARD   -m comment --comment
- > iptable -A KUBE-POSTROUTING -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE 
- > iptable -A KUBE-SERVICES  -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER 
- > iptable -A KUBE-LOAD-BALANCER   -m set --match-set KUBE-LOAD-BALANCER-FW dst,dst -j KUBE-FIREWALL
- > iptable -A KUBE-FIREWALL   -m set --match-set KUBE-LOAD-BALANCER-SOURCE-CIDR dst,dst,src -j RETURN
- > iptable -A KUBE-FIREWALL   -m set --match-set KUBE-LOAD-BALANCER-SOURCE-IP dst,dst,src -j RETURN
- > iptable -A KUBE-LOAD-BALANCER   -m set --match-set KUBE-LOAD-BALANCER-LOCAL dst,dst -j RETURN
- > iptable -A KUBE-NODE-PORT -m set --match-set KUBE-NODE-PORT-LOCAL-TCP dst -p tcp -j RETURN
- > iptable -A KUBE-NODE-PORT -m set --match-set KUBE-NODE-PORT-TCP dst -p tcp -j KUBE-MARK-MASQ
- > iptable -A KUBE-NODE-PORT -m set --match-set KUBE-NODE-PORT-LOCAL-UDP dst -p udp -j RETURN
- > iptable -A KUBE-NODE-PORT -m set --match-set KUBE-NODE-PORT-UDP dst -p udp -j KUBE-MARK-MASQ
- > iptable -A KUBE-SERVICES  -m set --match-set KUBE-CLUSTER-IP dst,dst -j KUBE-MARK-MASQ
- > iptable -A KUBE-SERVICES  -m set --match-set KUBE-EXTERNAL-IP dst,dst -j KUBE-MARK-MASQ
- > iptable -A KUBE-SERVICES  -m addrtype  --dst-type LOCAL -j KUBE-NODE-PORT //addrtype地址类型匹配模块
- > iptable -A KUBE-LOAD-BALANCER  -j KUBE-MARK-MASQ
- > iptable -A KUBE-FIREWALL  -j KUBE-MARK-DROP
- > iptable -A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst -j ACCEPT
- > iptable -A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst -j ACCEPT
- > iptable -A KUBE-FORWARD -m mark --mark masqueradeMark/masqueradeMark -j ACCEPT
- > iptable -A KUBE-FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
- > iptable -A KUBE-POSTROUTING -m mark !--mark masqueradeMark/masqueradeMark -j RETURN 
- > iptable -A KUBE-POSTROUTING -j MARK --xor-mark masqueradeMark
- > iptable -A KUBE-POSTROUTING -j MASQUERADE --random-fully
- > iptable -A KUBE-MARK-MASQ -j MARK --or-mark masqueradeMark
- cleanLegacyService：清理legacy服务
- serviceHealthServer.SyncServices和serviceHealthServer.SyncEndpoints
- conntrack.ClearEntriesForIP：清理UDP记录
- deleteEndpointConnections
### Proxier.syncService
- 调用ipvs,添加或者更新虚拟服务
- 绑定service ip到kube-ipvs0设备
### Proxier.syncEndpoint
- 调用ipvs添加新的endpoint到realserver
- 将待删除的endpoint放入gracefuldeleteManager优雅删除
## Service
- externalTrafficPolicy：外部流量策略
- > Cluster（默认）: Kubeproxy流量可以转发到其他Node的Pod,但会经过snat，回包需要从原路径返回；负载均衡会好，性能有损耗
- > Local: Kubeproxy流量只转发到本地，负载均衡差，效率高；可以通过外层的负载均衡，配合pod的反亲和性来达到目的
## core dns
