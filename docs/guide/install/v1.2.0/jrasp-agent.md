# jrasp 1.2.0 版本特性说明

+ 在宿主机上启动一个tomcat docker镜像

![img_1.png](./img_1.png)

+ 启动jrasp守护进程daemon，自动发现Java进程（包括容器中Java进程）

![img.png](./img.png)

+ 容器中Java进程pid为1，宿主机上进程为2523392

+ 控制台上看到进程

![img_3.png](./img_3.png)

## 特性1：注入容器中Java进程

进入到进程/proc目录，jrasp 需要的安装包已经自动安装到了容器中了（不包含 jrasp-daemon）。

![img_2.png](./img_2.png)

进程已经处于防护状态

![img_4.png](./img_4.png)


## 特性2：日志传输使用socket

1.1.x 版本日志写入磁盘，需要借助logagent完成日志收集和投递；

1.2.x 版本不再需要安装logagent；

所有日志通过socket传输到控制台，agent或者daemon断开连接后日志写入磁盘，重连后自动回传离线日志；

![img_5.png](./img_5.png)

## 特性3： 配置自动提取

编译时通过@RaspValue注解提取配置参数
![img_6.png](./img_6.png)


注：1.2.x 将是2024年主要维护版本