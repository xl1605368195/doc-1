# 命令执行

|类全限定名称|方法|参数|备注|
|:----:|:----:|:----:|:----|
|java.lang.UNIXProcess|forkAndExec|(int mode, byte[] helperpath,byte[] prog,byte[] argBlock, int argc,byte[] envBlock, int envc,byte[] dir,int[] fds,boolean redirectErrorStream)|jdk8以下|
|java.lang.ProcessImpl|forkAndExec|(int mode, byte[] helperpath,byte[] prog,byte[] argBlock, int argc,byte[] envBlock, int envc,byte[] dir,int[] fds,boolean redirectErrorStream)|jdk9以上|
|java.lang.ProcessImpl|create|(String cmdString, String envBlock,String dir,long[] stdHandles, ,boolean redirectErrorStream)|windows系统|

> linux & macos
```java
/**
 * Creates a process. Depending on the {@code mode} flag, this is done by
 * one of the following mechanisms:
 * <pre>
 *   1 - fork(2) and exec(2)
 *   2 - posix_spawn(3P)
 *   3 - vfork(2) and exec(2)
 *
 *  (4 - clone(2) and exec(2) - obsolete and currently disabled in native code)
 * </pre>
 * @param fds an array of three file descriptors.
 *        Indexes 0, 1, and 2 correspond to standard input,
 *        standard output and standard error, respectively.  On
 *        input, a value of -1 means to create a pipe to connect
 *        child and parent processes.  On output, a value which
 *        is not -1 is the parent pipe fd corresponding to the
 *        pipe which has been created.  An element of this array
 *        is -1 on input if and only if it is <em>not</em> -1 on
 *        output.
 * @return the pid of the subprocess
 */
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
/**
 * Create a process using the win32 function CreateProcess.
 * The method is synchronized due to MS kb315939 problem.
 * All native handles should restore the inherit flag at the end of call.
 *
 * @param cmdstr the Windows command line
 * @param envblock NUL-separated, double-NUL-terminated list of
 *        environment strings in VAR=VALUE form
 * @param dir the working directory of the process, or null if
 *        inheriting the current directory from the parent process
 * @param stdHandles array of windows HANDLEs.  Indexes 0, 1, and
 *        2 correspond to standard input, standard output and
 *        standard error, respectively.  On input, a value of -1
 *        means to create a pipe to connect child and parent
 *        processes.  On output, a value which is not -1 is the
 *        parent pipe handle corresponding to the pipe which has
 *        been created.  An element of this array is -1 on input
 *        if and only if it is <em>not</em> -1 on output.
 * @param redirectErrorStream redirectErrorStream attribute
 * @return the native subprocess HANDLE returned by CreateProcess
 */
private static synchronized native long create(String cmdstr,
                                  String envblock,
                                  String dir,
                                  long[] stdHandles,
                                  boolean redirectErrorStream)
    throws IOException;
```

jrasp 在native层面hook了上面的方法，防止命令执行绕过。