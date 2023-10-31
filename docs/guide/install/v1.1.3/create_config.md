# 创建配置

> jrasp-agent 安装包中包含了默认配置，能满足一般情况的使用，如果需要定制策略，则需要对配置进行修改

## 01.模块jar包上传

将模块批量上传到管理端。

模块可以来自于jrasp-agent安装包或者jrasp-agent源码编译输出

![img_27.png](img_27.png)


## 0.2模块配置

![img_29.png](img_29.png)



![img_30.png](img_30.png)



![img_31.png](img_31.png)

一个防护组件一般包含2个jar包（hook模块+算法模块）

先添加rce-hook模块
![img_32.png](img_32.png)

选中模块后，显示可以配置的参数 
![img_33.png](img_33.png)

确定

![img_34.png](img_34.png)


![img_35.png](img_35.png)


在此添加rce-algorithm模块

![img_36.png](img_36.png)


![img_37.png](img_37.png)



添加2个模块后，一个防护组件就完成了

![img_38.png](img_38.png)


确认模块其他信息
![img_39.png](img_39.png)



## 0.3新增配置


![img_40.png](img_40.png)

选择刚才生成的rce-test （其他防护模块生成方式一样）

![img_41.png](img_41.png)


生成的配置：

![img_42.png](img_42.png)



## 0.4配置下发


![img_43.png](img_43.png)


![img_44.png](img_44.png)


## 0.5 策略包导出


上面配置过程较为繁琐，20+jar包配置比较复杂。但是配置一次之后，就可以复用了。

![img_45.png](img_45.png)

如果有多个管理端，可以将刚才导出的压缩包倒入到新的管理端中，避免反复配置。

![img_46.png](img_46.png)