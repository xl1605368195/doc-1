# 架构设计

先来说下JRASP架构：

![img.png](../../.vuepress/public/images/guide/technology/jrasp.png)

+ 与其他rasp不同的是，jrasp除了有agent有单独的守护进程。

+ 安全插件模块化。每种模块解决一类安全漏洞，并且模块可以做到热升级；

（上图仅包含部署在服务器上的agent部分，不包括管理端等）




