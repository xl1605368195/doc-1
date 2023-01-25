# 安装组件

![img.png](docs/.vuepress/public/images/guide/install/jrasp.png)

主要分为下面的几个部分：
+ jrasp-admin/ui  管理端，消费kafka日志消息，下发配置
+ nacos           配置中心
+ jrasp-daemon    jrasp-agent 守护进程，注入、插件更新
+ jrasp-agent     注入到JVM中，安全防护
+ filebeat        日志收集
+ kafka/zookeeper 日志投递

## 安装文档

1.[zookeeper集群安装](docs/backup/zookeeper.mdzookeeper.md)

2.[kafka集群安装](docs/backup/kafka.mdkup/kafka.md)

3.[filebeat安装](docs/backup/filebeat.md/filebeat.md)

4.[nacos安装](docs/backup/nacos.mdkup/nacos.md)
