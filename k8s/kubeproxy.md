# kubeproxy
- 负责转发流量给service对应的ip：port，并实现部分负载均衡的作用
- 替代品有：cluster DNS
- watch apiserver，当监听到pod 或service变化时，修改本地的iptables规则或ipvs规则；
- dns使用的是：coredns
##  proxy mode
- ProxyModeUserspace   ProxyMode = "userspace" ： 即将被废弃
-	ProxyModeIPTables    ProxyMode = "iptables" ： 更快
-	ProxyModeIPVS        ProxyMode = "ipvs" ： 更快
-	ProxyModeKernelspace ProxyMode = "kernelspace"

### userspace
### iptables
- 表：承载链条，实现不同功能,按照优先级排列如下,通过-t参数指定
- > security表：为包指定SELinux 标记
- > raw：实现数据跟踪；只有两个链条PREROUTING，OUTPUT
- > mangle: 包修改;5个链条均有
- > nat : 网络地址转换；PREROUTING，OUTPUT，POSTROUTING
- > filter ： 包过滤，防火墙的主要功能实现；input，output，FORWARD
- 链条：PREROUTING，INPUT ， FORWARD ，OUTPUT，POSTROUTING，使用-A/-I等指定
- 规则：-d/-s 执行ip/域名；-dport/-sport指定端口，-j指定ACCEPT/DROP，-i执行接口,-p 指定协议；规则按照从上到下的优先级，最上面规则匹配到，则下层规则失效；
- SNAT：对ip报文的源地址做转换；家里网络访问公网资源，经过路由器时，内网地址192.XXX被转换为公网地址；
- DNAT：对IP报文的目的地址做转换转换公网地址为内网地址
- MASQUERADE：地址伪装；自动读取网卡地址，实现自动化的SNAT转换
### ipset
- linux 命令，建立资源集合，如IP
- iptable -m set --match-set 使用set
- -m 也可以指定为ipvs，
### ipvs 工作在INPUT，只做DNAT
- 基于lvs：linux虚拟服务器，丰富的负载均衡功能，以及直接再内核态转发数据的功能
- 核心概念：DS-负载均衡节点，对应的IP是DIP；VIP：一般是指DS的虚拟IP；
- 核心流程：流量进入DS，经过PREROUTING，因为是本机的VIP，所以到达INPUT；在INPUT中，使用负载均衡技术选择一个IP，修改目的地址为后端服务IP和端口；经过POSTROUTING转发到后端服务；后端服务处理完后会经过POSTROUTING->FORWARD->PREROUTING转发给client；
- 三种模式：NAT：ip层的修改，流量大时损耗大；DR：修改MAC地址，二层转发，要求一个以太网，效率最高；tun：隧道在ip层嵌套一层；
- 负载均衡算法：轮询；加权轮询；最少连接；加权最少连接；
- 操作使用ipvsadm，用户态管理ipvs的命令行：功能包括：1.创建VIP；2.为VIP添加后端服务地址

### conntrack
- 连接跟踪表：记录每个连接的源ip，目的ip和端口的等信息；
- 解决问题：当snat做转发发送到公网；然后返回时，路由器如何知道发给哪个PC？
### kernelspace

## 控制链
- KUBE-SERVICES
- KUBE-MARK-MASQ
- KUBE-NODE-PORT
- KUBE-FIREWALL
- KUBE-POSTROUTING
- KUBE-NODE-PORT-TCP：ipset规则
- flannel网卡

## core dns
