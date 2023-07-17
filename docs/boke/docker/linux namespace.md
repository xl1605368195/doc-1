# Linux Namespace


## UTS Namespace

UTS Namespace 主要用来隔离 nodename、domainname 2 个系统标识。在 UTS Namespace 里面，每个 Namespace 允许有自己的 hostname。

Namespace 系统调用用 c 语言实现是最好的。 这里仅用 golang 实现。下面的代码在 linux 系统上运行
```go
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func main() {
 
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		// Cloneflags Flags for clone calls (Linux only)
		Cloneflags: syscall.CLONE_NEWUTS,
	}

	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}

```

`exec.Command("sh")` 用来指定被 fork 出来的新进程内的初始命令，默认使用 sh 来执行。


```shell
[root@tx-jrasp-admin-01 docker]# ./main
sh-4.2# pwd
/root/study/docker
```

### 父子进程的 UTS Namespace 可以不同
```
─sshd(1145)─┬─sshd(11242)───bash(11267)───main(17471)─┬─sh(17474)───pstree(17517)
```
当前 shell 进程的 pid=17474，main 进程（shell 进程的父进程）的 pid=17471
```
sh-4.2# readlink /proc/17474/ns/uts
uts:[4026532152]
sh-4.2# readlink /proc/17471/ns/uts
uts:[4026531838]
```

### 修改子进程的hostname名称

默认情况下父子进程的 hostname 是相同的，子进程可以改hostname，父进程的 hostname 不改变
```shell
sh-4.2# hostname -b myshell
sh-4.2# hostname
myshell
```




### 获取子进程的hostname 
