# 命令执行(RCE)

|类全限定名称|方法|参数|备注|
|:----:|:----:|:----:|:----|
|java.lang.UNIXProcess|forkAndExec|(int mode, byte[] helperpath,byte[] prog,byte[] argBlock, int argc,byte[] envBlock, int envc,byte[] dir,int[] fds,boolean redirectErrorStream)|jdk8以下|
|java.lang.ProcessImpl|forkAndExec|(int mode, byte[] helperpath,byte[] prog,byte[] argBlock, int argc,byte[] envBlock, int envc,byte[] dir,int[] fds,boolean redirectErrorStream)|jdk9以上|
|java.lang.ProcessImpl|create|(String cmdString, String envBlock,String dir,long[] stdHandles, ,boolean redirectErrorStream)|windows系统|

> linux & macos
```java
private native int forkAndExec(int mode, byte[] helperpath,
        byte[] prog,
        byte[] argBlock, int argc,
        byte[] envBlock, int envc,
        byte[] dir,
        int[] fds,
        boolean redirectErrorStream)
        throws IOException;
```

> windows
```java
private static synchronized native long create(String cmdstr,
                                  String envblock,
                                  String dir,
                                  long[] stdHandles,
                                  boolean redirectErrorStream)
    throws IOException;
```

jrasp在native层面hook了上面的方法，防止命令执行绕过。