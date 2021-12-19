---
title: "为什么 Pod 一直 Terminating ?"
type: book
date: "2021-12-17"
---

## 概述

有时候删除 Pod 一直卡在 Terminating 状态，一直删不掉，本文给出可能原因。

## 磁盘爆满

如果容器运行时 (docker或containerd等) 的数据目录所在磁盘被写满，运行时就无法正常无法创建和销毁容器，kubelet 调用运行时去删除容器时就没有反应，看 event 类似这样：

```bash
Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod
```

解决方案：清理磁盘空间

## 存在 "i" 文件属性

如果容器的镜像本身或者容器启动后写入的文件存在 "i" 文件属性，此文件就无法被修改删除，而删除 Pod 时会清理容器目录，但里面包含有不可删除的文件，就一直删不了，Pod 状态也将一直保持 Terminating，kubelet 报错:

``` log
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.922965   14109 remote_runtime.go:250] RemoveContainer "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257" from runtime service failed: rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.923027   14109 kuberuntime_gc.go:126] Failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
```

通过 `man chattr` 查看 "i" 文件属性描述:

``` txt
       A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no
link can be created to this file and no data can be written to the file.  Only the superuser
or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```

彻底解决当然是不要在容器镜像中或启动后的容器设置 "i" 文件属性，临时恢复方法： 复制 kubelet 日志报错提示的文件路径，然后执行 `chattr -i <file>`:

``` bash
chattr -i /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash
```

执行完后等待 kubelet 自动重试，Pod 就可以被自动删除了。

## docker 17 的 bug

docker hang 住，没有任何响应，看 event:

```bash
Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

怀疑是17版本dockerd的BUG。可通过 `kubectl -n cn-staging delete pod apigateway-6dc48bf8b6-clcwk --force --grace-period=0` 强制删除pod，但 `docker ps` 仍看得到这个容器

处置建议：

* 升级到docker 18. 该版本使用了新的 containerd，针对很多bug进行了修复。
* 如果出现terminating状态的话，可以提供让容器专家进行排查，不建议直接强行删除，会可能导致一些业务上问题。

## 存在 Finalizers

k8s 资源的 metadata 里如果存在 `finalizers`，那么该资源一般是由某程序创建的，并且在其创建的资源的 metadata 里的 `finalizers` 加了一个它的标识，这意味着这个资源被删除时需要由创建资源的程序来做删除前的清理，清理完了它需要将标识从该资源的 `finalizers` 中移除，然后才会最终彻底删除资源。比如 Rancher 创建的一些资源就会写入 `finalizers` 标识。

处理建议：`kubectl edit` 手动编辑资源定义，删掉 `finalizers`，这时再看下资源，就会发现已经删掉了

## 低版本 kubelet list-watch 的 bug

之前遇到过使用 v1.8.13 版本的 k8s，kubelet 有时 list-watch 出问题，删除 pod 后 kubelet 没收到事件，导致 kubelet 一直没做删除操作，所以 pod 状态一直是 Terminating

## dockerd 与 containerd 的状态不同步

判断 dockerd 与 containerd 某个容器的状态不同步的方法：

* describe pod 拿到容器 id
* docker ps 查看的容器状态是 dockerd 中保存的状态
* 通过 docker-container-ctr 查看容器在 containerd 中的状态，比如:
  ``` bash
  $ docker-container-ctr --namespace moby --address /var/run/docker/containerd/docker-containerd.sock task ls |grep a9a1785b81343c3ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0
  a9a1785b81343c3ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0    30639    STOPPED
  ```

containerd 看容器状态是 stopped 或者已经没有记录，而 docker 看容器状态却是 runing，说明 dockerd 与 containerd 之间容器状态同步有问题，目前发现了 docker 在 aufs 存储驱动下如果磁盘爆满可能发生内核 panic :

``` txt
aufs au_opts_verify:1597:dockerd[5347]: dirperm1 breaks the protection by the permission bits on the lower branch
```

如果磁盘爆满过，dockerd 一般会有下面类似的日志:

``` log
Sep 18 10:19:49 VM-1-33-ubuntu dockerd[4822]: time="2019-09-18T10:19:49.903943652+08:00" level=error msg="Failed to log msg \"\" for logger json-file: write /opt/docker/containers/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f-json.log: no space left on device"
```

随后可能发生状态不同步，已提issue:  https://github.com/docker/for-linux/issues/779

* 临时恢复: 执行 `docker container prune` 或重启 dockerd
* 长期方案: 运行时推荐直接使用 containerd，绕过 dockerd 避免 docker 本身的各种 BUG

## Daemonset Controller 的 BUG

有个 k8s 的 bug 会导致 daemonset pod 无限 terminating，1.10 和 1.11 版本受影响，原因是 daemonset controller 复用 scheduler 的 predicates 逻辑，里面将 nodeAffinity 的 nodeSelector 数组做了排序（传的指针），spec 就会跟 apiserver 中的不一致，daemonset controller 又会为 rollingUpdate类型计算 hash (会用到spec)，用于版本控制，造成不一致从而无限启动和停止的循环。

* issue: https://github.com/kubernetes/kubernetes/issues/66298
* 修复的PR: https://github.com/kubernetes/kubernetes/pull/66480

升级集群版本可以彻底解决，临时规避可以给 rollingUpdate 类型 daemonset 不使用 nodeAffinity，改用 nodeSelector。

## mount 的目录被其它进程占用

dockerd 报错 `device or resource busy`:

``` bash
May 09 09:55:12 VM_0_21_centos dockerd[6540]: time="2020-05-09T09:55:12.774467604+08:00" level=error msg="Handler for DELETE /v1.38/containers/b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905 returned error: container b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905: driver \"overlay2\" failed to remove root filesystem: remove /data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged: device or resource busy"
```

查找还有谁在"霸占"此目录:

``` bash
$ grep 8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59 /proc/*/mountinfo
/proc/27187/mountinfo:4500 4415 0:898 / /var/lib/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work
/proc/27187/mountinfo:4688 4562 0:898 / /var/lib/docker/overlay2/81c322896bb06149c16786dc33c83108c871bb368691f741a1e3a9bfc0a56ab2/merged/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work
```

> 自行替换容器 id

找到进程号后查看此进程更多详细信息:

``` bash
ps -f 27187
```
