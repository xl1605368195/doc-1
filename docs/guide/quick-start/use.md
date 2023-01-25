# 快速使用

## 下载安装

最新版本`v1.1.0` 2022-10-03日更新

+ 下载/解压安装包 
 
```shell
// 即将发布
tar -xvf jrasp-1.1.0.tar.gz -C /usr/local/
```

## 查看 Java 进程 pid

```shell
jps -l
11844 hello.Application
```

## 加载RASP
选择 pid=18477 进程注入
```shell
cd /usr/local/jrasp/bin
./attach -p 18477
```

日志：
```shell
2022/10/03 18:06:57 attach java process,pid: 11844
2022/10/03 18:06:58 jvm create uds socket file success
2022/10/03 18:06:58 command socket init success: [0.0.0.0:65325]
2022/10/03 18:06:58 attach jvm success
```

## 查看日志

+ `jrasp/logs/jrasp-daemon.log` 是守护进程日志

+ `jrasp/logs/jrasp-agent-0.log`  是agent日志

+ `jrasp/logs/jrasp-module-0.log` 是模块日志 (模块自身运行日志，不包含攻击日志)

+ `jrasp/logs/jrasp-attack-0.log` 是攻击日志

（java agent 输出日志每个进程一个文件）

## 卸载RASP
```shell
./attach -p 18477 -s
2022/10/03 18:27:07 attach java process,pid: 11844
2022/10/03 18:27:07 jvm create uds socket file success
2022/10/03 18:27:07 command socket init success: [0.0.0.0:65325]
2022/10/03 18:27:07 stop agent
2022/10/03 18:27:08 result:success
```

