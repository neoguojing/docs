# 竞争检测
- race.g
- race0.g

## 使用
- -race：启动竞争检测，对应内部配置BuildRace
- // +build !race.： 禁止file的检测
- 使用race将拖慢系统5-15倍
- 内部使用：ThreadSanitizer工具
## 函数
- const raceenabled = true
- func raceread(uintptr)
- func racewrite(uintptr)



## 引用
- https://www.infoq.com/presentations/go-race-detector/
