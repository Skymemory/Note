#### 三次握手过程

![](./three-hand.jpg)

#### 状态图

![](./tcp_status.jpeg)

#### 问题排查工具

netstat -s:查看网卡统计信息

ss	[ OPTIONS ]	[ STATE-FILTER ] [ ADDRESS-FILTER ]

```shell
ADDRESS-FILTER:

dst 10.0.0.0/24:22

dport gt :443

比较符号支持: lt、gt、eq等
过滤器间可以通过or、and逻辑连接符，not取反

```

ss命令详细文档说明: [SS Utility: Quick Intro](https://www.cyberciti.biz/files/ss.html)

#### Unix Domain Socket与TCP/IP Socket的区别：

IPC通信有两种类型：同一主机IPC、跨主机IPC，针对同一主机IPC通信，Unix Domain Socket效率更高，主要原因在于不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。

#### shutdown

- SHUT_RD：关闭读端
- SHUT_WR：关闭写端
- SHUT_RDWR：关闭读写端