# 单机版本(无管理端)

> 适用内网环境

> 仅安装jrasp-agent

> 配置、日志不传云端


## 安装

安装步骤同：

[快速安装(使用管理端)-## 第一步 安装 agent](./saas.md)

## 说明

jrasp 提供不依赖于jdk的注入工具 jrasp-attach （Golang语言），用户对JVM进程注入 jrasp agent，主要功能有：
+ 注入 jrasp agent；
+ 查看哪些类被hook；
+ 更新模块参数；
+ 卸载 jrasp agent；

进入到 jrasp 安装目录

```shell
cd /usr/local/jrasp/bin
```

## 帮助教程
```shell
[root@VM-0-17-centos bin]# ./attach -h
Usage of ./attach:
  -d string
        usage for update module data. example: ./attach -p <pid> -d rce-hook:k1=v1;k2=v2;k3=v31,v32,v33
  -l    usage for list transform class. example: ./attach -p <pid> -l
  -p int
        usage for attach java pid. example: ./attach -p <pid> (default -1)
  -s    usage for stop agent. example: ./attach -p <pid> -s
  -v    usage for inject version. example: ./attach -v
```

## 加载 jrasp agent

```shell
[root@VM-0-17-centos bin]# ./attach -p 6841
2022/12/10 14:32:34 attach java process,pid: 6841
2022/12/10 14:32:38 jvm create uds socket file success
2022/12/10 14:32:38 command socket init success: [0.0.0.0:50523]
2022/12/10 14:32:38 attach jvm success
```

## 更新模块参数
这里以rce-hook模块为例子，更新模块的`disable`参数值为true
```java
$ ./attach -p 6841 -d rce-hook:disable=true
2022/12/10 14:42:37 attach java process,pid: 6841
2022/12/10 14:42:37 jvm create uds socket file success
2022/12/10 14:42:37 command socket init success: [0.0.0.0:50905]
2022/12/10 14:42:37 update module data,rce-hook:disable=true
2022/12/10 14:42:37 update parameters result:true
```

## 查看 HOOK 类
```shell
[root@VM-0-17-centos bin]# ./attach -p 6841 -l
2022/12/10 14:35:02 attach java process,pid: 6841
2022/12/10 14:35:02 jvm create uds socket file success
2022/12/10 14:35:02 command socket init success: [0.0.0.0:50523]
2022/12/10 14:35:02 list transform class:
java/lang/UNIXProcess#forkAndExec(I[B[B[BI[BI[B[IZ)I,true
```
`true`：表示该方法字节码修改成功

`false`：表示已经该类不存在或者未被加载（java类的懒加载）

## 卸载  jrasp agent
```shell
[root@VM-0-17-centos bin]# ./attach -p 6841 -s
2022/12/10 14:36:48 attach java process,pid: 6841
2022/12/10 14:36:48 jvm create uds socket file success
2022/12/10 14:36:48 command socket init success: [0.0.0.0:50523]
2022/12/10 14:36:48 stop agent
2022/12/10 14:36:53 result:success
```
