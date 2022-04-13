# kubeproxy
- 负责转发流量给service对应的ip：port，并实现部分负载均衡的作用
- 替代品有：cluster DNS

## 概念
- iptable
- ipvs
- 
##  proxy mode
- ProxyModeUserspace   ProxyMode = "userspace" ： 即将被废弃
-	ProxyModeIPTables    ProxyMode = "iptables" ： 更快
-	ProxyModeIPVS        ProxyMode = "ipvs" ： 更快
-	ProxyModeKernelspace ProxyMode = "kernelspace"

### userspace
### iptables
### ipvs
### kernelspace
