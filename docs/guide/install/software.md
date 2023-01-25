# 常用软件快速安装

包含 jdk11、jdk8、Golang的安装教程

##  快速安装jdk11

### linux

#### 下载 jdk
```shell
wget https://repo.huaweicloud.com/java/jdk/11+28/jdk-11_linux-x64_bin.tar.gz
```

#### 解压

```shell
mkdir -p /usr/local/java
tar -xvf jdk-11_linux-x64_bin.tar.gz -C /usr/local/java
```

#### 配置环境变量

```shell
vim /etc/profile
```

文件末尾增加下面配置项
```shell
export JAVA_HOME=/usr/local/java/jdk-11
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

#### 保存后，令配置生效

```shell
source /etc/profile
```

#### 版本检查

```shell
java -version
```

##  快速安装jdk8
### linux

#### 下载 jdk
```shell
wget https://repo.huaweicloud.com/java/jdk/8u181-b13/jdk-8u181-linux-x64.tar.gz
```

#### 解压

```shell
mkdir -p /usr/local/java
tar -xvf jdk-8u181-linux-x64.tar.gz -C /usr/local/java
```

#### 配置环境变量

```shell
vim /etc/profile
```

文件末尾增加下面配置项
```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_181
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

#### 保存后，令配置生效

```shell
source /etc/profile
```

#### 版本检查

```shell
java -version
```

## 快速安装Go

+ go版本`1.17.5`

+ 操作系统：linux

### 1.下载安装包
```shell
yum install wget -y
wget https://golang.google.cn/dl/go1.17.5.linux-amd64.tar.gz
```

### 2.解压
```shell
tar zxf go1.17.5.linux-amd64.tar.gz -C /usr/local/
```

### 3.添加环境变量
```shell
vim /etc/profile
vim /etc/bashrc
分别在最后一行添加:
export PATH=$PATH:/usr/local/go/bin
```

### 4.使环境变量生效
```shell
source /etc/profile
source /etc/bashrc
```

### 5.ok啦~
```shell
go version      
go version go1.17.5 linux/amd64
```