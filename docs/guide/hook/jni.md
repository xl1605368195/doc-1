# JNI

|类全限定名称|方法|参数|备注|
|:----:|:----:|:----:|:----|
|java.lang.System|load|(String filename)| |
|java.lang.System|loadLibrary|(String libname)| |

```java
public static void load(String filename) {
    Runtime.getRuntime().load0(Reflection.getCallerClass(), filename);
}
```

```java
public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
}
```
