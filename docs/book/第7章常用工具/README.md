# 性能诊断工具

工欲善其事、必先利其器，在RASP研发过程中不可避免的会遇到各种问题，如CPU飙高、线程死锁、内存泄露，甚至JDK的bug等，如何快速、准确的的定位到故障、性能瓶颈是非常关键的，
熟练使用各种工具是排查问题的基础。本章主要介绍各种性能诊断工具的原理以及使用，并通过几个实际例子来说明如何使用。

本章将先介绍jdk自带的工具如jps、jstack和jmap等，还将介绍图形化工具 jconsole、vm，开源免费的工具arthas和btrace，商业化工具jprofile等。


参考文档：https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/index.html

https://docs.oracle.com/en/java/javase/20/docs/specs/jvmti.html


