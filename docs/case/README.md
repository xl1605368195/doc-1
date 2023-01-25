# 漏洞检测

目前JRASP支持的类型漏洞：`远程代码/命令执行漏洞(RCE)`、`反序列化漏洞`、`任意文件读取/删除/写入漏洞`、`SQL注入漏洞`、 `服务器端请求伪造漏洞(SSRF)`、 `外部实体注入(XXE)` 、 `SPEL/OGNL表达式注入`


## 2022

|CVE编号|漏洞名称|漏洞类型|
|:----:|:----:|:----:|
|[Apache commons text](CVE-2022-42889.md)|Apache commons text |任意代码执行|
|[Apache Spark shell](CVE-2022-33891.md)|Apache Spark shell |任意命令执行|
|[Spring4Shell](CVE-2022-22965.md)|Spring rce |远程命令执行|
|[Spring-Cloud-Function-SPEL](Spring-Cloud-Function-spel.md)|Spring Cloud Function 表达式注入|远程代码执行|
|[CVE-2022-22947](CVE-2022-22947.md)|Spring Cloud Gateway 代码注入|远程代码执行|




## 2021
|CVE编号|漏洞名称|漏洞类型|
|:----:|:----:|:----:|
|[CVE-2021-35464](CVE-2021-35464.md)|ForgeRock AM远程代码执行漏洞|反序列化远程代码执行|
|[CVE-2021-34429](CVE-2021-34429.md)|Jetty WEB-INF 文件读取漏洞|敏感文件访问|
|[CVE-2021-29505](CVE-2021-29505.md)|XStream远程代码执行漏洞|反序列化远程代码执行|
|[CVE-2021-28164](CVE-2021-28164.md)|Jetty WEB-INF 文件读取漏洞|敏感文件访问|
|[CVE-2021-26295](CVE-2021-26295.md)|Apache OfBiz反序列化命令执行漏洞|反序列化远程命令执行|
|[CVE-2021-25646](CVE-2021-25646.md)|Apache Druid远程代码执行漏洞|远程代码执行|
|[CVE-2021-21234](CVE-2021-21234.md)|SpringBoot Actuator Logview 任意文件读取|任意文件访问|
|[Apache Solr](ApacheSolr.md)|Apache Solr 任意文件删除|任意文件删除|


## 2020
[Jackson-databind反序列化漏洞(CVE-2020-36188)](CVE-2020-36188.md)、[Jackson-databind反序列化漏洞(CVE-2020-35728)](CVE-2020-35728.md)、

[XStream任意文件删除漏洞(CVE-2020-26259)](CVE-2020-26259.md)、[Struts 远程代码执行漏洞(CVE-2020-17530)](CVE-2020-17530.md)、

[Apache Flink 任意文件访问(CVE-2020-17519)](CVE-2020-17519.md)、[Apache Unomi远程代码代码(CVE-2020-13942)](CVE-2020-13942.md)、[WebLogic远程命令执行漏洞(CVE-2020-14882)](CVE-2020-14882.md)、

[Tomcat Session 反序列漏洞(CVE-2020-9484)](CVE-2020-9484.md)、[Spring Cloud Config目录穿越漏洞(CVE-2020-5410)](CVE-2020-5410.md)

[Apache Tomcat文件包含漏洞(CVE-2020-1938)](CVE-2020-1938.md)、[Spring Boot Actuator H2 命令执行](SpringBootActuatorH2.md)


## 2019之前

[Apache Log4j反序列化漏洞(CVE-2019-17571)](CVE-2019-17571.md)、[Apache Solr Velocity模板语言执行任意命令(CVE-2019-17558)](CVE-2019-17558.md)、

[Jackson 反序列化命令执行(CVE-2019-12384)](CVE-2019-12384.md)、[Apache Solr 命令执行(CVE-2019-0193)](CVE-2019-0193.md)、

[JavaMelody XXE漏洞(CVE-2018-15531)](CVE-2018-15531.md)、[Spring Data Commons 命令执行(CVE-2018-1273)](CVE-2018-1273.md)、

[Spring-messagingSpEL表达式注入(CVE-2018-1270)](CVE-2018-1270.md)、[Spring Data Commons XXE漏洞(CVE-2018-1259)](CVE-2018-1259.md)、

[Spring-Data-REST RCE漏洞(CVE-2017-8046)](CVE-2017-8046.md)、[Apache Solr命令执行(CVE-2017-12629) ](CVE-2017-12629.md)、

[Apache Shiro反序列化漏洞(CVE-2016-4437)](CVE-2016-4437.md)、[ActiveMQ 命令执行(CVE-2016-3088)](CVE-2016-3088.md)

