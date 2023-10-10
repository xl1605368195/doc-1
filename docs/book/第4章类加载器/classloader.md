# 第4章类加载器

类加载器负责把描述类的数据从class字节码文件加载到JVM，并对数据进行检验、解析和初始化，最终形成可以被JVM直接使用的Java类型。

一般在开发web项目时很少直接使用类加载器，因为web容器为我们屏蔽了类加载器的复杂性，而我们只需要实现具体的业务逻辑即可。而如果开发过Java中间件，你会发现类加载器使用非常频繁。

本章先介绍类加载器的API及使用，然后依次介绍ClassLoader的源码，JDK和web中间件的类加载器，最后介绍热加载技术的实现原理。

## 4.1 ClassLoader源码解析

ClassLoader是一个抽象类，不能直接使用，因此我们需要继承并重写其中的方法。它的主要方法有defineClass、loadClass、findClass和resolveClass等及其重载方法。主要方法的定义如下：
```java
 // 方法的输入是字节码的byte数组，输出是Class对象，它的作用是将别字节码数组解析JVM能够识别的Class对象
 protected final Class<?> defineClass(String name, byte[] b, int off, int len)
 
 // 通过类名称查找Class对象
 public Class<?> loadClass(String name)
 
 // 通过类名称加载类
 protected Class<?> findClass(String name)

 // 类加载后调用该方法完成类的链接
 protected final void resolveClass(Class<?> c)
```

先来实现一个简单的类加载器NetworkClassLoader，这个类加载器具备从网络加载类文件的能力， 实现代码如下：
```java
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URL;

public class NetworkClassLoader extends ClassLoader {
    
    // 下载地址
    private String downloadUrl;

    public NetworkClassLoader(String downloadUrl) {
        this.downloadUrl = downloadUrl;
    }

    // 实现类的查找方法
    @Override
    public Class findClass(String name) {
        byte[] b = loadClassData(name);
        return defineClass(name, b, 0, b.length);
    }
    
    // 从远程下载类文件，从而获得类的字节码数组
    private byte[] loadClassData(String name) {
        // load the class data from the connection
        InputStream input = null;
        ByteArrayOutputStream baos = null;
        String path = classNameToPath(name);
        try {
            URL url = new URL(path);
            byte[] buff = new byte[1024 * 4];
            int len = -1;
            input = url.openStream();
            baos = new ByteArrayOutputStream();
            while ((len = input.read(buff)) != -1) {
                baos.write(buff, 0, len);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (input == null) {
                try {
                    input.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return baos.toByteArray();
    }
    
    // 类名称转化为服务器下载的地址
    private String classNameToPath(String name) {
        return downloadUrl + "/" + name.replace(".", "/") + ".class";
    }

    // 测试方法
    public class Main {
        public static void main(String[] args) throws Exception {
            // 下载地址
            String baseUrl = "https://wwww.jrasp.com";
            // 初始化网络类加载器
            NetworkClassLoader loader = new NetworkClassLoader(baseUrl);
            // 加载位于 https://wwww.jrasp.com/Foo.class的类，并创建实例
            Object foo = loader.loadClass("Foo").newInstance();
        }
    }
}
```

被加载的类Foo是一个简单类，在创建实例对象时输出"create new instance"，Foo类的代码如下：
```java
public class Foo {
    public Foo() {
        System.out.println("create new instance");
    }
}
// 运行Main方法，输出结果如下：
// create new instance
```

ClassLoader主要功能是类查找、加载和链接等过程，除了加载类之外，类加载器还负责加载资源如配置文件或图片等。

## 4.2 ClassLoader源码解析

有了上面的使用基础，再来分析下类加载器及其重要实现类的源码。

### 4.2.1 loadClass

ClassLoader调用其loadClass方法来加载class的，loadClass核心代码如下：

> 代码位置：src/java.base/share/classes/java/lang/ClassLoader.java

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 首先, 检查类是否已经被加载了
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    // 当前类加载器的父加载不为空，尝试从父类加载器加载
                    c = parent.loadClass(name, false);
                } else {
                    // 父加载器为空，使用启动类加载器加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 忽略异常，继续查找
            }

            if (c == null) {
                // 父加载器加载不到，调用当前类加载器重写的findClass查找
                c = findClass(name);
            }
        }
        // 链接类
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

上面的类加载顺序可以总结为：优先从尝试父加载器去加载（如果父加载器为null，则调用系统类加载器BootstrapClassLoader去加载），父加载器都尝试失败后才会交由当前ClassLoader重写的findClass方法去加载。如下图4-1所示：

> 图4-1 类加载器的委托模型

![图4-1 类加载器的委托模型.png](images%2F%E5%9B%BE4-1%20%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E5%A7%94%E6%89%98%E6%A8%A1%E5%9E%8B.png)

### 4.2.2 findClass
在加载class的过程中，如果父加载器都没有找到，则调用子类加载器重写的findClass方法继续查找，
findClass方法如下。

> 代码位置：src/java.base/share/classes/java/lang/ClassLoader.java

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    // 调用时抛出异常  
    throw new ClassNotFoundException(name);
}
```
可以看到该方法里面抛出异常，因此不能直接调用，需要子类来实现。URLClassLoader是ClassLoader的子类并重写了findClass方法。
URLClassLoader的属性与构造器如下：
> 代码位置：src/java.base/share/classes/java/net/URLClassLoader.java

```java
// 类和资源的查找路径
private final URLClassPath ucp;

public URLClassLoader(URL[] urls, ClassLoader parent) {
    // 指定父加载器
    super(parent);
    // ... 权限检查代码省略
    this.acc = AccessController.getContext();
    // 初始化 ucp 属性
    ucp = new URLClassPath(urls, acc);
}
```

实现ClassLoader的findClass方法加载指定路径下的类。
> 代码位置：src/java.base/share/classes/java/net/URLClassLoader.java
```java
protected Class<?> findClass(final String name) throws ClassNotFoundException {
    // 1、将类的全限定名变成.class文件路径的方式
    String path = name.replace('.', '/').concat(".class");
    // 2、在URLClassPath中查找是否存在
    Resource res = ucp.getResource(path, false);
    // ... 异常处理忽略
    return defineClass(name, res);
}
```

URLClassLoader的findClass方法的执行逻辑主要分为三步：
+ 将类的全限定名变成.class文件路径的方式；
+ 在URL中查找文件是否存在；
+ 调用defineClass完成类的链接和初始化；

### 4.2.3 defineClass
defineClass与findClass一起使用，findClass负责读取自于磁盘或者网络的字节码，而defineClass将字节码解析为Class对象，在defineClass方法中使用resolveClass方法完成对Class的链接。源代码如下:

> 代码位置：src/java.base/share/classes/java/lang/ClassLoader.java

```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                     ProtectionDomain protectionDomain)
    throws ClassFormatError {
    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    // 调用native方法完成链接
    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
    postDefineClass(c, protectionDomain);
    return c;
}
```
defineClass的实现在defineClass1方法中，defineClass1是一个native方法，具体实现hotspot中，实现较为复杂，一般不需要特别关注。ClassLoader加载一个class文件到JVM时需要经过的步骤，如下图4-2所示：

> 图4-2 JVM加载类的阶段

![图4-2 JVM加载类的阶段](images/%E5%9B%BE4-2%20JVM%E5%8A%A0%E8%BD%BD%E7%B1%BB%E7%9A%84%E9%98%B6%E6%AE%B5.png)

一般我们只需要重写ClassLoader的findClass方法获取需要加载的类的字节码，然后调用defineClass方法生成Class对象。如果想要在类加载到JVM中时就被链接，可以调用resolveClass方法，也可以选择交给JVM在类初始化时链接。

## 4.3 JDK的类加载器

JDK自身的jar包如rt.jar和tools.jar等中的类也需要使用类加载器来加载，下面的代码用来获取JDK内置的类加载器。
```java
public class JdkClassloader {
    public static void main(String[] args) {
        // 获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);
        
        // 获取系统类加载器的父类加载器 --> 扩展类加载器或者平台类加载器
        ClassLoader platformClassLoader = systemClassLoader.getParent();
        System.out.println(platformClassLoader);
        
        // 获取扩展类加载器的父类加载器 --> 启动类加载器（C/C++）
        ClassLoader bootstrapClassLoader = platformClassLoader.getParent();
        System.out.println(bootstrapClassLoader);
    }
}
```

在JDK8上运行：
```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@4a574795
null
```

在JDK11上运行：
```
jdk.internal.loader.ClassLoaders$AppClassLoader@512ddf17
jdk.internal.loader.ClassLoaders$PlatformClassLoader@3cda1055
null
```
可以看到JDK8和JDK11类加载器的类名称存在差异，下面分别说明其实现。

### 4.3.1 JDK8的类加载器

#### 4.3.1.1 AppClassloader
AppClassloader也称为System ClassLoader，继承了URLClassLoader，是Java虚拟机默认的类加载器之一，主要用来加载用户类和第三方依赖包，在JVM启动命令行中设置-Djava.class.path参数来指定加载路径。
> 代码位置：src/share/classes/sun/misc/Launcher$AppClassLoader.java
```java
// AppClassLoader继承URLClassLoader
static class AppClassLoader extends URLClassLoader {
    
    public static ClassLoader getAppClassLoader(final ClassLoader extcl)
            throws IOException {
        // 搜索路径java.class.path
        final String s = System.getProperty("java.class.path");
        final File[] path = (s == null) ? new File[0] : getClassPath(s);

        URL[] urls = (s == null) ? new URL[0] : pathToURLs(path);
        return new AppClassLoader(urls, extcl);
    }

    /*
     * Creates a new AppClassLoader
     */
    AppClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent, factory);
    }

    /**
     * 重写了loadClass，支持类的包权限检查
     */
    public Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        int i = name.lastIndexOf('.');
        if (i != -1) {
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                sm.checkPackageAccess(name.substring(0, i));
            }
        }
        // 调用父类URLClassLoader完成类加载
        return (super.loadClass(name, resolve));
    }
    
    // 其他方法省略...
}    
```

#### 4.3.1.2 ExtClassLoader

ExtClassLoader称为扩展类加载器，继承了URLClassLoader，主要负责加载Java的扩展类库，默认加载${JAVA_HOME}/jre/lib/ext/
目录下的所有jar包，也可以用参数-Djava.ext.dirs来设置它的搜索路径。
> 代码位置：src/share/classes/sun/misc/Launcher$ExtClassLoader.java
```java
// ExtClassLoader继承URLClassLoader
static class ExtClassLoader extends URLClassLoader {

    public static ExtClassLoader getExtClassLoader() throws IOException {
        final File[] dirs = getExtDirs();

        try {
            return new ExtClassLoader(dirs);
        } catch (java.security.PrivilegedActionException e) {
            throw (IOException) e.getException();
        }
    }
    
    public ExtClassLoader(File[] dirs) throws IOException {
        super(getExtURLs(dirs), null, factory);
    }

    private static File[] getExtDirs() {
        // 通过系统变量指定加载路径
        String s = System.getProperty("java.ext.dirs");
        File[] dirs;
        if (s != null) {
            StringTokenizer st =
                    new StringTokenizer(s, File.pathSeparator);
            int count = st.countTokens();
            dirs = new File[count];
            for (int i = 0; i < count; i++) {
                dirs[i] = new File(st.nextToken());
            }
        } else {
            dirs = new File[0];
        }
        return dirs;
    }
}    
```

JDK8的类加载器的继承关系如下图4-3所示：

> 图4-3 JDK8的类加载器的继承关系

![图4-3 JDK8的类加载器的继承关系](images/%E5%9B%BE4-3%20JDK8%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

#### 4.3.1.3 JDK8的类加载器的初始化
JDK的类加载器的初始化在Launcher类中。

> 源码位置：src/share/classes/sun/misc/Launcher.java

```java
public class Launcher {

    public Launcher() {
        // 创建ExtClassLoader
        ClassLoader extcl = ExtClassLoader.getExtClassLoader();
        
        // 创建AppClassLoader
        ClassLoader loader = AppClassLoader.getAppClassLoader(extcl);
        
        // 设置当前线程的ContextClassLoader
        Thread.currentThread().setContextClassLoader(loader);

        // 异常处理的代码省略
        
    }
    
    // ...
}    
```
可以看到，初始化过程较为简单，先初始化ExtClassLoader，然后在初始化AppClassLoader，并且设置AppClassLoader的父加载器为ExtClassLoader。

### 4.3.2 JDK11的类加载器

JDK9实现模块化之后，对Classloader有所改造，其中一点就是将ExtClassLoader改为PlatformClassLoader，
模块化之后不同的Classloader加载各自对应的模块。因为JDK11是一个长期支持的稳定版本，这里以JDK11的源代码来说明类加载器的变化。JDK11的类加载器的继承关系如下图4-4所示：

> 图4-4 JDK11的类加载器的继承关系

![图4-4 JDK11的类加载器的继承关系](images/%E5%9B%BE4-4%20JDK11%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

#### 4.3.2.1 BuiltinClassLoader 

BuiltinClassLoader是PlatformClassLoader、BootClassLoader和AppClassloader的父类，功能上与URLClassLoader相似，都是基于UrlClassPath来实现类的查找，但BuiltinClassLoader还支持从模块中加载类。

BuiltinClassLoader的属性与构造函数如下：

> 代码位置：src/java.base/share/classes/jdk/internal/loader/BuiltinClassLoader.java

```java
// 类加载器路径
private final URLClassPath ucp;

BuiltinClassLoader(String name, BuiltinClassLoader parent, URLClassPath ucp) {
    // 确保当父加载器是bootloader时返回null
    // name 是类加载器的名称
    super(name, parent == null || parent == ClassLoaders.bootLoader() ? null : parent);

    this.parent = parent;
    this.ucp = ucp;
    
    this.nameToModule = new ConcurrentHashMap<>();
    this.moduleToReader = new ConcurrentHashMap<>();
}
```

BuiltinClassLoader也重写了loadClass方法，loadClass实际调用loadClassOrNull方法，来看下loadClassOrNull方法的实现。

> 源码位置：src/java.base/share/classes/jdk/internal/loader/BuiltinClassLoader.java
```java
protected Class<?> loadClassOrNull(String cn, boolean resolve) {
    // 加锁，保证线程安全
    synchronized (getClassLoadingLock(cn)) {
        // 先去找一次class是否已经被加载了，此方法是ClassLoader中的native方法
        Class<?> c = findLoadedClass(cn);
        if (c == null) {
            // 这里会需要去先加载模块信息
            LoadedModule loadedModule = findLoadedModule(cn);
            if (loadedModule != null) {
                BuiltinClassLoader loader = loadedModule.loader();
                if (loader == this) {
                    if (VM.isModuleSystemInited()) {
                        c = findClassInModuleOrNull(loadedModule, cn);
                    }
                } else {
                    // 委托其他类加载器加载
                    c = loader.loadClassOrNull(cn);
                }
            } else {
                // 先调用父加载器的相关方法去加载一次
                if (parent != null) {
                    c = parent.loadClassOrNull(cn);
                }

                // 如果没加载到，则用当前加载器去加载
                if (c == null && hasClassPath() && VM.isModuleSystemInited()) {
                    // 此方法内会调用到defineClas方法完成类的定义
                    c = findClassOnClassPathOrNull(cn);
                }
            }

        }

        if (resolve && c != null)
            resolveClass(c);

        return c;
    }
}
```
还有和通常的双亲委派不同，如果一个class属于某个module那么会直接调用该module的类加载器去加载，
而不是说直接用当前类加载器的双亲委派模型去加载。 但是找到这个class对应的类加载器后，还是会按照双亲委派去加载。

BuiltinClassLoader也重写了ClassLoader的findClass方法。

> 源码位置：src/java.base/share/classes/jdk/internal/loader/BuiltinClassLoader.java

```java
@Override
protected Class<?> findClass(String cn) throws ClassNotFoundException {
    
    // 在模块中尝试查找
    LoadedModule loadedModule = findLoadedModule(cn);

    Class<?> c = null;
    if (loadedModule != null) {
        //  加载任务委派给模块的加载器
        if (loadedModule.loader() == this) {
            c = findClassInModuleOrNull(loadedModule, cn);
        }
    } else {
        // 类路径下查找
        if (hasClassPath()) {
            c = findClassOnClassPathOrNull(cn);
        }
    }

    // 都没有找到，抛出异常
    if (c == null)
        throw new ClassNotFoundException(cn);

    return c;
}
```
其中findClassOnClassPathOrNull是在类路径下查找类。

> 源码位置：src/java.base/share/classes/jdk/internal/loader/BuiltinClassLoader.java

```java
    private Class<?> findClassOnClassPathOrNull(String cn) {
        String path = cn.replace('.', '/').concat(".class");
        // 权限检查代码省去...
        Resource res = ucp.getResource(path, false);
        if (res != null) {
            try {
              return defineClass(cn, res);
           } catch (IOException ioe) {
            // TBD on how I/O errors should be propagated
           }
        }
        return null;
    }
```

##### 4.3.2.2 BuiltinClassLoader的子类以及初始化

ClassLoaders类中分别初始化BootClassLoader、PlatformClassLoader和AppClassLoader类加载器。

> 源码位置：src/java.base/share/classes/jdk/internal/loader/ClassLoaders.java

```java
public class ClassLoaders {

    // JDK内置类加载器
    private static final BootClassLoader BOOT_LOADER;
    private static final PlatformClassLoader PLATFORM_LOADER;
    private static final AppClassLoader APP_LOADER;

    //  初始化类加载器对象
    static {
        // 可以使用 -Xbootclasspath/a 或者 -javaagent 中的Boot-Class-Path属性指定
        String append = VM.getSavedProperty("jdk.boot.class.path.append");
       // 初始化BOOT_LOADER
        BOOT_LOADER =
            new BootClassLoader((append != null && append.length() > 0)
                ? new URLClassPath(append, true)
                : null);
        
        // 初始化PLATFORM_LOADER并指定AppClassLoader的父加载器BOOT_LOADER        
        PLATFORM_LOADER = new PlatformClassLoader(BOOT_LOADER);

        // 获取classpath路径
        String cp = System.getProperty("java.class.path");
        if (cp == null || cp.length() == 0) {
            String initialModuleName = System.getProperty("jdk.module.main");
            cp = (initialModuleName == null) ? "" : null;
        }
        URLClassPath ucp = new URLClassPath(cp, false);
        // 初始化APP_LOADER并指定AppClassLoader的父加载器为PLATFORM_LOADER
        APP_LOADER = new AppClassLoader(PLATFORM_LOADER, ucp);
    }
  
    // ...
 }   
```

从类加载器实例的初始化代码可以看出，BootClassLoader用来加载`jdk.boot.class.path.append`参数指定的类，在初始化PLATFORM_LOADER是指定BOOT_LOADER为其父类，在初始化AppClassLoader是指定PLATFORM_LOADER为其父类，构成了类加载器的三层结构。

再来看下JDK9以上特有的PlatformClassLoader类：

```java
private static class PlatformClassLoader extends BuiltinClassLoader {
    
    PlatformClassLoader(BootClassLoader parent) {
        // 类加载器名称为platform
        super("platform", parent, null);
    }
    
    // ...
}
```

不同类加载器负责加载对应的模块。
+ BOOT_MODULES是由引导加载程序定义的模块：
> 代码来源：jdk11-1ddf9a99e4ad/make/common/Modules.gmk

```text
java.base               java.datatransfer
java.desktop            java.instrument
java.logging            java.management
java.management.rmi     java.naming
java.prefs              java.rmi
java.security.sasl      java.xml
jdk.internal.vm.ci      jdk.jfr
jdk.management          jdk.management.jfr
jdk.management.agent    jdk.net
jdk.sctp                jdk.unsupported
jdk.naming.rmi
```
+ PLATFORM_MODULES是由平台加载程序定义的模块：
```text
java.net.http           java.scripting  
java.security.jgss      java.smartcardio    
java.sql                java.sql.rowset
java.transaction.xa     java.xml.crypto
jdk.accessibility       jdk.charsets
jdk.crypto.cryptoki     jdk.crypto.ec
jdk.dynalink            jdk.httpserver
jdk.jsobject            jdk.localedata
jdk.naming.dns          jdk.scripting.nashorn
jdk.security.auth       jdk.security.jgss
jdk.xml.dom             jdk.zipfs
jdk.crypto.mscapi       jdk.crypto.ucrypto
java.compiler           jdk.aot
jdk.internal.vm.compiler
jdk.internal.vm.compiler.management
java.se
```

+ JRE_TOOL_MODULES是JRE中包含的工具，由AppClassLoader加载;
```java
jdk.jdwp.agent
jdk.pack
jdk.scripting.nashorn.shell
```
未列出的其他模块由AppClassLoader加载。

## 4.4 web容器的加载器

前面介绍了java中类加载的一般模型：双亲委派模型，这个模型适用于大多数类加载的场景，但对于web容器却是不适用的。这是因为servlet规范对web容器的类加载做了一些规定，简单的来说有以下几条：

+ WEB-INF/classes和WEB-INF/lib路径下的类会优先于父容器中的类加载，比如WEB-INF/classes下有个Abc类，CLASSPATH下也有个Abc类，web 容器加载器会优先加载位于WEB-INF/classes下的类，这与双亲委托模型的加载行为相反。
+ java.lang.Object等系统类不遵循第一条， WEB-INF/classes或WEB-INF/lib中的类不能替换系统类。对于哪些是系统类，其实没有做出具体规定， web容器通常是通过枚举了一些类来进行判断的。
+ web容器的自身的实现类不被应用中的类引用，即web容器的实现类不能被任何应用类加载器加载。对于哪些是web容器的类也是通过枚举包名称来进行判断。

### 4.4.1 Jetty类加载器

为了实现上面的三个要求并实现不同部署应用间依赖的隔离，Jetty定义了自己的类加载器WebAppClassLoader，类加载器的继承关系如下：
![图4-5 Jetty类加载器的继承关系.png](images%2F%E5%9B%BE4-5%20Jetty%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)
WebAppClassLoader的属性如下：
```java
// 类加载器上下文
private final Context _context;     
// 父加载器
private final ClassLoader _parent;  
// 加载文件的后缀 .zip或者.jar
private final Set<String> _extensions = new HashSet<String>(); 
// 加载器名称
private String _name = String.valueOf(hashCode()); 
// 类加载之前转换器
private final List<ClassFileTransformer> _transformers = new CopyOnWriteArrayList<>(); 
```

Jetty 中以类的 package 路径名来区分，当类的 package 路径名位包含于以下路径时，会被认为是系统类。系统类是对应用类可见。

```java
// 系统类不能被应用jar包中的类替换，并且只能被system classloader加载
public static final ClassMatcher __dftSystemClasses = new ClassMatcher(
    "java.","javax.","org.xml.","org.w3c."
);
```

Server 类不对任何应用可见，Jetty 同样是用 package 路径名来区分哪些是 Server 类。WebAppContext 中配置如下：

```java
// 使用system classloader加载，并且对web application不可见    
public static final ClassMatcher __dftServerClasses = new ClassMatcher(
    "org.eclipse.jetty."
);
```

我们可以通过，WebAppContext.addServerClasses或 WebAppContext.addServerClassMatcher方法设置 Server 类。 需要注意的是，Server 类是对所有应用都不可见，但是 WEB-INF/lib 下的应用类可以替换 Server 类。

> 代码位置：jetty-webapp/src/main/java/org/eclipse/jetty/webapp/WebAppContext.java

```java
public static void addServerClasses(Server server, String... pattern) {
    addClasses(__dftServerClasses, SERVER_SRV_CLASSES, server, pattern);
}       

public static void addSystemClasses(Server server, String... pattern) {
    addClasses(__dftSystemClasses, SERVER_SYS_CLASSES, server, pattern);
}  

public void addServerClassMatcher(ClassMatcher serverClasses) {
    _serverClasses.add(serverClasses.getPatterns());
}        

public void addSystemClassMatcher(ClassMatcher systemClasses) {
    _systemClasses.add(systemClasses.getPatterns());
}                                                            
```

WebAppClassLoader的构造函数如下：
```java
public WebAppClassLoader(ClassLoader parent, Context context) 
        throws IOException {
    // 指定父加载器
    super(new URL[]{}, parent != null ? parent
            : (Thread.currentThread().getContextClassLoader() != null ? Thread.currentThread().getContextClassLoader()
            : (WebAppClassLoader.class.getClassLoader() != null ? WebAppClassLoader.class.getClassLoader()
            : ClassLoader.getSystemClassLoader())));
    _parent = getParent();
    _context = context;
    if (_parent == null)
        throw new IllegalArgumentException("no parent classloader!");
    
    // 类加载器可以加载的文件类型：jar或者zip包 
    _extensions.add(".jar");
    _extensions.add(".zip");
}                                                                                                                     
```
构造函数可以显示指定父类加载器，默认情况下为空，即将当前的线程上下文classLoader指定为当前的parent，
而这个线程上下文classLoader如果没有用户指定的话默认又将是前面提到过的System ClassLoader。

再看下loadClass方法：
```java
@Override                                                                                                  
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {                 
    synchronized (getClassLoadingLock(name)) {
        ClassNotFoundException ex = null;
        Class<?> parentClass = null; // 来源于父加载器
        Class<?> webappClass = null; // 来源于webapp加载器

        // 先从已经加载的类中查找
        webappClass = findLoadedClass(name);
        if (webappClass != null) {
            return webappClass;
        }

        // 先尝试从当前类加载器加载（这里true表示检查类是否是系统类，如果不是，返回加载的类）
        webappClass = loadAsResource(name, true);
        if (webappClass != null) {
            return webappClass;
        }

        // 然后尝试当前类加载器的父加载器加载
        try {
            parentClass = _parent.loadClass(name);
            // 判断是否允许加载server类，或者当前类不是 server 类
            if (Boolean.TRUE.equals(__loadServerClasses.get()) 
                    || !_context.isServerClass(parentClass)) {
                return parentClass;
            }
        } catch (ClassNotFoundException e) {
            ex = e;
        }
        
        // 尝试从当前类加载器加载（这里false表示不检查类是否是系统类）
        webappClass = loadAsResource(name, false); 
        if (webappClass != null) {
            return webappClass;
        }
        throw ex == null ? new ClassNotFoundException(name) : ex;
    }
}                                                                                                          
```


### 4.4.2 Tomcat类加载器
与Jetty容器一样，Tomcat也需要遵循servlet三条规范。Tomcat的类加载器的继承关系如下：

![图4-6 Tomcat类加载器的继承关系.png](images%2F%E5%9B%BE4-6%20Tomcat%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

#### WebappClassLoader
> apache-tomcat-10.1.13-src/java/org/apache/catalina/loader/WebappLoader.java
```java
public class WebappClassLoader extends WebappClassLoaderBase {
    public WebappClassLoader() {
        super();
    }
    public WebappClassLoader(ClassLoader parent) {
        super(parent);
    } 
    
    //...
}
```

WebappClassLoader继承WebappClassLoaderBase，类加载的功能主要在WebappClassLoaderBase中实现。看下代码：
> 代码来源：apache-tomcat-10.1.13-src/java/org/apache/catalina/loader/WebappClassLoaderBase.java

先来看下构造函数：
```java

protected boolean delegate = false;

// 加载JavaSE的类加载器
private ClassLoader javaseClassLoader;

// 当前类加载器的父加载器
protected final ClassLoader parent;

protected WebappClassLoaderBase() {

    super(new URL[0]);
    // 初始化没有指定父加载器，则父加载器为系统类加载器
    ClassLoader p = getParent();
    if (p == null) {
        p = getSystemClassLoader();
    }
    this.parent = p;

    // 初始化javaseClassLoader为平台类加载器或者扩展类加载器
    ClassLoader j = String.class.getClassLoader();
    if (j == null) {
        j = getSystemClassLoader();
        while (j.getParent() != null) {
            j = j.getParent();
        }
    }
    this.javaseClassLoader = j;

    securityManager = System.getSecurityManager();
    if (securityManager != null) {
        refreshPolicy();
    }
}
```
再来看下重写的loadClass方法。
```java

```

```java
public abstract class WebappClassLoaderBase extends URLClassLoader
        implements Lifecycle, InstrumentableClassLoader, WebappProperties, PermissionCheck {
	// ...	省略不需要关注的代码
    protected WebappClassLoaderBase() {

        super(new URL[0]);
		// 获取当前WebappClassLoader的父加载器
        ClassLoader p = getParent();
        if (p == null) {
            p = getSystemClassLoader();
        }
        this.parent = p;
		
        // 设置javaseClassLoader为平台类加载器或者扩展类加载器
        ClassLoader j = String.class.getClassLoader();
        if (j == null) {
            j = getSystemClassLoader();
            while (j.getParent() != null) {
                j = j.getParent();
            }
        }
        this.javaseClassLoader = j;

        // 权限代码省战略...
    }

    // 省略不需要关注的代码...
    @Override
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            
            Class<?> clazz = null;

			// 本地类缓存中查找
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                return clazz;
            }

            // Web应用程序本地类缓存中没有，可以从系统类加载器缓存中查找，
			// 如果找到说明AppClassLoader之前已经加载过这个类
            clazz = findLoadedClass(name);
            if (clazz != null) {
                return clazz;
            }

			// 将类似java.lang.String这样的类名这样转换成java/lang/String
            String resourceName = binaryNameToPath(name, false);
			// 获取引导类加载器（BootstrapClassLoader）
            ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {
		        // 引导类加载器根据转换后的类名获取资源url，如果url不为空，就说明找到要加载的类
                URL url;
                if (securityManager != null) {
                    PrivilegedAction<URL> dp = new PrivilegedJavaseGetResource(resourceName);
                    url = AccessController.doPrivileged(dp);
                } else {
                    url = javaseLoader.getResource(resourceName);
                }
                tryLoadingFromJavaseLoader = (url != null);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                tryLoadingFromJavaseLoader = true;
            }

           // 首先，从扩展类加载器（ExtClassLoader）加载
           if (tryLoadingFromJavaseLoader) {
               return javaseLoader.loadClass(name);
            }
            
            //   delegate允许类委托给父类加载
            boolean delegateLoad = delegate || filter(name, true);
            
            if (delegateLoad) {
                return Class.forName(name, false, parent);
            }

            // 在当前web路径加载
            return clazz = findClass(name);

            // 经过上面几个步骤还未加载到类，则采用系统类加载器（也称应用程序类加载器）进行加载
            if (!delegateLoad) {
               return Class.forName(name, false, parent);
            }
        }
        // 最终，还未加载到类，报类未找到的异常
        throw new ClassNotFoundException(name);
    }
	// ...
}
```

#### 4.2.2.2 JSP类加载器（JasperLoader）

```java
public class JasperLoader extends URLClassLoader {

    private final PermissionCollection permissionCollection;
    private final SecurityManager securityManager;

    // JSP类加载器的父加载器是Web应用程序类加载器（WebappClassLoader）
    public JasperLoader(URL[] urls, ClassLoader parent,
                        PermissionCollection permissionCollection) {
        super(urls, parent);
        this.permissionCollection = permissionCollection;
        this.securityManager = System.getSecurityManager();
    }

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    @Override
    public synchronized Class<?> loadClass(final String name, boolean resolve)
        throws ClassNotFoundException {

        Class<?> clazz = null;

        // 从JVM的类缓存中查找
        clazz = findLoadedClass(name);
        if (clazz != null) {
            if (resolve) {
                resolveClass(clazz);
            }
            return clazz;
        }

        // 当使用SecurityManager安全管理器时，允许访问访类
        if (securityManager != null) {
            int dot = name.lastIndexOf('.');
            if (dot >= 0) {
                try {
                    // Do not call the security manager since by default, we grant that package.
                    if (!"org.apache.jasper.runtime".equalsIgnoreCase(name.substring(0,dot))){
                        securityManager.checkPackageAccess(name.substring(0,dot));
                    }
                } catch (SecurityException se) {
                    String error = "Security Violation, attempt to use " +
                        "Restricted Class: " + name;
                    se.printStackTrace();
                    throw new ClassNotFoundException(error);
                }
            }
        }
       // 如果类名不是以org.apache.jsp包名开头的，则采用WebappClassLoader加载
        if( !name.startsWith(Constants.JSP_PACKAGE_NAME + '.') ) {
            // Class is not in org.apache.jsp, therefore, have our
            // parent load it
            clazz = getParent().loadClass(name);
            if( resolve ) {
                resolveClass(clazz);
            }
            return clazz;
        }
	// 如果是org.apache.jsp包名开头JSP类，就调用父类URLClassLoader的findClass方法
	// 动态加载类文件，解析成Class类，返回给调用方
        return findClass(name);
    }
}

```
从源码中我们可以看到，JSP类加载原理是，先从JVM类缓存中（也就是Bootstrap类加载器加载的类）加载，如果不是核心类库的类，就从Web应用程序类加载器WebappClassLoader中加载，如果还未找到，就说明是jsp类，则通过动态解析jsp类文件获得要加载的类。

经过上面两个Tomcat核心类加载器的剖析，我们也就知道了Tomcat类的加载原理了。
下面我们来总结一下：Tomcat会为每个Web应用程序创建一个WebappClassLoader类加载器进行类的加载，不同的类加载器实例加载的类是会被认为是不同的类，即使它们的类名相同，这样的话就可以实现在同一个JVM下，允许Tomcat容器的不同部分以及在容器上运行的不同Web应用程序可以访问的各种不同版本的类库。
针对JSP类，会由专门的JSP类加载器（JasperLoader）进行加载，该加载器会针对JSP类在每次加载时都会解析类文件，Tomcat容器会启动一个后台线程，定时检测JSP类文件的变化，及时更新类文件，这样就实现JSP文件的热加载功能。

## 4.5 热加载与卸载

在类的加载过程中，我们知道会先检查该类是否已经加载，如果已经加载了，则不会从jar包或者路径上查找类，而是使用缓存中的类。JVM表示一个类是否是相同的类有两个条件：第一个是类的全限定名称是否相同，第二个是类的加载器实例是否是同一个。 因此要实现类的热加载，可以使用不同的类加载器来加载同一个类文件。

使用不同的类加载器实例加载同一个类文件，随着加载次数增加，类的个数也会不断增加，如果不及时清理元空间/永久代，会有内存溢出的风险。
然而类卸载的条件非常苛刻，一般要同时具备下面的三个条件才可以卸载，并且需要JVM执行fullgc后才能完全清除干净。类卸载的三个条件（来源于JVM虚拟机规范）。

+ 该类所有的实例都已经被GC；

+ 加载该类的ClassLoader实例已经被GC；

+ 该类的java.lang.Class对象没有在任何地方被引用；

full GC的时机我们是不可控的，那么同样的我们对于Class的卸载也是不可控的。从上面的三个条件可以看出JVM自带的类加载器不会被回收，因此JVM的类不会被卸载。只有自定义类加载器才有卸载的可能。下面给出一个具体的需求，并使用热加载来完成。应用在运行时加载一个class脚本，class脚本可以做到热更新。有这样一个脚本接口，具有获取版本号和执行运算的功能。

```java
public interface Script {
    // 执行运算
    String run(String key);
}
```

脚本的实现类，负责具体的计算功能。
```java
public class ScriptImpl implements Script {

    public ScriptImpl() {
    }

    public String run(String key) {
        return key;
    }
}

```

JVM运行过程中替换脚本的实现，既可以实现脚本的更新功能。
```java
public class Main {
    public static void main(String[] args) throws Exception {
        ClassLoader appClassloader = Main.class.getClassLoader();

        ScriptClassLoader scriptClassLoader1 = new ScriptClassLoader("resources", appClassloader);
        Class<?> scriptImpl1 = scriptClassLoader1.loadClass("ScriptImpl");
        System.out.println(scriptImpl1.hashCode());

        ScriptClassLoader scriptClassLoader2 = new ScriptClassLoader("resources", appClassloader);
        Class<?> scriptImpl2 = scriptClassLoader2.loadClass("ScriptImpl");
        
        // class对象不相同
        assert scriptImpl1 != scriptImpl2;
    }
}
```
使用不同的类加载器还在同一个类，得到的class对象不一样。运行时更新 ScriptImpl 类的实现即可ScriptClassLoader的实现如下：

```java
public class ScriptClassLoader extends ClassLoader {
    private String classDir;

    public ScriptClassLoader(String classDir,ClassLoader classLoader) {
        super(classLoader);
        this.classDir = classDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] classDate = getDate(name);
            if (classDate == null) {
                return null;
            }
            return defineClass(name, classDate, 0, classDate.length);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private byte[] getDate(String className) throws IOException {
        InputStream in = null;
        ByteArrayOutputStream out = null;
        String path = classDir + File.separatorChar +
                className.replace('.', File.separatorChar) + ".class";
        try {
            in = new FileInputStream(path);
            out = new ByteArrayOutputStream();
            byte[] buffer = new byte[2048];
            int len = 0;
            while ((len = in.read(buffer)) != -1) {
                out.write(buffer, 0, len);
            }
            return out.toByteArray();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } finally {
            in.close();
            out.close();
        }
        return null;
    }
}
```
