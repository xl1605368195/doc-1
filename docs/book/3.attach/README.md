# 第3章 Attach机制

Attach机制从JDK1.6开始引入，主要是给运行中的Java进程注入一个Java Agent。
Java Agent有着广泛的使用场景， 如Java性能诊断工具jstack、jmap 和Arthas等都使用了该技术。

本章将从Attach API的基本使用、实现原理、开源工具和常见的坑等几个方面介绍Attach技术。
