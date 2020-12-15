# GRPC

MaxSendMsgSize : 服务端允许发送的最大字节数,默认4m
MaxRecvMsgSize : 最大允许接受的字节数

保活策略:
type EnforcementPolicy struct {
	// MinTime is the minimum amount of time a client should wait before sending a keepalive ping.
	MinTime time.Duration // The current default value is 5 minutes.
	// If true, server expects keepalive pings even when there are no active streams(RPCs).
	PermitWithoutStream bool // false by default.
}


## http(s)反向代理

https://github.com/grpc-ecosystem/grpc-gateway
