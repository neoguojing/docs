# kubeproxy
- 负责转发流量给service对应的ip：port，并实现部分负载均衡的作用
- 替代品有：cluster DNS

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

### ipset
- linux 命令，建立资源集合，如IP
- iptable -m set --match-set 使用set
### ipvs
### kernelspace
