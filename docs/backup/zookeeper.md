# zookeeper 集群安装

## 集群节点

准备3台机器, 节点配置4核8GB内存50GB磁盘

节点1：172.20.52.173 （内网）

节点2：172.20.52.174 （内网）

节点3：172.20.52.175 （内网）

如果已经有zookeeper集群，可以跳过此安装步骤。

## 安装命令
### 将下面的命令复制到终端执行
```json
mkdir -p /opt/zookeeper;
mkdir -p /tmp/zookeeper;
wget https://repo.huaweicloud.com/apache/zookeeper/zookeeper-3-tools.7.0/apache-zookeeper-3-tools.7.0-bin.tar.gz;
tar -zxvf apache-zookeeper-3.7.0-bin.tar.gz -C /opt/zookeeper;       
mkdir -p /opt/zookeeper;
mkdir -p /tmp/zookeeper;        
## 配置zoo.cfg
cat << EOF > /opt/zookeeper/apache-zookeeper-3.7.0-bin/conf/zoo.cfg;
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
server.1=172.20.52.173:2888:3888
server.2=172.20.52.174:2888:3888
server.3=172.20.52.175:2888:3888
EOF
## myid
cat << EOF > /tmp/zookeeper/myid;
3
EOF
##  自动拉起与开启启动       
cat << EOF > /usr/lib/systemd/system/zookeeper.service;
[Unit]
Description=Zookeeper server manager
        
[Service]
Type=forking
Environment=JAVA_HOME=/usr/local/java/jdk1.8.0_181
ExecStart=/opt/zookeeper/apache-zookeeper-3.7.0-bin/bin/zkServer.sh start
ExecStop=/opt/zookeeper/apache-zookeeper-3.7.0-bin/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/apache-zookeeper-3.7.0-bin/bin/zkServer.sh restart
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable zookeeper
systemctl stop zookeeper
systemctl start zookeeper
systemctl status zookeeper
```
需要注意的是：/tmp/zookeeper/myid 文件的节点编号每个节点不一样，依次为 1、2、3 


### 安装结果验证
+ 上面的脚本执行成功，如下图所示

![img.png](docs/.vuepress/public/images/guide/install/zk-install.png)

+ 全部节点安装完成之后，查看 /opt/zookeeper/apache-zookeeper-3.7.0-bin/logs/zookeeper--server-{机器名称}.log 是否有错误日志，没有就是安装成功

![img.png](docs/.vuepress/public/images/guide/install/zk-install-success.png)


### 常见故障

文件丢失 /tmp/zookeeper/myid
