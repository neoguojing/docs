# k8s

## opporator开发
https://zhuanlan.zhihu.com/p/246550722

## nodeport
- 流量转发给kube-proxy,kube-proxy下发路由规则给iptable，同时创建nodeport的端口监听
- 通过iptable 查看 KUBE-EXTERNAL-SERVICES，为nodeport的条目
- 高可用问题：1.pod不在这台机器，则需要转移流量；2.node机器挂掉，则表现为请求不可达，但是服务事实上可用的行为
