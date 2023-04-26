# k8s中fd与thread限制(一)

## 背景
linux中为了防止进程恶意使用资源，系统使用ulimit来限制进程的资源使用情况（包括文件描述符，线程数，内存大小等）。同样地在容器化场景中，需要限制其系统资源的使用量。
<!--more-->
## 限制方法
- **ulimit**: docker 默认支持ulimit设置，可以在dockerd中配置 default-ulimits 可为宿主机所有容器配置默认的ulimit，docker启动时可添加 --ulimit 为每个容器配置ulimit会覆盖默认的设置；目前k8s暂不支持ulimit
- **cgroup**: docker 默认支持cgroup中内存、cpu、pid等的限制，对于线程限制可通过 --pids-limit 可限制每个容器的pid总数，dockerd暂无默认的pid limit设置；k8s 限制线程数，可通过在kubelet中开启SupportPodPidsLimit特性，设置pod级别pid limit
- **/etc/securiy/limits.conf,systcl.confg**: 通过ulimit命令设置只对当前登录用户有效，永久设置可通过limits.conf配置文件实现，以及系统级别限制可通过systcl.confg配置文件

## 实验对比
### 环境
**本地环境**：
os: Ubuntu 16.04.6 LTS 4.4.0-154-generic
docker: 18.09.7
base-image: alpine:v3.9

**k8s环境**：
kubelet: v1.10.11.1
docker: 18.09.6

### ulimit
用户级别资源限制，分为soft限制与hard限制
- soft ： 用户可修改，但不能超过硬限制
- hard：只有root用户可修改

**修改方式**： ulimit命令，临时修改；/etc/security/limits.conf，永久修改

**工作原理**： 根据 PAM （ Pluggable Authentication Modules 简称 PAM）机制，应用程序启动时，按 /etc/pam.d 配置加载 pam_xxxx.so 模块。 /etc/pam.d 下包含了 login 、sshd 、su 、sudo 等程序的 PAM 配置文件， 因此用户重新登录时，将调用 pam_limits.so 加载 limits.conf 配置文件

#### 文件描述符限制

```
RLIMIT_NOFILE
              This specifies a value one greater than the maximum file
              descriptor number that can be opened by this process.
              Attempts (open(2), pipe(2), dup(2), etc.)  to exceed this
              limit yield the error EMFILE.  (Historically, this limit was
              named RLIMIT_OFILE on BSD.)

              Since Linux 4.5, this limit also defines the maximum number of
              file descriptors that an unprivileged process (one without the
              CAP_SYS_RESOURCE capability) may have "in flight" to other
              processes, by being passed across UNIX domain sockets.  This
              limit applies to the sendmsg(2) system call.  For further
              details, see unix(7).
```

根据定义，nofile 限制进程所能最多打开的文件数量，作用范围进程。
1. 设置 ulimit nofile限制soft 100/hard 200，默认启动为root用户
```
$ docker run -d --ulimit nofile=100:200  cr.d.xiaomi.net/containercloud/alpine:webtool top
```
2. 进入容器查看， fd soft限制为100个
```
/ # ulimit -a
-f: file size (blocks)             unlimited
-t: cpu time (seconds)             unlimited
-d: data seg size (kb)             unlimited
-s: stack size (kb)                8192
-c: core file size (blocks)        unlimited
-m: resident set size (kb)         unlimited
-l: locked memory (kb)             64
-p: processes                      unlimited
-n: file descriptors               100
-v: address space (kb)             unlimited
-w: locks                          unlimited
-e: scheduling priority            0
-r: real-time priority             0
```
3. 使用ab测试，并发90个http请求，创建90个socket，正常运行
```bash
/ # ab -n 1000000 -c 90 http://61.135.169.125:80/ &
/ # lsof | wc -l 
108
/ # lsof | grep -c ab
94
```

4. 并发100个http请求，受到ulimit限制
    ```bash
    / #  ab -n 1000000 -c 100 http://61.135.169.125:80/
    This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/

    Benchmarking 61.135.169.125 (be patient)
    socket: No file descriptors available (24)
    ```

#### 线程限制
```
RLIMIT_NPROC
              This is a limit on the number of extant process (or, more pre‐
              cisely on Linux, threads) for the real user ID of the calling
              process.  So long as the current number of processes belonging
              to this process's real user ID is greater than or equal to
              this limit, fork(2) fails with the error EAGAIN.

              The RLIMIT_NPROC limit is not enforced for processes that have
              either the CAP_SYS_ADMIN or the CAP_SYS_RESOURCE capability.
```

由定义可知，nproc进程限制的范围是对于每个uid，并且对于root用户无效。

##### 容器uid

同一主机上运行的所有容器共享同一个内核(主机的内核)，docker通过namspace对pid/utc/network等进行了隔离，虽然docker中已经实现了user namespace，但由于各种原因，默认没有开启，见[docker user namespace](https://docs.docker.com/engine/security/userns-remap/)

```
$ docker run -d  cr.d.xiaomi.net/containercloud/alpine:webtool top
```

宿主机中查看top进程，显示root用户

```
$ ps -ef |grep top
root      4096  4080  0 15:01 ?        00:00:01 top
```

容器中查看id，uid为0对应宿主机的root用户,虽然同为root用户，但Linux Capabilities不同，实际权限与宿主机root要少很多

在容器中切换用户到operator(uid为11)，执行sleep命令，主机中查看对应进程用户为app，对应uid同样为11
```
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
/ # su operator
/ $ id
uid=11(operator) gid=0(root) groups=0(root)
/ $ sleep 100
$ ps -ef |grep 'sleep 100'
app      19302 19297  0 16:39 pts/0    00:00:00 sleep 100
$ cat /etc/passwd | grep app
app:x:11:0::/home/app:
```

##### 验证不同用户下ulimit的限制
设置 ulimit nproc限制soft 10/hard 20，默认启动为root用户
```
$ docker run -d --ulimit nproc=10:20  cr.d.xiaomi.net/containercloud/alpine:webtool top
```

进入容器查看， fd soft限制为100个
```
/ # ulimit -a
-f: file size (blocks)             unlimited
-t: cpu time (seconds)             unlimited
-d: data seg size (kb)             unlimited
-s: stack size (kb)                8192
-c: core file size (blocks)        unlimited
-m: resident set size (kb)         unlimited
-l: locked memory (kb)             64
-p: processes                      10
-n: file descriptors               1048576
-v: address space (kb)             unlimited
-w: locks                          unlimited
-e: scheduling priority            0
-r: real-time priority             0
```

启动30个进程
```
/ # for i in `seq 30`;do sleep 100 &; done
/ # ps | wc -l 
36
```

切换到operator用户
```
/ # su operator
# 启动多个进程，到第11个进程无法进行fork
/ $ for i in `seq 8`; do
> sleep 100 &
> done
/ $ sleep 100 &
/ $ sleep 100 &
sh: can't fork: Resource temporarily unavailable
```

root下查看
```
/ # ps -ef | grep operator
   79 operator  0:00 sh
   99 operator  0:00 sleep 100
  100 operator  0:00 sleep 100
  101 operator  0:00 sleep 100
  102 operator  0:00 sleep 100
  103 operator  0:00 sleep 100
  104 operator  0:00 sleep 100
  105 operator  0:00 sleep 100
  106 operator  0:00 sleep 100
  107 operator  0:00 sleep 100
  109 root      0:00 grep operator
/ # ps -ef | grep operator| wc -l
10
```

##### 验证ulimit在不同容器相同uid下的限制

设置 ulimit nproc限制soft 3/hard 3，默认启动为operator用户,起4个容器，第四个启动失败
```
$ docker run -d --ulimit nproc=3:3 --name nproc1 -u operator  cr.d.xiaomi.net/containercloud/alpine:webtool top
eeb1551bf757ad4f112c61cc48d7cbe959185f65109e4b44f28085f246043e65
$ docker run -d --ulimit nproc=3:3 --name nproc2 -u operator  cr.d.xiaomi.net/containercloud/alpine:webtool top
42ff29844565a9cb3af2c8dd560308b1f31306041d3dbd929011d65f1848a262
$ docker run -d --ulimit nproc=3:3 --name nproc3 -u operator  cr.d.xiaomi.net/containercloud/alpine:webtool top
b7c9b469e73f969d922841dd77265467959eda28ed06301af8bf83bcf18e8c23
$ docker run -d --ulimit nproc=3:3 --name nproc4 -u operator  cr.d.xiaomi.net/containercloud/alpine:webtool top
b49d8bb58757c88f69903059af2ee7e2a6cc2fa5774bc531941194c52edfd763
$
$ docker ps -a |grep nproc
b49d8bb58757        cr.d.xiaomi.net/containercloud/alpine:webtool      "top"                    16 seconds ago      Exited (1) 15 seconds ago                               nproc4
b7c9b469e73f        cr.d.xiaomi.net/containercloud/alpine:webtool      "top"                    23 seconds ago      Up 22 seconds                                           nproc3
42ff29844565        cr.d.xiaomi.net/containercloud/alpine:webtool      "top"                    31 seconds ago      Up 29 seconds                                           nproc2
eeb1551bf757        cr.d.xiaomi.net/containercloud/alpine:webtool      "top"                    38 seconds ago      Up 36 seconds                                           nproc1
```

##### 总结
- ulimit限制fd总数，限制级别进程，可对所有用户生效
- ulimit限制线程总数，限制级别用户（uid)，限制同一个uid下所有线程/进程数，对于root账号无效
- 对于目前线上情况，有较小的概率因ulimit限制导致fork失败，如同一个宿主机中有多个work容器且基础镜像相同（即uid相同），若一个容器线程泄露，由于ulimit限制会影响其他容器正常运行

### cgroup
cgroup中对pid进行了隔离，通过更改docker/kubelet配置，可以限制pid总数，从而达到限制线程总数的目的。线程数限制与系统中多处配置有关，取最小值，参考[stackoverflow上线程数的设置](https://stackoverflow.com/questions/34452302/how-to-increase-maximum-number-of-jvm-threads-linux-64bit)

- docker，容器启动时设置 --pids-limit 参数，限制容器级别pid总数
- kubelet，开启SupportPodPidsLimit特性，设置--pod-max-pids参数，限制node每个pod的pid总数

以kubelet为例，开启SupportPodPidsLimit，`--feature-gates=SupportPodPidsLimit=true`

1. 配置kubelet，每个pod允许最大pid数目为150
```bash
[root@node01 ~]# ps -ef |grep kubelet
root     18735     1 14 11:19 ?        00:53:28 ./kubelet --v=1 --address=0.0.0.0 --feature-gates=SupportPodPidsLimit=true --pod-max-pids=150 --allow-privileged=true --pod-infra-container-image=cr.d.xiaomi.net/kubernetes/pause-amd64:3.1 --root-dir=/home/kubelet --node-status-update-frequency=5s --kubeconfig=/home/xbox/kubelet/conf/kubelet-kubeconfig --fail-swap-on=false --max-pods=254 --runtime-cgroups=/systemd/system.slice/frigga.service --kubelet-cgroups=/systemd/system.slice/frigga.service --make-iptables-util-chains=false
```

2. 在pod中起测试线程，root下起100个线程
```
/ # for i in `seq 100`; do
> sleep 1000 &
> done
/ # ps | wc -l
106
```

3. operator 下，创建线程受到限制，系统最多只能创建150个
```
/ # su operator
/ $ 
/ $ for i in `seq 100`; do
> sleep 1000 &
> done
sh: can't fork: Resource temporarily unavailable
/ $ ps | wc -l
150
```

4. 在cgroup中查看，pids达到最大限制
```
[root@node01 ~]# cat /sys/fs/cgroup/pids/kubepods/besteffort/pod8b61d4de-a7ad-11e9-b5b9-246e96ad0900/pids.current 
150
[root@node01 ~]# cat /sys/fs/cgroup/pids/kubepods/besteffort/pod8b61d4de-a7ad-11e9-b5b9-246e96ad0900/pids.max 
150
```

5. 总结
cgroup对于pid的限制能够达到限制线程数目的，目前docker只支持对每个容器的限制，不支持全局配置；kubelet只支持对于node所有pod的全局配置，不支持具体每个pod的配置

### limits.conf/sysctl.conf

limits.conf是ulimit的具体配置，目录项/etc/security/limit.d/中的配置会覆盖limits.conf。

sysctl.conf为机器级别的资源限制，root用户可修改，目录项/etc/security/sysctl.d/中的配置会覆盖sysctl.conf，在/etc/sysctl.conf中添加对应配置（fd: fs.file-max = {}; pid: kernel.pid_max = {}）

1. 测试容器中修改sysctl.conf文件
    ```
    $ docker run -d --ulimit nofile=100:200 cr.d.xiaomi.net/containercloud/alpine:webtool top
    cb1250c8fd217258da51c6818fa2ce2e2f6e35bf1d52648f1f432e6ce579cf0d
    $ docker exec -it cb1250c sh

    / # ulimit -a
    -f: file size (blocks)             unlimited
    -t: cpu time (seconds)             unlimited
    -d: data seg size (kb)             unlimited
    -s: stack size (kb)                8192
    -c: core file size (blocks)        unlimited
    -m: resident set size (kb)         unlimited
    -l: locked memory (kb)             64
    -p: processes                      unlimited
    -n: file descriptors               100
    -v: address space (kb)             unlimited
    -w: locks                          unlimited
    -e: scheduling priority            0
    -r: real-time priority             0
    / # 
    / # echo 10 > /proc/sys/kernel/pid_max
    sh: can't create /proc/sys/kernel/pid_max: Read-only file system
    / # echo 10 > /proc/sys/kernel/pid_max
    sh: can't create /proc/sys/kernel/pid_max: Read-only file system
    / # echo "fs.file-max=5" >> /etc/sysctl.conf
    / # sysctl -p
    sysctl: error setting key 'fs.file-max': Read-only file system
    ```
2. 以priviledged模式测试，谨慎测试
    ```
    $ cat /proc/sys/kernel/pid_max
    32768
    $ docker run -d -- --ulimit nofile=100:200 cr.d.xiaomi.net/containercloud/alpine:webtool top
    $ docker exec -it pedantic_vaughan sh
    / # cat /proc/sys/kernel/pid_max
    32768
    / # echo 50000 > /proc/sys/kernel/pid_max
    / # cat /proc/sys/kernel/pid_max
    50000
    / # exit
    $ cat /proc/sys/kernel/pid_max
    50000 # 宿主机的文件也变成50000
    ```

3. 总结
由于docker隔离的不彻底，在docker中修改sysctl会覆盖主机中的配置，不能用来实现容器级别资源限制
limits.conf可以在容器中设置，效果同ulimit

## 结论
![pod-fd-limit](/img/blogImg/pod-fd-limit.png)

推荐方案如下：
- fd限制： 修改dockerd配置`default-ulimits`，限制进程级别fd
- thread限制：修改kubelet配置`--feature-gates=SupportPodPidsLimit=true --pod-max-pids={}`，cgroup级别限制pid，从而限制线程数
- 其他注意事项，调整节点pid.max参数；放开或者调大镜像中ulimit对非root账户nproc限制

## 引用
- https://docs.docker.com/engine/reference/commandline/run/#set-ulimits-in-container---ulimit
- http://man7.org/linux/man-pages/man2/getrlimit.2.html
- https://feichashao.com/ulimit_demo/
- https://medium.com/@mccode/understanding-how-uid-and-gid-work-in-docker-containers-c37a01d01cf
- https://docs.docker.com/engine/security/userns-remap/
