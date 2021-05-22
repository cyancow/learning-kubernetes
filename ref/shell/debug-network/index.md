---
title: 网络调试
type: book
date: "2021-05-21"
---

## 查看监听队列

```bash
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     129    128                *:80                             *:*
```

## 查看网络计数器

```bash
$ netstat -az
...
TcpExtListenOverflows           12178939              0.0
TcpExtListenDrops               12247395              0.0
...
```

```bash
netstat -s | grep -E 'drop|overflow'
```

## 查看 conntrack

```bash
$ conntrack -S
cpu=0   	found=770 invalid=3856 ignore=42570125 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=746284
cpu=1   	found=784 invalid=3647 ignore=41988392 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=718963
cpu=2   	found=25588 invalid=71264 ignore=243330690 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=2319295
cpu=3   	found=25706 invalid=70168 ignore=242616824 insert=0 insert_failed=0 drop=0 early_drop=0 error=18 search_restart=2320376
```

* 若有 insert_failed，表示存在 conntrack 插入失败，会导致丢包。


## 查看连接数

当前各种状态的 TCP 连接数:

```bash
$ netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 18
TIME_WAIT 457
```

## 测试网络连通性

不断 telnet 查看网络是否能通:

```bash
while true; do echo "" | telnet 10.0.0.3 443; sleep 0.1; done
```

* `ctrl+c` 终止测试
* 替换 `10.0.0.3` 与 `443` 为需要测试的 IP/域名 和端口

没有安装 telnet，也可以使用 nc 测试:

```bash
$ nc -vz 10.0.0.3 443
```