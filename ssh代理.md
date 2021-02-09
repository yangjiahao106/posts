---
title: ssh代理
date: 2019-9-22 23:24:48
tags:
---

# SSH代理的用法

## 正向代理 

例1：本地portA 转发到远程机HostB的portB端口

``` bash
# ssh -N -L 0.0.0.0:PortA:HostB:PortB user@HostB
```

例2：HostA 上启动一个 PortA 端口，通过 HostB 转发到 HostC:PortC上
```
HostA$ ssh -L 0.0.0.0:PortA:HostC:PortC  user@HostB
```

## 反向代理 

例1：HostB主机的PostB转发到本地PortA
``` base
# ssh -N -R HostB:PortB:0.0.0.0:PortA user@HostB
```       

例2：HostA 将自己可以访问的 HostB:PortB 暴露给外网服务器 HostC:PortC，在 HostA 上运行：
```
HostA$ ssh -R HostC:PortC:HostB:PortB  user@HostC
```

## sock 代理

在 HostA 的本地 1080 端口启动一个 socks5 服务，通过本地 socks5 代理的数据会通过 ssh 链接先发送给 HostB，再从 HostB 转发送给远程主机：
```bash
ssh -N -D 0.0.0.0:1080 user@HostB
```

## 其他参数
```
-C 为压缩数据
-q 安静模式
-T 禁止远程分配终端-n 关闭标准输入
-N 不执行远程命令
-f 参数，把 ssh 放到后台运行。
```