# 安装 jrasp

这里提供一个编译好的安装包 v1.0.5 (更新时间2022-06-06)
```json
## 安装包下载
wget  https://jrasp-daemon-1254321150.cos.ap-shanghai.myqcloud.com/2022-06-06/v1.0.5/jrasp-1.0.5.tar.gz
tar -xvf jrasp-1.0.5.tar.gz -C /usr/local/
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

安装成功如下所示：

![img.png](docs/.vuepress/public/images/guide/install/jrasp-install.png)
