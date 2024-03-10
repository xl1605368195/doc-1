# RCE模块

## HOOK类
> HOOK最底层的native方法，不可能绕过，
业内第一款能彻底防止绕过的RASP

+ java.lang.UNIXProcess#forkAndExec(I[B[B[BI[BI[B[IZ)I
+ java.lang.ProcessImpl#forkAndExec(I[B[B[BI[BI[B[IZ)I
+ java.lang.ProcessImpl#create(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;[JZ)J

## 检测算法
+ 检测算法1：命令执行白名单
+ 检测算法2：栈白名单
+ 检测算法3：jsp命令执行
+ 检测算法4：检测WebShell管理工具命令执行
+ 检测算法5：用户输入后门
+ 检测算法6：包含敏感字符
+ 检测算法7：栈特征检测
+ 检测算法8：命令执行监控
