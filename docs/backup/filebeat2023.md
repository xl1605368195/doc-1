# filebeat 安装

> 适配 jrasp-agent 1.1.2 版本

支持 linux、macos、windows

## linux
下载
``` shell
wget https://www.download.jrasp.com/v1.1.2-20230709/filebeat-1.1.2-linux-x86_64.tar.gz
```
解压
```shell
tar -zxvf filebeat-1.1.2-linux-x86_64.tar.gz
```
进入安装目录
```shell
cd filebeat
```
启动
```shell
./startup.sh
```
或者：linux可以使用systemctl设置自启动
修改下面的fileBeatHome的值为实际安装目录
```shell
fileBeatHome="/usr/local/filebeat"
## systemctl
cat << EOF > /usr/lib/systemd/system/filebeat.service
[Unit]
Description=filebeat
Wants=network-online.target
After=network-online.target
[Service]
User=root
ExecStart=${fileBeatHome}/filebeat -c ${fileBeatHome}/filebeat.yml
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl enable filebeat.service;
systemctl stop filebeat.service && systemctl start filebeat.service;
systemctl status filebeat.service;
```


## macos
下载
```shell
wget https://www.download.jrasp.com/v1.1.2-20230709/filebeat-1.1.2-darwin-x86_64.tar.gz
```
解压
```shell
tar -zxvf filebeat-1.1.2-darwin-x86_64.tar.gz
```
进入安装目录
```shell
cd filebeat
```
启动
```shell
./startup.sh
```

## windows

（安装资源准备中）

## 备注 

+ filebeat.yml

```yaml
jrasp.home: /usr/local/jrasp
        
filebeat.inputs:
- type: log 
  fields:
      kafka_topic: "jrasp-daemon"
  paths:
    - ${jrasp.home}/logs/jrasp-daemon.log
```
jrasp.home是jrasp的安装目录，如果jrasp安装目录不是默认目录，请修改为实际安装目录，然后重启。

上面的安装命令在linux上已经执行过上千次了，基本无问题。如果安装失败，请仔细检查filebeat日志监控路径是否为jrasp安装目录。