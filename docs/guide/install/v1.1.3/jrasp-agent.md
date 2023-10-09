# jrasp-agent  安装

> 适配 jrasp-agent 1.1.3 版本

支持 linux、macos、windows

## linux
#### jrasp-agent 安装
```shell
# 下载
wget https://www.download.jrasp.com/v1.1.3/jrasp-agent/jrasp-1.1.3-linux-amd64-bin.tar.gz
# 解压
tar -zxvf jrasp-1.1.3-linux-amd64-bin.tar.gz -C /usr/local/
# 进入安装目录
cd /usr/local/jrasp/bin/
# 启动
nohup ./service.sh >/dev/null 2>&1 &
```
或者 systemctl 自启动 （可选）
```shell
## 配置守护进程        
cat << EOF > /usr/lib/systemd/system/jrasp-daemon.service
[Unit]
Description=jrasp-daemon service

[Service]
Type=simple
WorkingDirectory=/usr/local/jrasp/bin
ExecStart=/usr/local/jrasp/bin/startup.sh
ExecStop=/usr/local/jrasp/bin/shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

## 设置开机启动与自动拉起
systemctl daemon-reload;
systemctl enable jrasp-daemon.service;
systemctl stop jrasp-daemon.service;
systemctl start jrasp-daemon.service;
systemctl status jrasp-daemon.service;
```
#### filebeat 安装
```shell
# 下载
wget https://www.download.jrasp.com/v1.1.2-20230709/filebeat-1.1.2-linux-x86_64.tar.gz
# 解压
tar -zxvf filebeat-1.1.2-linux-x86_64.tar.gz
# 进入安装目录
cd filebeat
# 启动
./startup.sh
```
使用systemctl自启动 （可选）


修改下面的fileBeatHome的值为fileBeat的解压目录
> 我这里filebeat解压到了`/usr/local/filebeat`下
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
#### jrasp-agent 安装
```shell
# 下载
wget https://www.download.jrasp.com/v1.1.3/jrasp-agent/jrasp-1.1.3-bin-darwin.tar.gz
# 解压 
tar -zxvf jrasp-1.1.3-bin-darwin.tar.gz
# 进入安装目录
cd jrasp/bin
# 启动
nohup ./service.sh >/dev/null 2>&1 &
```
#### filebeat 安装
```yaml
# 下载
wget https://www.download.jrasp.com/v1.1.2-20230709/filebeat-1.1.2-darwin-x86_64.tar.gz
# 解压
tar -zxvf filebeat-1.1.2-darwin-x86_64.tar.gz
# 进入安装目录
cd filebeat
# 启动
./startup.sh
```

## windows

请自行编译

## 管理端

https://www.server.jrasp.com:8088/rasp-admin

admin 123456

安装完成之后,主机注册成功：

![安装完成之后](./install.png)