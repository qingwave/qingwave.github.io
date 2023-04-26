# k8s中shell脚本启动如何传递信号


## 背景
在k8s或docker中，有时候我们需要通过shell来启动程序，但是默认shell不会传递信号（sigterm）给子进程，当在pod终止时应用无法优雅退出，直到最大时间时间后强制退出（`kill -9`）。
<!--more-->
## 分析
普通情况下，大多业务的启动命令如下
```bash
command: ["binary", "-flags", ...]
```
主进程做为1号进程会收到`sigterm`信号，优雅退出(需要程序捕获信号); 而通过脚本启动时，`shell`作为1号进程，不会显示传递信号给子进程，造成子进程无法优雅退出，直到最大退出时间后强制终止。

## 解决方案
### exec
如何只需一个进程收到信号，可通过`exec`，`exec`会替换当前shell进程，即`pid`不变
```bash
#! /bin/bash
# do something
exec binay -flags ...
```

正常情况测试命令如下，使用sleep来模拟应用`sh -c 'echo "start"; sleep 100'`：
`pstree`展示如下，`sleep`进程会生成一个子进程
```
bash(28701)───sh(24588)───sleep(24589)
```

通过`exec`运行后，命令`sh -c 'echo "start"; exec sleep 100'`
```
bash(28701)───sleep(24664)
```
加入`exec`后，`sleep`进程替换了shell进程，没有生成子进程

此种方式可以收到信号，但只适用于一个子进程的情况

### trap
在shell中可以显示通过`trap`捕捉信号传递给子进程
```bash
#!/bin/bash
echo "start"
binary -flags... &
pid="$!"

_kill() {
  echo "receive sigterm"
  kill $pid #传递给子进程
  wait $pid
  exit 0
}

trap _kill SIGTERM #捕获信号
wait #等待子进程退出
```

此种方式需要改动启动脚本，显示传递信号给子进程

## docker-init
[docker-init](https://docs.docker.com/engine/reference/run/#specify-an-init-process)即在docker启动时加入`--init`参数，docker-int会作为一号进程，会向子进程传递信号并且会回收僵尸进程。

遗憾的是k8s并不支持`--init`参数，用户可在镜像中声明init进程，更多可参考[container-init](./container-init.md)
```Dockerfile
RUN wget -O /usr/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.2/dumb-init_1.2.2_amd64
RUN chmod +x /usr/bin/dumb-init
ENTRYPOINT ["/usr/bin/dumb-init", "-v", "--"]

CMD ["nginx", "-g", "daemon off;"]
```

