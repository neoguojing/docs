# trace

## 总结
- trace启动：stw，记录stack trace，遍历g获取所有g的状态，start world
- 每个p有trace buffer，在事件触发时由g将信息写入buffer；buffer满是刷到全局buff list；
- 有专门的g负责dump trace到输出
- go tool trace my-output.out 
- trace信息：
- > proc start :新线程启动或者从系统调用恢复
- > proc stop: m和p解绑或者-> 系统调用或者退出
- > unblock : g从系统调用解绑
- > 
