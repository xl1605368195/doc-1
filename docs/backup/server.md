# 管理端安装


## 依赖的服务

上面的步骤已经安装好了 mysql、kafka、nacos 

+ mysql 

111.229.199.6

root

rVdLv225cxWQb1pv


+ kafka

106.14.26.4:9092,47.101.64.183:9092,139.224.220.2:9092

+ nacos

132.232.32.246:8848


## 下载安装 server
```json
wget https://jrasp-daemon-1254321150.cos.ap-shanghai.myqcloud.com/2022-07-03/v1.0.7/rasp-admin.tar.gz
```

安装目录下**conf/application-additional.yml**

替换下面红色方框中的配置项：

![img.png](docs/.vuepress/public/images/guide/install/server-config.png)
