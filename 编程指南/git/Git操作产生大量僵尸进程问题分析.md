# Git操作产生大量僵尸进程问题分析

当前项目中发现，在代码中使用Java调用Git进行克隆操作，如果遇到Git失败的某些场景，就会出现[[僵尸进程]]。

典型错误信息如下：

```
fatal: git upload-pack: aborting due to possible repository corruption on the remote side.
fatal: early EOF
fatal: index-pack failed
```

运行环境为：
- 操作系统： `alpine 3.12`
- git: `2.26.2-r0`
- ssh-client: `openssh-client-8.3-p1-r0`
- java: `openjdk11-jdk-11.0.8-p10-r0`
- ztProcess: `1.11`
- kotlin：`1.4.0`


## 尝试重现

找了一个有问题的Git仓库，在代码中单独进行克隆，看下是否能重现。

```kotlin
fun testClone() {  
    File(zombieTestPath).mkdirs()  
    File(zombieTestPath).deleteRecursively()  
      
    ProcessExecutor()  
        .command("git", "clone", "--bare", "--no-tags",  
 "ssh://git@code-cbu.huawei.com:2233/lWX456302/DMS-DMK.git", zombieTestPath)  
        .readOutput(true)  
        .exitValueNormal()  
        .execute()  
}
```

经过测试，已重现。

## 尝试替换Zt-exec库	

使用Java自带的ProcessBuilder库来启动进程，看下是否还有问题：

```kotlin
fun testClone1() {  
    File(zombieTestPath).mkdirs()  
    File(zombieTestPath).deleteRecursively()  
  
    ProcessBuilder()  
        .command("git", "clone", "--bare", "--no-tags",  
 "ssh://git@code-cbu.huawei.com:2233/lWX456302/DMS-DMK.git", zombieTestPath)  
        .start()  
        .waitFor()  
}
```

遗憾的是，仍然重现了。

## 尝试升级Git和SSH

很遗憾的是，我们使用alpine 3.12版本上，git和openssh-client版本已经是最新的了，这条路走不通。

## 外事不决问谷歌
使用关键字`git zombie ssh`进行搜索，在Github上发现了一个[类似的问题](https://github.com/breser/git2consul/issues/155)。根据说明来看，跟我们的现象基本一致。问题单里面提到了此问题的原因和[解决方案](https://github.com/krallin/tini)。

### 问题定位
问题在于我们容器启动的时候，把我们自己的Java进程设置为了初始进程（PID为1）。PID为1的进程在Linux里面是特殊的init进程，需要处理僵尸进程的场景。

```
PID    PPID     USER      STAT       COMMAND
1       0       root      S           java -XX:+UseG1GC ....
```

但是我们的Java程序没有正确处理进程发送的 `SIGCHLD` Singal信号，导致Git操作剩余的SSH进程一直存在。

网络上有一篇文章说明非常详细：

https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/

### 解决方案
使用tini来作为PID 1进程。

1. 修改基础镜像，将[tini](https://github.com/krallin/tini)作为[[ENTRY POINT]]

	```docker
	RUN apk add --no-cache tini

	ENTRYPOINT ["/sbin/tini", "--"]
	```
	
2. 业务镜像里面升级基础镜像。

	修改后的进程如下：

	```
	PID    PPID    USER    STAT       COMMAND
	8       1       root     S         java -XX:+UseG1GC ....
	1       0       root     S         /sbin/init -- java -XX:+UseG1GC ....
	```
	
这样僵尸进程🧟‍♂️就被正常处理了。