# jrasp-agent  安装

> 适配 jrasp-agent 1.1.2 版本

支持 linux、macos、windows

## linux
下载
```shell
wget https://www.download.jrasp.com/v1.1.2-20230709/jrasp-1.1.2-linux-amd64-bin.tar.gz
```

解压
```shell
tar -zxvf jrasp-1.1.2-linux-amd64-bin.tar.gz -C /usr/local/
```

进入安装目录
```shell
cd /usr/local/jrasp/bin/
```

启动
```shell
nohup ./service.sh >/dev/null 2>&1 &
```
或者 systemctl 自启动
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

## macos
下载
```shell
wget https://www.download.jrasp.com/v1.1.2-20230709/jrasp-1.1.2-bin-darwin.tar.gz
```

解压
```shell
tar -zxvf jrasp-1.1.2-bin-darwin.tar.gz
```

进入安装目录
```shell
cd jrasp/bin
```

启动
```shell
nohup ./service.sh >/dev/null 2>&1 &
```

## windows

（安装资源准备中）

----------------
下一步安装 [filebeat](filebeat.md)