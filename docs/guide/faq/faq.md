# 常见问题

## 1.支持容器环境的动态注入吗？

开源版（1.1.x版本）需要在容器内安装，商业版（1.2.x版本）在宿主机上运行支持容器内进程注入；

## 2.支持对同一节点上的多个进程注入吗？

支持，注入进程数量无限制。

## 3.异步线程调用问题如何解决？

目前【暂不解决】，强制关联线程有性能损耗。异步线程会丢失http/rpc报文消息，对检测算法的准确性、攻击溯源【有一定】的影响。

+ 对于**线程注入**场景已经解决。详情：[CHANGELOG v1.1.0](../technology/changelog.md)

## 4. attach之后对原进程性能有多大的影响？

jrasp在执行插桩时，本质上是在method的前后插入代码，会导致原来JVM里JIT编译生成的代码无效，需要退回原来的代码，再重新JIT编译 (JIT Depotimization)。
所以对于并发高的函数调用会调起抖动。 因此要避免一次性修改大量的类，对于已经修改的类不要做字节码回滚的操作，并且字节码的变更尽量在服务的低峰期进行； 建议在生产环境部署时，在服务的低峰期进行，有条件的可以切除流量。

值得注意的是：jrasp很早就观察到这种现象，并且在这方面做了一定的优化。

一方面：将检测模块拆分为hook模块+检测模块，hook模块只负责插桩并调用算法模块，这部分不需要反复变更（意味着不需要进行字节码修改），仅需要更改算法模块（无字节码修改，无性能损耗）；
另一方面：hook模块按照功能划分，每个模块实现单独的加载和卸载，实现了增量式变更，将字节码修改的影响降到最低； 


参考 [VM_RedefineClasses::flush_dependent_code](https://github.com/AdoptOpenJDK/openjdk-jdk11u/blob/jdk-11.0.10+7/src/hotspot/share/prims/jvmtiRedefineClasses.cpp#L3864)

```java
// Deoptimize all compiled code that depends on this class.
//
// If the can_redefine_classes capability is obtained in the onload
// phase then the compiler has recorded all dependencies from startup.
// In that case we need only deoptimize and throw away all compiled code
// that depends on the class.
//
// If can_redefine_classes is obtained sometime after the onload
// phase then the dependency information may be incomplete. In that case
// the first call to RedefineClasses causes all compiled code to be
// thrown away. As can_redefine_classes has been obtained then
// all future compilations will record dependencies so second and
// subsequent calls to RedefineClasses need only throw away code
// that depends on the class.
```
关于JIT的参考：

[hotspot-jit](http://ifeve.com/hotspot-jit/)

[OpenJDK-HotSpot-What-the-JIT](https://www.infoq.cn/article/OpenJDK-HotSpot-What-the-JIT)

## 5. jrasp目前有多少使用量？

截止到2024年3月，已经接入的主机数量超过5w+；多家公司接入数量达到万级别；


## 6.rasp如何与其他agent兼容

一般公司内部会有各种java agent，如何保证jrasp与其他agent的兼容性，主要从这3个方面：

1.代码隔离： jrasp采用自定义类加载器，与业务的隔离级别是类加载器级别的隔离，用户、其他agent无法访问rasp类，rasp无法访问用户类（仅在进入业务方法时可以访问）；

而常见的rasp为了能够实现修改jdk类、业务类，将rasp代码使用 bootclassloader 加载，全部可见。

2.字节码层面的兼容；基于方法的进入、返回、抛出异常修改方法体，不增加字段和方法，如果同一个方法被其他agent修改后，rasp在修改后的方法基础上更改；

3.打包方式：依赖的三方包，使用shade模式更改包名称，如agent都依赖的asm工具包等，解决依赖冲突问题

4. 其他agent应该排除观测rasp的类，即不转换rasp类
## 7.支持的web/jdk

+ 支持jdk6～17
+ web容器：tomcat6～11、jetty8～10、springboot1.x~3.x、undertow、weblogic等


## 8.安全模块

截止到2024-03，已经具备十多种安全模块，覆盖常见漏洞。

## 9.接入方式
接入方式有2种：
+ `静态接入`（修改jvm启动参数）
+ `动态接入`（用户无需修改任何代码/参数）

## 10.多语言支持

目前仅支持`Java` (JRASP), 未来考虑支持Golang （GoRASP）其他语言暂不支持
