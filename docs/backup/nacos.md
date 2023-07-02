# nacos安装

+ nacos 用作 jrasp-daemon 进程的服务发现与配置下发，具有重要作用，推荐使用集群部署方式；

+ 最低配置：4核8GB内存50GB磁盘

+ 部署节点：3个

### 集群部署（线上环境推荐）

nacos官方的安装包下载较慢，这里提供一个快捷下载链接

安装目录 `/opt/nacos`
```json
wget https://jrasp-daemon-1254321150.cos.ap-shanghai.myqcloud.com/nacos-server-2.0.3-tools.tar.gz;
tar -zxvf nacos-server-2.0.3.tar.gz -C /opt/;
cat << EOF > /opt/nacos/conf/cluster.conf
172.20.52.173:8848
172.20.52.174:8848
172.20.52.175:8848
EOF
cd /opt/nacos/bin;
sh startup.sh -p embedded
```
注意：
+ 这里请配置内网 ip
+ 在其他2台机器上执行上面的命令即可
+ 至少配置3个节点

nacos 集群成功安装完成

![img_6.png](docs/.vuepress/public/images/guide/install/nacos_install_success.png)


### 单机部署 （测试环境推荐）

解压之后，在bin目录下执行
```json
sh startup.sh -m standalone
```
![img_6.png](docs/.vuepress/public/images/guide/install/nacos_success_install_2.png)

