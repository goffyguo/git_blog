[TOC]

## JVM监控及诊断工具

### 命令行

#### 概述



#### 命令

##### jps：查看正在运行的Java进程

##### jstat：查看JVM统计信息

查看一个Java进程的内存和GC情况

jstat -gccapacity PID：堆内存分析

jstat -gcnew PID：年轻代GC分析，这里的TT和MTT可以看到在年轻代存活的对象的年龄和最大年龄

jstat -gcnewcapacity PID：年轻代内存分析

jstat -gcold PID：老年代GC分析

jstat -gcoldcapacity PID：老年代内存分析

jstat -gcmetacapacity PID：元数据内存分析

分析线上JVM进程时想知道的信息

- 新生代对象增长的速率
- Young GC触发频率
- Young GC的耗时
- 每次Young GC后有多少对象存活
- 每次Young GC有多少对象进入老年代
- 老年代对象增长速率
- Full GC的触发频率
- Full GC的耗时

jstat -gc PID 1000 10：根据自己的系统灵活多变的使用，如果系统负载很低，就把时间调高去观察对象增长的速率。另外一般系统又高峰和日常两种状态，可以分时段去查看对象增长速率。

##### jinfo：实时查看和修改JVM配置参数

##### jmap：导出内存映像文件&内存使用情况

基本语法：

导出内存映像文件：

- 手动
  - jmap -dump.format=b,file=< filename.hprof > < pid >
  - jmap -dump:live.format=b,file=< filename.hprof > < pid >
- 自动
  - -XX:+HeapDumpOnOutOfMemoryError
  - -XX:HeapDumpPath=< filename.hprof >

显示堆内存相关信息

- jmap -heap pid：这个命令会打印出来内存的相关一些参数的设置，然后就是当前堆内存里面一些基本各个区域的情况，比如Eden区总容量，已经使用的容量，剩余的空间容量，两个Survivor区的总容量，已经使用的容量和剩余的空间容量，老年代的总容量，已经使用的容量和剩余容量。
- jmap -histo pid：按照各种对象占用内存空间的大小降序排序，把占用内存最多的对象放在最上面。

jhat：JDK自带堆分析工具

jstack：打印JVM中线程快照

jcmd：多功能命令行

jstatd：远程主机信息收集

### GUI

#### 工具概览

#### 工具

jConsole

Visual VM

Eclipse  MAT

JProfiler 

Arthas

[Arthas 用户文档 — Arthas 3.5.4 文档 (aliyun.com)](https://arthas.aliyun.com/doc/)

[alibaba/arthas: Alibaba Java Diagnostic Tool Arthas/Alibaba Java诊断利器Arthas (github.com)](https://github.com/alibaba/arthas)

Java Mission Control

其他工具

## JVM 运行时参数

### JVM 参数选项类型

#### 标准参数选项

#### -X 参数选项

#### -XX 参数选项

### 添加 JVM 参数选项

#### IDEA

#### 运行 JAR 包

#### 通过 Tomcat 运行 wa 包

#### 程序运行过程中

### 常用的 JVM 参数选项

#### 打印设置的 XX 选项及值

#### 堆、栈、方法区等内存大小设置

#### OutOfMemory 相关的选项

#### 垃圾收集器相关选项

#### GC 日志相关选项

#### 其他参数

### 按功能性能区分 JVM 参数选项

### 通过 Java 代码获取 JVM 参数

