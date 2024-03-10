# JRASP反射加固实践

JRASP十分重视自身安全性的建设，采用了多种方式提高RASP自身的安全防护能力，包括：
+ 策略配置加密；
+ RASP自身代码与业务隔离；
+ 安全策略模块磁盘加密、运行时解密；
+ Agent与Daemon的socket通讯加密；

而在RASP绕过场景中，有一个重要的方式是调用RASP的关闭或者卸载开关。具体可以参考下面的文章：
> RASP的安全攻防研究实践 https://www.cnblogs.com/wh4am1/p/16780056.html

本文介绍下JRASP如何防止关键方法被反射调用。

## JDK代码中的反射与限制

一般的我们可以通过调用class对象的`getDeclaredMethods`来获取该类的所有的方法（包含私有方法），如：
```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) {
        Method[] declaredMethods = Thread.class.getDeclaredMethods();
        for (Method m : declaredMethods) {
            System.out.println(m.getName());
        }
    }
}
```
而对于一些执行权限较高的类如`sun.misc.Unsafe`，其中的`getUnsafe`方法却无法通过反射获取。
原因是`sun.misc.Unsafe`的静态块中有这么一行代码:
```java
static {
    Reflection.registerMethodsToFilter(Unsafe.class, new String[]{"getUnsafe"});
}
```
这个方法的作用通过方法名就可以看出来了: `注册方法到过滤器`。这个方法需要传两个参数,第一指明要过滤的Class对象,
第二个是个String数组,里面放要过滤掉的方法名称。

Reflection在sun.reflect包下,该类维护了两个Map:
```java
private static volatile Map<Class<?>, String[]> fieldFilterMap;
private static volatile Map<Class<?>, String[]> methodFilterMap;
```

以Class对象作为key值,一个保存需要过滤的属性名数组,一个保存需要过滤的方法名数组。
在我们通过反射去获取一个类的方法或属性到时候,Class对象会调用Reflection.filterMethods方法进行过滤。
下面是Class获取类全部方法的源码:


尽管`registerFieldsToFilter`和`registerFieldsToFilter`能够保护我们的代码避免被反射，但是存在一个限制条件：
只能是BootClassloader加载的类，其他类加载器的加载的类不会

## 方法调用者鉴权