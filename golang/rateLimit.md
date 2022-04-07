# 限流算法

## 固定/滑动窗口：在1s内限制100个请求；前500ms用完了限制；则后500ms无法访问；不均衡
## 令牌桶算法
- github/go/time
### 基本概念
- 桶：大小决定最大并发数
- 令牌：拿到令牌才能访问
- 速率：放入令牌的数量
- 截止时间now：表示截止某个时间期望获取多少个令牌
- maxFutureReserve ： 最大的等待时间
### 实现方法：
- 令牌数并非实时发放，而是根据截止时间和速率计算令牌数量
- 1.根据截止时间获取能够得到的令牌数量和上次拿令牌的时间
- 2.减去需要的令牌数，获取剩余令牌
- 3.剩余令牌小于0，则计算缺少的令牌产生需要等多久
- 4.若产生令牌的时间小于maxFutureReserve，则表示ok
- 5.ok：则更新最后获取时间、令牌数量和最后能够产生足够令牌的时间；放到Reservation中
### 缺点
- 令牌桶为空时，访问的频率和令牌桶放入速度一致；无法应对突发流量

## 漏斗算法 
- go.uber.org/ratelimit
- 给定速率，：可以计算每个请求到来的间隔；100/s则每个请求间隔10ms
- 维护一个时间余量使得早到的请求能够立即执行而不必等待
- 若请求迟到了，则消耗余量；
- 若下一个请求早到了，则增加余量
- 若余量的大于0，则表示可以sleep余量的时间
- 若余量小于0，则表示不够请求立即执行
- 为避免：余量过大，使得等待时间过长，则设置最大余量

### 最终表现为：请求均按照固定时间间隔到来