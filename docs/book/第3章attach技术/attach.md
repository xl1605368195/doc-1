# 第3章 Attach机制

Attach机制从JDK1.6开始引入，主要是给运行中的Java进程注入一个Java Agent。
Java Agent有着广泛的使用场景， 如 Java性能诊断工具jstack、jmap 和Arthas等都使用了该技术。

本章将从Attach API的基本使用、实现原理、开源工具和常见的坑等几个方面介绍Attach技术。

## 3.1 Attach API 简介
从JDK1.6开始可以使用Attach API连接到目标JVM上并让目标JVM加载一个Java Agent。
Attach API的包名称为`com.sun.tools.attach`。如下图3-1所示主要包含2个类：VirtualMachine和VirtualMachineDescriptor。

> 图3-1 Attach API 官方文档

![图3-1 Attach API 官方文档](images/图3-1 Attach API 官方文档.png)

VirtualMachine代表一个Java虚拟机，也就是监控的目标虚拟机，而VirtualMachineDescriptor用来描述虚拟机信息，配合VirtualMachine类完成各种功能。

主要的功能实现在`VirtualMachine`以及子类中，其它类起到辅助作用。下面将重点介绍VirtualMachine类的使用。下面的代码使用Attach API连接到进程pid为72695的JVM进程上，然后读取目标JVM的系统参数并输出到终端，最后调用detach与目标JVM断开连接。

```java
import java.util.Properties;

import com.sun.tools.attach.VirtualMachine;

public class Main {

    public static void main(String[] args) throws Exception {
        // attach to target VM
        VirtualMachine vm = VirtualMachine.attach("72695");

        // read target vm system properties
        Properties properties = vm.getSystemProperties();
        for (Object key : properties.keySet()) {
            System.out.println(key + "=" + properties.getProperty(key.toString()));
        }

        // detach
        vm.detach();
    }
}
```
上面代码输出目标JVM的系统属性参数，其结果如下所示。

```
java.runtime.name=Java(TM) SE Runtime Environment
java.protocol.handler.pkgs=org.springframework.boot.loader
java.vm.version=25.261-b12
gopherProxySet=false
java.vm.vendor=Oracle Corporation
// ... 其他参数省略
```

上面代码第9行处，可以直观的理解在调用attach方法之后，就获得了一个目标JVM的VirtualMachine对象，调用VirtualMachine对象的方法（第12行处调用getSystemProperties方法）就可以完成对目标JVM的操作。除了获取目标 JVM 系统参数的方法之外，VirtualMachine还有如下方法，如下所示。

```text
// 列出当前主机上的所有JVM
public static List<VirtualMachineDescriptor> list()
// 执行attach/detach
public static VirtualMachine attach(VirtualMachineDescriptor vmd)
public abstract void detach() throws IOException
// 加载Agent
public abstract void loadAgentLibrary(String agentLibrary, String options)
public void loadAgentLibrary(String agentLibrary)
public abstract void loadAgentPath(String agentPath, String options)
public void loadAgentPath(String agentPath)
public abstract void loadAgent(String agent, String options)
public void loadAgent(String agent)
// 获取JVM系统参数
public abstract Properties getSystemProperties() throws IOException
public abstract Properties getAgentProperties() throws IOException
// 在目标虚拟机中启动JMX管理代理
public abstract void startManagementAgent(Properties agentProperties) throws IOException
public abstract String startLocalManagementAgent() throws IOException;
```

## 3.2 实现原理
在上一节介绍了Attach API的基本使用，本节将结合JDK源码分析其中的原理。Attach机制本质上是进程间的通信，外部进程通过JVM提供的socket连接到目标JVM上并发送指令，目标JVM接受并处理指令然后返回处理结果。

### 3.2.1 Attach客户端源码解析

有了前面一节的使用API使用基础，我们将分析Attach API的实现原理并对相应的源码做解析，从而挖掘更多可用的功能。`VirtualMachine`是抽象类，不同厂商的虚拟机可以实现不同VirtualMachine子类，HotSpotVirtualMachine是HotSpot官方提供的VirtualMachine实现，它也是一个抽象类，在不同操作系统上还有各自实现，如Linux系统上的JDK11上实现类的名称为VirtualMachineImpl（JDK8上实现类名称为LinuxVirtualMachine）。JDK8上VirtualMachine实现类的的继承关系如下图3-2所示：

> 图3-2 VirtualMachine实现类的继承关系

![图3-2 VirtualMachineImpl继承关系.png](images%2F%E5%9B%BE3-2%20VirtualMachineImpl%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

先来看下`HotSpotVirtualMachine`类的loadAgentLibrary方法

> 代码位置：src/jdk.attach/share/classes/sun/tools/attach/HotSpotVirtualMachine.java

```java
private void loadAgentLibrary(String agentLibrary, boolean isAbsolute, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    if (agentLibrary == null) {
        throw new NullPointerException("agentLibrary cannot be null");
    }
    
    // jdk11返回字符串"return code: 0"
    String msgPrefix = "return code: ";
    // 执行load指令，给目标 jvm 传输 agent jar路径和参数
    InputStream in = execute("load",
                             agentLibrary,
                             isAbsolute ? "true" : "false",
                             options);
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(in))) {
        String result = reader.readLine();
        // 返回结果
        if (result == null) {
            throw new AgentLoadException("Target VM did not respond");
        } else if (result.startsWith(msgPrefix)) {
            int retCode = Integer.parseInt(result.substring(msgPrefix.length()));
            if (retCode != 0) {
                throw new AgentInitializationException("Agent_OnAttach failed", retCode);
            }
        } else {
            throw new AgentLoadException(result);
        }
    }
}
```

上面的代码是加载一个Java Agent，核心实现在 `execute` 方法中，来看下execute方法的源码：
```java
// 在目标JVM上执行给定的命令，需要由子类来实现
abstract InputStream execute(String cmd, Object ... args)
    throws AgentLoadException, IOException;
```
execute是一个抽象方法，需要在子类中实现，HotSpotVirtualMachine类中的其他方法大多数最终都会调用这个execute方法。

再来看下Linux系统上的实现类`LinuxVirtualMachine`代码。

> 代码位置：src/jdk.attach/linux/classes/sun/tools/attach/VirtualMachineImpl.java

``` java    
VirtualMachineImpl(AttachProvider provider, String vmid)
    throws AttachNotSupportedException, IOException
{
    super(provider, vmid);

    int pid;
    try {
        pid = Integer.parseInt(vmid);
    } catch (NumberFormatException x) {
        throw new AttachNotSupportedException("Invalid process identifier");
    }
    // 在/tmp目录下寻找socket文件是否存在                    
    File socket_file = new File(tmpdir, ".java_pid" + pid);
    socket_path = socket_file.getPath();
    if (!socket_file.exists()) {
        // 创建 attach_pid 文件
        File f = createAttachFile(pid);
        try {
            // 向目标JVM 发送 kill -3 信号
            sendQuitTo(pid);

            // 等待目标JVM创建socket文件
            final int delay_step = 100;
            final long timeout = attachTimeout();
            long time_spend = 0;
            long delay = 0;
            do {
                // 计算等待时间
                delay += delay_step;
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException x) { }

                time_spend += delay;
                if (time_spend > timeout/2 && !socket_file.exists()) {
                    sendQuitTo(pid); // 发送kill -3 信号
                }
            } while (time_spend <= timeout && !socket_file.exists());
            
            // 等待时间结束后，确认socket文件是否存在
            if (!socket_file.exists()) {
                throw new AttachNotSupportedException(
                    String.format("Unable to open socket file %s: " +
                                  "target process %d doesn't respond within %dms " +
                                  "or HotSpot VM not loaded", socket_path,
                                  pid, time_spend));
            }
        } finally {
            // 最后删除 attach_pid 文件
            f.delete();
        }
    }

    // 确认socket文件权限
    checkPermissions(socket_path);

    // 尝试连接socket，确认可以连接到目标JVM
    int s = socket();
    try {
        connect(s, socket_path);
    } finally {
        close(s);
    }
}
```
再次梳理下attach通信的过程：

第一步： 发起attach的进程在/tmp目录下查找目标JVM是否已经创建了.java_pid<pid> ，如果已经创建，直接跳到第六步；

第二步： attach进程创建socket通信的握手文件.attach_pid<pid>；

第三步： attach进程给目标JVM发送SIGQUIT（kill -3）信号，提示目标JVM外部进程发起了attach请求；

第四步： attach进程循环等待目标JVM创建.java_pid<pid>文件；

第五步： 删除握手文件.attach_pid<pid>文件；

第六步： attach进程校验socket文件权限；

第七步： attach进程测试socket连接可用性；

上面详细说明了socket连接的建立过程，下面将介绍发送命令的协议。

> 代码位置：src/jdk.attach/linux/classes/sun/tools/attach/VirtualMachineImpl.java

```java
InputStream execute(String cmd, Object ... args) throws AgentLoadException, IOException {
    // 参数、socket_path校验
        
    // create UNIX socket
    int s = socket();

    // connect to target VM
    try {
        connect(s, socket_path);
    } catch (IOException x) {
        // 错误处理
    }

    IOException ioe = null;

    // 发送attach请求信息
    try {
        // 发送协议
        writeString(s, PROTOCOL_VERSION);
        // 发送命令
        writeString(s, cmd);
        // 发送参数，最多三个参数
        for (int i=0; i<3; i++) {
            if (i < args.length && args[i] != null) {
                writeString(s, (String)args[i]);
            } else {
                // 没有参数，发送空字符串代替
                writeString(s, "");
            }
        }
    } catch (IOException x) {
        ioe = x;
    }

    // 读取执行结果
    SocketInputStream sis = new SocketInputStream(s);

    // 读取命令的执行状态
    int completionStatus;
    try {
        completionStatus = readInt(sis);
    } catch (IOException x) {
        // 错误处理
    }

    if (completionStatus != 0) {
        // 错误处理
    }
    
    return sis;
}
```
从上面的代码可以知道一次命令发送，先发送版本协议，然后是命令，最后是参数，并且参数的个数最多为3个。

为了更加清晰的看到通信协议的内容，在Linux上使用strace命令能够跟踪attach的系统调用过程。

```shell
strace -f java Main 2> main.out
```

在 main.out 文件中找到attach通信过程，从开始写入部分可以看出依次先写入协议号、命令、命令参数，
然后读取返回结果。
```text
// 建立UDS链接
[pid 31412] socket(AF_LOCAL, SOCK_STREAM, 0) = 6
[pid 31412] connect(6, {sa_family=AF_LOCAL, sun_path="/tmp/.java_pid27730"}, 110) = 0
// 开始写入
[pid 31412] write(6, "1", 1)            = 1   // 协议号
[pid 31412] write(6, "\0", 1)           = 1   // 分割符号
[pid 31412] write(6, "properties", 10)  = 10  // 命令
[pid 31412] write(6, "\0", 1)           = 1   // 分割符号
[pid 31412] write(6, "\0", 1 <unfinished ...> // 参数1
[pid 31412] write(6, "\0", 1)           = 1   // 参数2
[pid 31412] write(6, "\0", 1)           = 1   // 参数3
// 读取返回结果
[pid 31412] read(6, "0", 1)             = 1
[pid 31412] read(6, "\n", 1)            = 1
[pid 31412] read(6, "#Thu Jul 27 17:52:11 CST 2023\nja"..., 128) = 128
[pid 31412] read(6, "oot.loader\nsun.boot.library.path"..., 128) = 128
[pid 31412] read(6, "poration\njava.vendor.url=http\\:/"..., 128) = 128
[pid 31412] read(6, ".pkg=sun.io\nuser.country=CN\nsun."..., 128) = 128
[pid 31412] read(6, "e=Java Virtual Machine Specifica"..., 128) = 128
```

因此Attach客户端的发送协议可以总结为下面的字符串序列。
```text
1 byte PROTOCOL_VERSION
1 byte '\0'
n byte command
1 byte '\0'
n byte arg1
1 byte '\0'
n byte arg2
1 byte '\0'
n byte arg3
1 byte '\0'
```
### 3.2.2 Attach服务端源码解析

我们再来看下接收Attach命令的服务端是如何实现的，这部分代码是c/c++语言，但是也是不难理解的。
以Linux系统为例子，说明目标JVM如何处理Attach请求和执行指定的命令。

Linux系统下Attach机制信号与线程的创建流程可以描述为下图3-3。

> 图3-3 Attach机制信号与线程的处理流程

![图3-3 Attach机制信号与线程的处理流程](images/图3-3 Attach机制信号与线程的处理流程.png)

先来看下目标JVM如何处理`kill -3`信号。JVM初始化过程中会创建2个线程，线程名称分别为`Signal Dispatcher`和`Attach Listener`，Signal Dispatcher线程用来处理信号量，Attach Listener线程用来响应Attach请求。

JVM线程的的初始化都在`Threads::create_vm`中，当然与Attach有关的线程也在这个方法中初始化。

> 代码位置：src/hotspot/share/runtime/thread.cpp

```c++
// 代码位置 src/hotspot/share/runtime/thread.cpp
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {

  // 参数和系统初始化，省略....

  // 初始化Signal Dispatcher线程支持信号量处理
  os::initialize_jdk_signal_support(CHECK_JNI_ERR);

  // 目标JVM没有禁用Attach机制
  if (!DisableAttachMechanism) {
  	// 在JVM启动时删除已经存在的通信文件.java_pid<pid>
    AttachListener::vm_start();
    
    // 如果JVM启动参数设置-XX:+StartAttachListener或者
    // 减少了信号量的使用而不能延迟启动，则在JVM启动时初始化Attach Listener
    // 默认情况下AttachListener是延迟启动模式，即在JVM启动时不会立即创建Attach Listener线程
    if (StartAttachListener || AttachListener::init_at_startup()) {
      // 初始化Attach Listener线程
      AttachListener::init();
    }
  }
  
  // 参数和系统初始化，省略....
}  
```
上面的代码中分别初始化Signal Dispatcher和Attach Listener线程，Signal Dispatcher在JVM 启动时初始化，Attach Listener则延迟初始化。下面分别详细说下各自的初始化流程。

#### 3.2.2.1 Signal Dispatcher线程

`initialize_jdk_signal_support`的实现代码如下所示：

> 代码位置：src/hotspot/share/runtime/os.cpp

```c++
// 代码位置 src/hotspot/share/runtime/os.cpp
// 初始化JDK的信号支持系统
void os::initialize_jdk_signal_support(TRAPS) {
  // 没有禁止信号量的使用
  if (!ReduceSignalUsage) {
  
    // 线程名称 Signal Dispatcher
    const char thread_name[] = "Signal Dispatcher";
    
    // ... 线程初始化过程

    // 设置线程入口 signal_thread_entry
    JavaThread* signal_thread = new JavaThread(&signal_thread_entry);
    
    // ...
    
    // 注册SIGBREAK信号处理handler
    os::signal(SIGBREAK, os::user_handler());
  }
}
```
JVM创建了一个单独的线程来实现信号处理，这个线程名称为Signal Dispatcher。该线程的入口是signal_thread_entry函数。入口函数代码：

>代码清单：Signal Dispatcher线程的入口
>
>代码位置 src/hotspot/share/runtime/os.cpp

```c++
#ifndef SIGBREAK
#define SIGBREAK SIGQUIT  // SIGBREAK就是SIGQUIT
#endif

// Signal Dispatcher线程的入口
static void signal_thread_entry(JavaThread* thread, TRAPS) {
  os::set_priority(thread, NearMaxPriority);
  // 处理信号
  while (true) {
    int sig;
    {
      sig = os::signal_wait(); //阻塞等待信号
    }
    if (sig == os::sigexitnum_pd()) {
       // 停止Signal Dispatcher信号处理线程
       return;
    }
    
    // 循环处理各种信号
    switch (sig) {
    	// 当接收到SIGBREAK信号，就执行接下来的代码
      case SIGBREAK: {
        
        // 如果没有禁用attach机制并且是attach请求则初始化AttachListener
        // 如果AttachListener没有初始化，则进行初始化并返回true
        if (!DisableAttachMechanism && AttachListener::is_init_trigger()) {
          continue;
        }
        
        // 如果上面条件不满足，则打印线程栈等信息
        VM_PrintThreads op;
        VMThread::execute(&op);    // 线程栈信息
        VM_PrintJNI jni_op;
        VMThread::execute(&jni_op);// JNI global references数量
        VM_FindDeadlocks op1(tty);
        VMThread::execute(&op1);   // 死锁信息
        Universe::print_heap_at_SIGBREAK(); // 堆、元空间的使用占比
        
        // 启用-XX:+PrintClassHistogram，则强制执行一次full GC
        if (PrintClassHistogram) {
          // 下面的true表示force full GC before heap inspection
          VM_GC_HeapInspection op1(tty, true);
          VMThread::execute(&op1);
        }
        if (JvmtiExport::should_post_data_dump()) {
          JvmtiExport::post_data_dump();
        }
        break;
      }
      default: {
        // Dispatch the signal to java
        // ...其他信号处理
      }
    }
  }
}
```
代码行号1～3定义了宏SIGBREAK，可以看出，SIGBREAK信号就是SIGQUIT。代码26行的DisableAttachMechanism参数可以禁止attach，默认为false，即允许attach。

再来看下`AttachListener::is_init_trigger`的实现。

> 代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
// 如果在JVM工作目录或者/tmp目录下存在文件.attach_pid<pid>
// 表示是启动attach机制
bool AttachListener::is_init_trigger() {
  // 记录AttachListener的初始状态
  // JVM 用一个全局变量_is_initialized记录AttachListener 的状态
  if (init_at_startup() || is_initialized()) {
    // AttachListener在JVM启动时已经初始化或者已经是初始化的状态
    return false;               
  }
  
  // 检查.attach_pid<pid>是否存在
  char fn[PATH_MAX + 1];
  int ret;
  struct stat64 st;
  sprintf(fn, ".attach_pid%d", os::current_process_id());
  RESTARTABLE(::stat64(fn, &st), ret);
  if (ret == -1) {
    // .attach_pid<pid>文件不存在，打印日志，代码省略...
  }
  
  // 当前进程的.attach_pid<pid>文件存在，创建AttachListener线程
  if (ret == 0) {
    // attach文件权限校验（root权限或者权限相同）
    if (os::Posix::matches_effective_uid_or_root(st.st_uid)) {
      // 创建AttachListener线程
      init();
      return true;
    } 
  }
  return false;
}
```

在Signal Dispatcher线程接收到SIGBREAK信号后，有两种处理方法，第一种是初始化AttachListener线程；第二种打印线程栈等快照信息。处理方式取决于.attach_pid<pid>握手文件是否存在，如果存在则调用AttachListener的init方法，初始化成功后返回true。

#### 3.2.2.2 Attach Listener
Attach机制通过Attach Listener线程来进行相关命令的处理，下面来看一下Attach Listener线程是如何初始化的。从上面的代码分析可以看出，AttachListener可以在JVM启动时（立即初始化），也可以在首次收到SIGBREAK信号后，由Signal Dispatcher线程完成初始化（延迟初始化）。

来看下Attach Listener初始化过程。

>代码清单：Attach Listener初始化过程
>
>代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
void AttachListener::init() {
  
  // 线程名称Attach Listener
  const char thread_name[] = "Attach Listener";
  
  // ... 线程初始化过程
  
  // 设置AttachListener线程的入口函数attach_listener_thread_entry
  JavaThread* listener_thread = new JavaThread(&attach_listener_thread_entry);
  
  // ... 设置线程状态
}
```

上面的代码初始化了一个线程，并设置线程的入口函数。重点分析下attach_listener_thread_entry函数。

> 代码位置：src/hotspot/share/services/attachListener.cpp

```c++
// Attach Listener线程从队列中获取操作命令，并执行命令对应的函数
static void attach_listener_thread_entry(JavaThread* thread, TRAPS) {
  // STEP1：AttachListener初始化
  if (AttachListener::pd_init() != 0) {
    return;
  }
  // STEP2：设置AttachListener的全局状态
  AttachListener::set_initialized();

  for (;;) {
    // STEP3：从队列中取AttachOperation
    AttachOperation* op = AttachListener::dequeue();
    // find the function to dispatch too
    AttachOperationFunctionInfo* info = NULL;
    for (int i=0; funcs[i].name != NULL; i++) {
      const char* name = funcs[i].name;
      if (strcmp(op->name(), name) == 0) {
        info = &(funcs[i]); break;
      }}
      // dispatch to the function that implements this operation
      // ... 执行具体的操作
      res = (info->func)(op, &st);
    //...
  }
}
```
第一步：先执行AttachListener socket的初始化操作；

第二步：初始化完成后设置，AttachListener的状态为initialized；

第三步：从队列中取AttachOperation，并且调用对应的处理函数处理并返回结果。

下面分别对这个三个过程详细分析。

##### AttachListener::pd_init

> 代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
int AttachListener::pd_init() {
  
  // linux 系统下的初始化操作  
  int ret_code = LinuxAttachListener::init();
  
  // ...
  
  return ret_code;
}
```
实际执行的是LinuxAttachListener::init，不同操作系统执行初始化逻辑不同。在Linux系统中实际执行LinuxAttachListener::init。

> 代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
// 创建了一个socket并监听socket文件
int LinuxAttachListener::init() {
  char path[UNIX_PATH_MAX];          // socket file
  char initial_path[UNIX_PATH_MAX];  // socket file during setup
  int listener;                      // listener socket (file descriptor)

  // register function to cleanup
  ::atexit(listener_cleanup);

  int n = snprintf(path, UNIX_PATH_MAX, "%s/.java_pid%d",
                   os::get_temp_directory(), os::current_process_id());
  if (n < (int)UNIX_PATH_MAX) {
    n = snprintf(initial_path, UNIX_PATH_MAX, "%s.tmp", path);
  }
  if (n >= (int)UNIX_PATH_MAX) {
    return -1;
  }

  // create the listener socket
  listener = ::socket(PF_UNIX, SOCK_STREAM, 0);
  if (listener == -1) {
    return -1;
  }

  // 绑定socket
  struct sockaddr_un addr;
  memset((void *)&addr, 0, sizeof(addr));
  addr.sun_family = AF_UNIX;
  strcpy(addr.sun_path, initial_path);
  ::unlink(initial_path);
  int res = ::bind(listener, (struct sockaddr*)&addr, sizeof(addr));
  if (res == -1) {
    ::close(listener);
    return -1;
  }

  // 开启监听
  res = ::listen(listener, 5);
  if (res == 0) {
    RESTARTABLE(::chmod(initial_path, S_IREAD|S_IWRITE), res);
    if (res == 0) {
      // make sure the file is owned by the effective user and effective group
      // e.g. the group could be inherited from the directory in case the s bit is set
      RESTARTABLE(::chown(initial_path, geteuid(), getegid()), res);
      if (res == 0) {
        res = ::rename(initial_path, path);
      }
    }
  }n'n'n'h
  if (res == -1) {
    ::close(listener);
    ::unlink(initial_path);
    return -1;
  }
  set_path(path);
  set_listener(listener);

  return 0;
}
```
AttachListener::pd_init()方法调用了LinuxAttachListener::init()方法，完成了套接字的创建和监听。

##### LinuxAttachListener::dequeue

for循环的执行逻辑，流程简略的概括为下面的步骤：
+ 从dequeue拉取一个需要执行的AttachOperation对象；
+ 查询匹配的命令处理函数；
+ 执行匹配到的命令执行函数并返回结果；

AttachOperation的全部操作函数表如下：

> 代码位置：src/hotspot/share/services/attachListener.cpp

```c++
static AttachOperationFunctionInfo funcs[] = {
  { "agentProperties",  get_agent_properties },
  { "datadump",         data_dump },
  { "dumpheap",         dump_heap },
  { "load",             load_agent },
  { "properties",       get_system_properties },
  { "threaddump",       thread_dump },
  { "inspectheap",      heap_inspection },
  { "setflag",          set_flag },
  { "printflag",        print_flag },
  { "jcmd",             jcmd },
  { NULL,               NULL }
};
```
对于加载Agent来说，对应的命令就是上面的load。现在，我们知道了Attach Listener大概的工作模式，但是还是不太清楚任务从哪来，这个秘密就藏在AttachListener::dequeue这行代码里面，接下来我们来分析一下dequeue这个函数：

> 代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
LinuxAttachOperation* LinuxAttachListener::dequeue() {
  for (;;) {
    // 等待attach进程连接socket
    struct sockaddr addr;
    socklen_t len = sizeof(addr);
    RESTARTABLE(::accept(listener(), &addr, &len), s);
    // 校验attach进程的权限
    struct ucred cred_info;
    socklen_t optlen = sizeof(cred_info);
    if (::getsockopt(s, SOL_SOCKET, SO_PEERCRED, (void*)&cred_info, &optlen) == -1) {
      ::close(s);
      continue;
    }
    // 读取socket获取操作的对象
    LinuxAttachOperation* op = read_request(s);
    return op;
  }
}
```
dequeue方法是一个for循环，会循环使用accept方法，接受socket中传过来的数据，并且在验证通信的另一端的uid与gid与自身的euid与egid相同后，执行read_request方法，从socket读取内容，并且把内容包装成AttachOperation类的一个实例。接下来看看read_request是如何解析socket数据流的。

> 代码位置：src/hotspot/os/linux/attachListener_linux.cpp

```c++
LinuxAttachOperation* LinuxAttachListener::read_request(int s) {
  // 缓存区最大长度计算，省略...
  
  char buf[max_len];
  int str_count = 0;
  
  // 数据流写入buf
  // 包括版本去掉命令数据的分割符号代码"\0"
  // 版本协议校验等，省略...

  // 参数遍历
  ArgumentIterator args(buf, (max_len)-left);

  // 协议版本
  char* v = args.next();
  // 命令名称  
  char* name = args.next();
  if (name == NULL || strlen(name) > AttachOperation::name_length_max) {
    return NULL;
  }
  
  // 创建AttachOperation对象
  LinuxAttachOperation* op = new LinuxAttachOperation(name);
  
  // 从buf中读取AttachOperation参数  
  for (int i=0; i<AttachOperation::arg_count_max; i++) {
    char* arg = args.next();
    if (arg == NULL) {
      op->set_arg(i, NULL);
    } else {
      if (strlen(arg) > AttachOperation::arg_length_max) {
        delete op;
        return NULL;
      }
      op->set_arg(i, arg);
    }
  }
  
  // 将socket引用设置到op对象中
  op->set_socket(s);
  return op;
}
```

这是Linux上的实现，不同的操作系统实现方式不一样。Attach Listener线程监听.java_pid<pid>文件，等待Attach 客户端发起连接，解析Attach 客户端的Attach request 请求信息，将请求的字节流包装成一个AttachOperation类型的对象，之后就会从表里查询对应的处理函数，然后进行处理并返回处理结果。

 Attach 机制详细的交互流程可以用下面的图3-4描述。

> 图3-4 Attach交互处理流程

![图3-4 Attach交互处理流程](images/图3-4 Attach交互处理流程.png)


### 3.2.3 Attach机制涉及到的JVM参数

这里重新总结下Attach机制涉及到JVM参数。如下表3-1所示。

> 表3-1 Attach机制相关的JVM参数

| 名称 | 含义                       | 默认值   |
|----|--------------------------|-------|
| ReduceSignalUsage | 禁止信号量使用                  | false |
| DisableAttachMechanism | 禁止attach到当前JVM           | false |
| StartAttachListener | JVM 启动时初始化AttachListener | false |
| EnableDynamicAgentLoading | 允许运行时加载Agent             | true  |

JVM 参数都在`src/hotspot/share/runtime/globals.hpp` 中定义

## 3.3 Attach开源工具

### 3.3.1 使用golang实现Attach注入工具

上一节中，详细分析了Attach通信建立和发送数据全过程，本节将使用Golang语言构建实现一个轻量级的Attach工具，并使用Attach工具获取目标JVM的堆栈信息。代码来源于开源项目：https://github.com/tokuhirom/go-hsperfdata

#### 3.3.1.1 建立通信

> 代码位置：attach/attach_linux.go

```go
package attach

import (
	"fmt"
	"net"
	"os"
	"path/filepath"
	"syscall"
	"time"
)

// 执行attach
func force_attach(pid int) error {
  // 进程的工作目录下创建.attach_pid<pid>文件
	attach_file := fmt.Sprintf("/proc/%d/cwd/.attach_pid%d", pid, pid)
	f, err := os.Create(attach_file)
	if err != nil {
		return fmt.Errorf("Canot create file:%v:%v", attach_file, err)
	}
	f.Close()
	
  // 给目标JVM发送SIGQUIT信号
	err = syscall.Kill(pid, syscall.SIGQUIT)
	if err != nil {
		return fmt.Errorf("Canot send sigkill:%v:%v", pid, err)
	}
	
  // 检查.java_pid<pid>文件是否存在
	sockfile := filepath.Join(os.TempDir(), fmt.Sprintf(".java_pid%d", pid))
	for i := 0; i < 10; i++ {
		if exists(sockfile) {
			return nil
		}
		time.Sleep(200 * time.Millisecond)
	}
	return fmt.Errorf("Canot attach process:%v", pid)
}

// 建立与目标JVM的UDS通信
func GetSocketFile(pid int) (string, error) {
	sockfile := filepath.Join(os.TempDir(), fmt.Sprintf(".java_pid%d", pid))
	if !exists(sockfile) {
		err := force_attach(pid)
		if err != nil {
			return "", err
		}
	}
	return sockfile, nil
}

// 创建UDS
func New(pid int) (*Socket, error) {
	sockfile, err := GetSocketFile(pid)
	if err != nil {
		return nil, err
	}

	addr, err := net.ResolveUnixAddr("unix", sockfile)
	if err != nil {
		return nil, err
	}
	
	c, err := net.DialUnix("unix", nil, addr)
	if err != nil {
		return nil, err
	}
	return &Socket{c}, nil
}

func exists(name string) bool {
	if _, err := os.Stat(name); err != nil {
		if os.IsNotExist(err) {
			return false
		}
	}
	return true
}
```

上面的`force_attach`方法创建attach_pid 文件并向目标JVM发送kill -3信号；

#### 3.3.1.2 发送命令和参数

```go
package attach

import (
	"fmt"
	"io"
	"net"
	"strconv"
)

const PROTOCOL_VERSION = "1"
const ATTACH_ERROR_BADVERSION = 101

type Socket struct {
	sock *net.UnixConn
}

func (sock *Socket) Close() error {
	return sock.sock.Close()
}

// read
func (sock *Socket) Read(b []byte) (int, error) {
	return sock.sock.Read(b)
}

// read
func (sock *Socket) ReadString() (string, error) {
	retval := ""
	for {
		buf := make([]byte, 1024)
		read, err := sock.Read(buf)
		if err != nil {
			if err == io.EOF {
				break
			}
			return retval, err
		}
		retval += string(buf[0 : read-1])
	}
	return retval, nil
}

// jstack
func (sock *Socket) RemoteDataDump() (string, error) {
	err := sock.Execute("threaddump")
	if err != nil {
		return "", err
	}

	return sock.ReadString()
}

// jcmd
func (sock *Socket) Jcmd(args ...string) (string, error) {
	err := sock.Execute("jcmd", args...)
	if err != nil {
		return "", err
	}

	return sock.ReadString()
}

// write
func (sock *Socket) Execute(cmd string, args ...string) error {
	err := sock.writeString(PROTOCOL_VERSION)
	if err != nil {
		return err
	}
	err = sock.writeString(cmd)
	if err != nil {
		return err
	}
	for i := 0; i < 3; i++ {
		if len(args) > i {
			err = sock.writeString(args[i])
			if err != nil {
				return err
			}
		} else {
			err = sock.writeString("")
			if err != nil {
				return err
			}
		}
	}

	i, err := sock.readInt()
	if i != 0 {
		if i == ATTACH_ERROR_BADVERSION {
			return fmt.Errorf("Protocol mismatch with target VM")
		} else {
			return fmt.Errorf("Command failed in target VM")
		}
	}
	return err
}

func (sock *Socket) readInt() (int, error) {
	b := make([]byte, 1)
	buf := make([]byte, 0)
	for {
		_, err := sock.Read(b)
		if err != nil {
			return 0, err
		}
		if '0' <= b[0] && b[0] <= '9' {
			buf = append(buf, b[0])
			continue
		}

		if len(buf) == 0 {
			return 0, fmt.Errorf("cannot read int")
		} else {
			return strconv.Atoi(string(buf))
		}
	}
}

func (sock *Socket) writeString(s string) error {
	return sock.write([]byte(s))
}

func (sock *Socket) write(bytes []byte) error {
	{
		wrote, err := sock.sock.Write(bytes)
		if err != nil {
			return err
		}
		if wrote != len(bytes) {
			return fmt.Errorf("cannot write")
		}
	}
	{
		wrote, err := sock.sock.Write([]byte("\x00"))
		if err != nil {
			return err
		}
		if wrote != 1 {
			return fmt.Errorf("cannot write null byte")
		}
	}
	return nil
}
```
上面代码主要功能是`Execute`方法, 该方法向socket写入上指定的字符序列。

#### 3.3.1.3 获取目标JVM的堆栈信息
再来看下main方法，接受pid参数并dump目标jvm的堆栈信息
```go
package main

import (
	"attach/attach"
	"fmt"
	"log"
	"os"
	"strconv"
)

// threaddump
func main() {

	if len(os.Args) == 1 {
		fmt.Printf("Usage: jstack pid\n")
		os.Exit(1)
	}
	pid, err := strconv.Atoi(os.Args[1])
	if err != nil {
		log.Fatal("invalid pid: %v", err)
	}

	sock, err := attach.New(pid)
	if err != nil {
		log.Fatalf("cannot open unix socket: %s", err)
	}
	err = sock.Execute("threaddump")
	if err != nil {
		log.Fatalf("cannot write to unix socket: %s", err)
	}

	stack, err := sock.ReadString()
	fmt.Printf("%s\n", stack)

}

```

输出结果：
```text
$ ./main 75193
2023-07-29 01:58:32
Full thread dump Java HotSpot(TM) 64-Bit Server VM (11.0.2+9-LTS mixed mode):

Threads class SMR info:
_java_thread_list=0x00007fc8a5f83fe0, length=11, elements={
0x00007fc8a68e4800, 0x00007fc8a68e9800, 0x00007fc8a705f000, 0x00007fc8a7055000,
0x00007fc8a7062000, 0x00007fc8a68f3800, 0x00007fc8a6068800, 0x00007fc8a8043800,
0x00007fc8a68e6800, 0x00007fc8a9813800, 0x00007fc8a71ac000
}

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 cpu=12.90ms elapsed=236130.65s tid=0x00007fc8a705f000 nid=0x3c03 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #5 daemon prio=9 os_prio=31 cpu=1845.75ms elapsed=236130.65s tid=0x00007fc8a7055000 nid=0x3d03 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   No compile task


// 篇幅有限省略...
```

### 3.3.2 jattach

#### 3.3.2.1 简介
jattach是一个不依赖于jdk/jre的运行时注入工具，并且具备jmap、jstack、jcmd和jinfo等功能，
同时支持linux、windows、macos等操作系统。项目地址：https://github.com/jattach/jattach

支持的命令包括：

+ load：加载agent
+ properties： 获取系统参数
+ agentProperties： 获取agent系统参数
+ datadump： 堆和线程的状态信息
+ threaddump：dump堆栈
+ dumpheap：dump内存的使用状态
+ inspectheap：fullgc 之后 dump 堆内存
+ setflag：修改可变VM参数
+ printflag：输出JVM系统参数
+ jcmd： 执行jcmd命令

#### 3.3.2.2 源码解析

> 代码位置：src/posix/jattach.c

```c
int jattach(int pid, int argc, char** argv) {
    // 获取attach进程和目标JVM进程的用户权限
    uid_t my_uid = geteuid();
    gid_t my_gid = getegid();
    uid_t target_uid = my_uid;
    gid_t target_gid = my_gid;
    int nspid;
    if (get_process_info(pid, &target_uid, &target_gid, &nspid) < 0) {
        fprintf(stderr, "Process %d not found\n", pid);
        return 1;
    }

    // Container support: switch to the target namespaces.
    // Network and IPC namespaces are essential for OpenJ9 connection.
    enter_ns(pid, "net");
    enter_ns(pid, "ipc");
    int mnt_changed = enter_ns(pid, "mnt");

    // In HotSpot, dynamic attach is allowed only for the clients with the same euid/egid.
    // If we are running under root, switch to the required euid/egid automatically.
    // 这里做进程权限切换
    // 在HotSpot虚拟机上，动态attach需要发起attach的进程与目标进程具备相同的权限
    // 如果attach进程权限是root（特权进程），可以实现自动切换到目标进程权限
    if ((my_gid != target_gid && setegid(target_gid) != 0) ||
        (my_uid != target_uid && seteuid(target_uid) != 0)) {
        perror("Failed to change credentials to match the target process");
        return 1;
    }

    get_tmp_path(mnt_changed > 0 ? nspid : pid);

    // Make write() return EPIPE instead of abnormal process termination
    signal(SIGPIPE, SIG_IGN);

    if (is_openj9_process(nspid)) {
        return jattach_openj9(pid, nspid, argc, argv);
    } else {
        return jattach_hotspot(pid, nspid, argc, argv);
    }
}
```
需要注意的是，在发起attach之前，需要将attach进程的权限设置为与目标JVM权限一致。
jattach给我们编译了各种平台的可执行文件，对于构建跨平台运行时注入工具很有用。我们仅需要使用即可，无需关心里面的实现。

## 3.4.attach 的常见坑

#### 不同版本JDK在Attach成功后返回结果差异性

+ 现象

当使用JDK11去attach JDK8应用时，会抛异常com.sun.tools.attach.AgentLoadException: 0 ，
但实际上已经attach成功了。异常结果如下：

```text
Start arthas failed, exception stack trace: com.sun.tools.attach.AgentLoadException: 0
 at jdk.attach/sun.tools.attach.HotSpotVirtualMachine.loadAgentLibrary(HotSpotVirtualMachine.java:108)
 at jdk.attach/sun.tools.attach.HotSpotVirtualMachine.loadAgentLibrary(HotSpotVirtualMachine.java:119)
 at jdk.attach/sun.tools.attach.HotSpotVirtualMachine.loadAgent(HotSpotVirtualMachine.java:147)
```

+ 原因

在不同的JDK中HotSpotVirtualMachine#loadAgentLibrary方法的返回值不一样 ，在JDK8中返回0表示attach成功。

> 代码位置：src/share/classes/sun/tools/attach/HotSpotVirtualMachine.java

```java
private void loadAgentLibrary(String agentLibrary, boolean isAbsolute, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    InputStream in = execute("load",
                             agentLibrary,
                             isAbsolute ? "true" : "false",
                             options);
    try {
        // 返回0表示attach成功
        int result = readInt(in);
        if (result != 0) {
            throw new AgentInitializationException("Agent_OnAttach failed", result);
        }
    } finally {
        in.close();

    }
}
```

JDK11返回的是"return code: 0"表示attach成功。

```java
// 代码位置：src/jdk.attach/share/classes/sun/tools/attach/HotSpotVirtualMachine.java
private void loadAgentLibrary(String agentLibrary, boolean isAbsolute, String options) 
    throws AgentLoadException, AgentInitializationException, IOException 
{   
    // 返回结果
    String msgPrefix = "return code: "; 
    InputStream in = execute("load", 
                             agentLibrary, 
                             isAbsolute ? "true" : "false", 
                             options); 
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(in))) { 
        String result = reader.readLine(); 
        if (result == null) { 
            throw new AgentLoadException("Target VM did not respond"); 
        } else if (result.startsWith(msgPrefix)) { 
            int retCode = Integer.parseInt(result.substring(msgPrefix.length())); 
            // "return code: 0" 表示attach成功 
            if (retCode != 0) { 
                throw new AgentInitializationException("Agent_OnAttach failed", retCode); 
            } 
        } else { 
            throw new AgentLoadException(result); 
        } 
    } 
} 
```

+ 方案

发起Attach的进程需要兼容不同版本JDK返回结果。下面是arthas诊断工具对这个问题的兼容性处理方案：

> 代码位置：arthas/core/src/main/java/com/taobao/arthas/core/Arthas.java

```java
try {
    virtualMachine.loadAgent(arthasAgentPath,
            configure.getArthasCore() + ";" + configure.toString());
} catch (IOException e) {
    // 处理返回值为 "return code: 0"
    if (e.getMessage() != null && e.getMessage().contains("Non-numeric value found")) {
        AnsiLog.warn(e);
        AnsiLog.warn("It seems to use the lower version of JDK to attach the higher version of JDK.");
        AnsiLog.warn(
                "This error message can be ignored, the attach may have been successful, and it will still try to connect.");
    } else {
        throw e;
    }
} catch (com.sun.tools.attach.AgentLoadException ex) {
    // 处理返回值为 "0"   
    if ("0".equals(ex.getMessage())) {
        // https://stackoverflow.com/a/54454418
        AnsiLog.warn(ex);
        AnsiLog.warn("It seems to use the higher version of JDK to attach the lower version of JDK.");
        AnsiLog.warn(   
                "This error message can be ignored, the attach may have been successful, and it will still try to connect.");
    } else {
        throw ex;
    }
}
```
上面的代码可以看出，在Attach抛出异常后，对异常进行分类处理，当抛出IOException并且异常的message中有"Non-numeric value found"，表示该异常是由于低版本Attach API attach 到高版本JDK上；
当抛出的异常是AgentLoadException并且message的值为"0"时，表示该异常是由于高版本Attach API attach 到低版本JDK导致。


#### .java_pid<pid>文件被删除

+ 现象

当执行attach命令如jstack时，出现报错Unable to open socket file: target process not responding or HotSpot VM not loaded。错误如下所示：

```text
MacBook-Pro admin$ jstack 33000
33000: Unable to open socket file: target process not responding or HotSpot VM not loaded
The -F option can be used when the target process is not responding
```

并且/tmp目录下没有attach通讯的.java_pid<pid>文件。

```text
MacBook-Pro admin$ ls .java_pid3000
ls: .java_pid3000: No such file or directory
```

然而，重启Java进程之后又可以使用jstack等attach工具了

+ 原因

很不幸，这是一个JDK的bug，原因是JVM在首次被attach时会创建.java_pid<pid>用于socket通信，
文件/tmp目录下（不同操作系统tmp目录位置不同，Linux 系统为/tmp 目录），该目录不可以被参数修改。
在Attach listener初始化过程中，这个文件首次被创建后，JVM会标记Attach Listener为initialized状态，
如果文件被删除了，这个Java进程无法被Attach。

+ 方案

  对于JDK8来说，只能重启进程；社区的讨论以及官方修复；


官方修复的pr给Attach Listener增加了INITIALIZING、NOT_INITIALIZED、INITIALIZED多种状态，并且在INITIALIZED状态下通过AttachListener::check_socket_file进行自检，如果发现文件不存在，会清理之前的listener，并重新建立。

修复代码如下，在代码行号为17处，对.attach_pid<pid>文件进行检测。
```c++
// Attempt to transit state to AL_INITIALIZING.
AttachListenerState cur_state = AttachListener::transit_state(AL_INITIALIZING, AL_NOT_INITIALIZED);
if (cur_state == AL_INITIALIZING) {
 // Attach Listener has been started to initialize. Ignore this signal.
  continue;
} else if (cur_state == AL_NOT_INITIALIZED) {
  // Start to initialize.
  if (AttachListener::is_init_trigger()) {
     // Attach Listener has been initialized.
     // Accept subsequent request.
      continue;
  } else {
     // Attach Listener could not be started.
     // So we need to transit the state to AL_NOT_INITIALIZED.
     AttachListener::set_state(AL_NOT_INITIALIZED);
  }
} else if (AttachListener::check_socket_file()) {
  // .attach_pid<pid>文件进行检测
  // Attach Listener has been started, but unix domain socket file
  // does not exist. So restart Attach Listener.
  continue;
}
```
需要说明的是，该修复仅限JDK11以上版本。

#### attach进程的权限问题

+ 现象

如果在root用户下执行jstack，而目标JVM进程不是root权限启动，执行报错如下：

```text
Unable to open socket file: target process not responding or HotSpot VM not loaded
The -F option can be used when the target process is not responding
```

+ 原因

下面是在JDK8上LinuxAttachListener线程接受命令的过程。在代码26行处会严格校验发起attach进程的uid和gid是否与目标JVM 一致。

> 代码位置：jdk8/src/hotspot/os/linux/vm/attachListener_linux.cpp

```c++

LinuxAttachOperation* LinuxAttachListener::dequeue() {
  for (;;) {
    int s;

    // wait for client to connect
    struct sockaddr addr;
    socklen_t len = sizeof(addr);
    RESTARTABLE(::accept(listener(), &addr, &len), s);
    if (s == -1) {
      return NULL;      // log a warning?
    }

    // get the credentials of the peer and check the effective uid/guid
    // - check with jeff on this.
    struct ucred cred_info;
    socklen_t optlen = sizeof(cred_info);
    if (::getsockopt(s, SOL_SOCKET, SO_PEERCRED, (void*)&cred_info, &optlen) == -1) {
      ::close(s);
      continue;
    }
    uid_t euid = geteuid();
    gid_t egid = getegid();
  
    // 严格校验 uid、gid
    if (cred_info.uid != euid || cred_info.gid != egid) {
      ::close(s);
      continue;
    }

    // peer credential look okay so we read the request
    LinuxAttachOperation* op = read_request(s);
    if (op == NULL) {
      ::close(s);
      continue;
    } else {
      return op;
    }
  }
}
```

原则是上root权限不应该受到限制，因此JDK11对这个"不太合理"的限制做了解除，可以使用root权限attach任意用户启动的Java进程。

> 代码位置：jdk11/src/hotspot/os/linux/attachListener_linux.cpp

```c++
LinuxAttachOperation* LinuxAttachListener::dequeue() {
  for (;;) {
    int s;

    // wait for client to connect
    struct sockaddr addr;
    socklen_t len = sizeof(addr);
    RESTARTABLE(::accept(listener(), &addr, &len), s);
    if (s == -1) {
      return NULL;      // log a warning?
    }

    // get the credentials of the peer and check the effective uid/guid
    struct ucred cred_info;
    socklen_t optlen = sizeof(cred_info);
    if (::getsockopt(s, SOL_SOCKET, SO_PEERCRED, (void*)&cred_info, &optlen) == -1) {
      log_debug(attach)("Failed to get socket option SO_PEERCRED");
      ::close(s);
      continue;
    }
    // 允许root权限attach
    if (!os::Posix::matches_effective_uid_and_gid_or_root(cred_info.uid, cred_info.gid)) {
      log_debug(attach)("euid/egid check failed (%d/%d vs %d/%d)",
              cred_info.uid, cred_info.gid, geteuid(), getegid());
      ::close(s);
      continue;
    }

    // peer credential look okay so we read the request
    LinuxAttachOperation* op = read_request(s);
    if (op == NULL) {
      ::close(s);
      continue;
    } else {
      return op;
    }
  }
}
```

matches_effective_uid_and_gid_or_root 的实现如下： 

> 代码位置：jdk11/src/hotspot/os/linux/attachListener_linux.cpp

```c++
bool os::Posix::matches_effective_uid_and_gid_or_root(uid_t uid, gid_t gid) {
    return is_root(uid) || (geteuid() == uid && getegid() == gid);
}
```

+ 解决方案

切换到与用户相同权限执行然后再执行Attach。 在介绍jattach工具时已经对这部分代码做了详细分析，这里不在赘述。

#### com.sun.tools.attach.AttachNotSupportedException: no providers installed

+ 原因以及解决方案
是因为引用的tools.jar包有问题，应该这样引用tools.jar
```xml
<dependency>
	<groupId>com.sun</groupId>
	<artifactId>tools</artifactId>
	<version>1.5.0</version>
	<scope>system</scope>
	<systemPath>/path/to/your/jdk/lib/tools.jar</systemPath>
</dependency>
```

systemPath标签用来指定本地的tools.jar位置，可以把tools.jar的绝对路径配置成相对路径：
```xml
<dependency>
	<groupId>com.sun</groupId>
	<artifactId>tools</artifactId>
	<version>1.5.0</version>
	<scope>system</scope>
	<systemPath>${env.JAVA_HOME}/lib/tools.jar</systemPath>
</dependency>
```

