# ASM VS Javassist

## javassist

• Javassist (Java Programming Assistant) makes Java bytecode manipulation simple. It is a class library for editing bytecodes in Java; it enables Java programs to define a new class at runtime and to modify a class file when the JVM loads it. Unlike other similar bytecode editors

• Javassist provides two levels of API: source level and bytecode level. If the users use the source-level API, they can edit a class file without knowledge of the specifications of the Java bytecode. The whole API is designed with only the vocabulary of the Java language. You can even specify inserted bytecode in the form of source text; Javassist compiles it on the fly. On the other hand, the bytecode-level API allows the users to directly edit a class file as other editors.

• Javassist lets you inspect, edit, and create Java binary classes.

• Javassist isn’t the only library for working with bytecode, but it does have one feature in particular that makes it a great starting point for experimenting with classworking: you can use Javassist to alter the bytecode of a Java class without actually needing to learn anything about bytecode or the Java virtual machine (JVM) architecture.

• Aspect Oriented Programming: Javassist can be a good tool for adding new methods into a class and for inserting before/after/around advice at the both caller and callee sides.

• Reflection: One of applications of Javassist is runtime reflection; Javassist enables Java programs to use a metaobject that controls method calls on base-level objects. No specialized compiler or virtual machine are needed.

• Javassist also provides lower-level API for directly editing a class file. To use this level of API, you need detailed knowledge of the Java bytecode and the class file format while this level of API allows you any kind of modification of class files.

## ASM

• ASM is an all purpose Java bytecode manipulation and analysis framework. It can be used to modify existing classes or dynamically generate classes, directly in binary form. Provided common transformations and analysis algorithms allow to easily assembling custom complex transformations and code analysis tools.

• ASM offer similar functionality as other bytecode frameworks, but it is focused on simplicity of use and performance. Because it was designed and implemented to be as small and as fast as possible, it makes it very attractive for using in dynamic systems*.

• ASM is a Java class manipulation tool designed to dynamically generate and manipulate Java classes, which are useful techniques to implement adaptable systems. ASM is based on a new approach, compared to equivalent existing tools, which consists in using the “visitor” design pattern without explicitly representing the visited tree with objects. This new approach gives much better performances than those of existing tools, for most of practical needs.

## Javassist & ASM

• Javassist source level API is much easier to use than the actual bytecode manipulation in ASM

• Javassist provides a higher level abstraction layer over complex bytecode level operations. Javassist source level API requires very less or no knowledge of actual bytecodes, so much easier & faster to implement.

• Javassist uses reflection mechanism which makes it slower compared to ASM which uses Classworking techniques at runtime.

• Overall ASM is much faster & gives better performance than Javassist. Javassist uses a simplified version of Java source code, which it then compiles into bytecode. That makes Javassist very easy to use, but it also limits the use of bytecode to what can be expressed within the limits of the Javassist source code.

• In conclusion, if someone needs easier way of dynamically manipulating or creating Java classes Javassist API should be used & where the performance is the key issue ASM library should be used.

翻译

+ Javassist源代码级API比ASM中实际的字节码操作更容易使用

+ Javassist在复杂的字节码级操作上提供了更高级别的抽象层。Javassist源代码级API只需要很少的字节码知识，甚至不需要任何实际字节码知识，因此实现起来更容易、更快。

+ Javassist使用反射机制，这使得它比运行时使用Classworking技术的ASM慢。

+ 总的来说ASM比Javassist快得多，并且提供了更好的性能。Javassist使用Java源代码的简化版本，然后将其编译成字节码。这使得Javassist非常容易使用，但是它也将字节码的使用限制在Javassist源代码的限制之内。


Table 1. Class construction times

|Framework|First time（ms）|Later times（ms）|
|:----:|:----:|:----:|
|Javassist|257|5.2|
|BCEL|473|5.5|
|ASM|62.4|1.1|

The Table 1 results show that ASM does live up to its billing as faster than the other frameworks, and this advantage applies both to startup time and to repeated uses.

## 结论
总之，如果有人需要更简单的方法来动态操作或创建Java类，那么应该使用Javassist API 。**如果需要注重性能地方，应该使用ASM库**。
