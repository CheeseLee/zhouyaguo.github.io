---
layout: post
title: "Hotspot Command Line Tools"
date: 2014-5-25
comments: true
categories:
- Java
---

Oracle Hotspot JDK自带很多实用工具，对分析生产问题非常有用，常用的有：（注： IBM J9没有这些命令）

  - jps
  - jstat
  - jinfo
  - jmap
  - jhat
  - jstack

## jps

- 平时见大多数同事查找某java进程pid时，都习惯性输入```ps -ef|grep $USER|grep java``` ，其实只需要简单的输入```jps``` 即可列出运行中的java进程

    ```
    ➜  ~  jps -l
    30707 sun.tools.jps.Jps
    30397 com.zyg.test2.Main
    4061 /home/zyg/jbdevstudio/studio//plugins/org.eclipse.equinox.launcher_1.3.0.v20130327-1440.jar

    ```

## jstat

- jstat可以查看类装载、内存使用、gc情况、JIT编译情况。在没有GUI的生产环境下很有用。

    ```
    ➜  ~  jstat -help
    Usage: jstat -help|-options
           jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
    ```

    ```
    ➜  ~  jstat -options
    -class
    -compiler
    -gc
    -gccapacity
    -gccause
    -gcnew
    -gcnewcapacity
    -gcold
    -gcoldcapacity
    -gcpermcapacity
    -gcutil
    -printcompilation

    ```

- 查看class的载入，卸载等情况

    ```
    ➜  ~  jstat -class 30397
    Loaded  Bytes  Unloaded  Bytes     Time
       317   629.9        0     0.0       0.06
    ```

- 查看gc情况，单位是 **KB**（E=Eden, S0=Survivor0, S1=Survivor1, O=Old, P=Permanent, YGC=YoungGC, FGC=FullGC, C=Current, U=Utilization, T=Time）

    ```
    ➜  ~  jstat -gc 1197
     S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
    20224.0 20736.0  0.0   10240.0 482816.0 268280.7 1048576.0   403041.5  148352.0 148332.1     89    9.233   0      0.000    9.233
    ```

- -gcutil与-gc类型，只不过形式是**使用内存占总内存的百分比**。

    ```
    ➜  ~  jstat -gcutil 1197
      S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
      0.00  49.38  78.74  38.44  99.99     89    9.233     0    0.000    9.233
    ```

## jinfo

- jinfo命令可以查看运行中的jvm进程的 **System properties**

    ```
    ➜  ~  jinfo  30397
    Attaching to process ID 30397, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 20.45-b01
    Java System Properties:

    java.runtime.name = Java(TM) SE Runtime Environment
    sun.boot.library.path = /home/zyg/Downloads/jdk1.6.0_45/jre/lib/amd64
    java.vm.version = 20.45-b01
    java.vm.vendor = Sun Microsystems Inc.
    java.vendor.url = http://java.sun.com/
    path.separator = :
    java.vm.name = Java HotSpot(TM) 64-Bit Server VM
    file.encoding.pkg = sun.io
    sun.java.launcher = SUN_STANDARD
    user.country = US
    sun.os.patch.level = unknown
    java.vm.specification.name = Java Virtual Machine Specification
    user.dir = /home/zyg/workspace/testJavaProject
    java.runtime.version = 1.6.0_45-b06
    java.awt.graphicsenv = sun.awt.X11GraphicsEnvironment
    java.endorsed.dirs = /home/zyg/Downloads/jdk1.6.0_45/jre/lib/endorsed
    os.arch = amd64
    java.io.tmpdir = /tmp
    line.separator =

    java.vm.specification.vendor = Sun Microsystems Inc.
    os.name = Linux
    sun.jnu.encoding = UTF-8
    java.library.path = /home/zyg/Downloads/jdk1.6.0_45/jre/lib/amd64/server:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/amd64:/home/zyg/Downloads/jdk1.6.0_45/jre/../lib/amd64:/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
    java.specification.name = Java Platform API Specification
    java.class.version = 50.0
    sun.management.compiler = HotSpot 64-Bit Tiered Compilers
    os.version = 3.11.10-7-desktop
    user.home = /home/zyg
    user.timezone =
    java.awt.printerjob = sun.print.PSPrinterJob
    file.encoding = UTF-8
    java.specification.version = 1.6
    java.class.path = /home/zyg/workspace/testJavaProject/target/classes
    user.name = zyg
    java.vm.specification.version = 1.0
    sun.java.command = com.zyg.test2.Main
    java.home = /home/zyg/Downloads/jdk1.6.0_45/jre
    sun.arch.data.model = 64
    user.language = en
    java.specification.vendor = Sun Microsystems Inc.
    java.vm.info = mixed mode
    java.version = 1.6.0_45
    java.ext.dirs = /home/zyg/Downloads/jdk1.6.0_45/jre/lib/ext:/usr/java/packages/lib/ext
    sun.boot.class.path = /home/zyg/Downloads/jdk1.6.0_45/jre/lib/resources.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/rt.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/sunrsasign.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/jsse.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/jce.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/charsets.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/lib/modules/jdk.boot.jar:/home/zyg/Downloads/jdk1.6.0_45/jre/classes
    java.vendor = Sun Microsystems Inc.
    file.separator = /
    java.vendor.url.bug = http://java.sun.com/cgi-bin/bugreport.cgi
    sun.io.unicode.encoding = UnicodeLittle
    sun.cpu.endian = little
    sun.cpu.isalist =

    VM Flags:

    -Dfile.encoding=UTF-8


    ```

## jmap

jmap命令主要用于内存方面的信息查看，比如：查看heap各区的使用情况、permanent区的使用情况、正在等待回收的对象的情况、抓heapdump

- 查看帮助

    ```
    ➜  ~  jmap
    Usage:
        jmap [option] <pid>
            (to connect to running process)
        jmap [option] <executable <core>
            (to connect to a core file)
        jmap [option] [server_id@]<remote server IP or hostname>
            (to connect to remote debug server)

    where <option> is one of:
        <none>               to print same info as Solaris pmap
        -heap                to print java heap summary
        -histo[:live]        to print histogram of java object heap; if the "live"
                             suboption is specified, only count live objects
        -permstat            to print permanent generation statistics
        -finalizerinfo       to print information on objects awaiting finalization
        -dump:<dump-options> to dump java heap in hprof binary format
                             dump-options:
                               live         dump only live objects; if not specified,
                                            all objects in the heap are dumped.
                               format=b     binary format
                               file=<file>  dump heap to <file>
                             Example: jmap -dump:live,format=b,file=heap.bin <pid>
        -F                   force. Use with -dump:<dump-options> <pid> or -histo
                             to force a heap dump or histogram when <pid> does not
                             respond. The "live" suboption is not supported
                             in this mode.
        -h | -help           to print this help message
        -J<flag>             to pass <flag> directly to the runtime system
    ```

- 查看当全heap内各区的使用情况

    ```
    ➜  ~  jmap -heap 30397
    Attaching to process ID 30397, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 20.45-b01

    using thread-local object allocation.
    Parallel GC with 2 thread(s)

    Heap Configuration:
       MinHeapFreeRatio = 40
       MaxHeapFreeRatio = 70
       MaxHeapSize      = 973078528 (928.0MB)
       NewSize          = 1310720 (1.25MB)
       MaxNewSize       = 17592186044415 MB
       OldSize          = 5439488 (5.1875MB)
       NewRatio         = 2
       SurvivorRatio    = 8
       PermSize         = 21757952 (20.75MB)
       MaxPermSize      = 85983232 (82.0MB)

    Heap Usage:
    PS Young Generation
    Eden Space:
       capacity = 15269888 (14.5625MB)
       used     = 916336 (0.8738861083984375MB)
       free     = 14353552 (13.688613891601562MB)
       6.000934649946352% used
    From Space:
       capacity = 2490368 (2.375MB)
       used     = 0 (0.0MB)
       free     = 2490368 (2.375MB)
       0.0% used
    To Space:
       capacity = 2490368 (2.375MB)
       used     = 0 (0.0MB)
       free     = 2490368 (2.375MB)
       0.0% used
    PS Old Generation
       capacity = 40501248 (38.625MB)
       used     = 0 (0.0MB)
       free     = 40501248 (38.625MB)
       0.0% used
    PS Perm Generation
       capacity = 21757952 (20.75MB)
       used     = 2635128 (2.5130538940429688MB)
       free     = 19122824 (18.23694610595703MB)
       12.111103103821536% used
    ```

- 查看permanent区情况（同时还可以查看到每个classloader的情况）

    ```
    ➜  ~  jmap -permstat 30397
    Attaching to process ID 30397, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 20.45-b01
    2948 intern Strings occupying 246880 bytes.
    finding class loader instances ..Finding object size using Printezis bits and skipping over...
    Finding object size using Printezis bits and skipping over...
    Finding object size using Printezis bits and skipping over...
    Finding object size using Printezis bits and skipping over...
    Finding object size using Printezis bits and skipping over...
    done.
    computing per loader stat ..done.
    please wait.. computing liveness..................done.
    class_loader	classes	bytes	parent_loader	alive?	type

    <bootstrap>	974	6008336	  null  	live	<internal>
    0x00000000ecad5898	0	0	  null  	live	sun/misc/Launcher$ExtClassLoader@0x00000000c0fd2878
    0x00000000ecef16c0	1	3104	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000c0e67830
    0x00000000ecede048	1	3104	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000c0e67830
    0x00000000ecadc480	11	126224	0x00000000ecad5898	live	sun/misc/Launcher$AppClassLoader@0x00000000c1042060
    0x00000000eceea818	1	3104	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000c0e67830

    total = 6	988	6143872	    N/A    	alive=3, dead=3	    N/A
    ```

- 抓heapdump

    ```
    ➜  ~  jmap -dump:file=/tmp/test 30397
    Dumping heap to /tmp/test.dat ...
    Heap dump file created
    ```

## jhat

jhat(JVM Heap Analysis Tool)，顾名思义，就是用来分析heapdump文件的。jhat内置了一个http server，分析完heapdump后，可以在浏览器中查看结果。

**注意：在生产环境中不要使用jhat工具，因为分析heapdump很耗cpu与内存，对生产环境会造成影响。**

通常jhat命令用的比较少，既然不在生产上，而是离线分析，还是使用Eclipse Memory Analyzer工具分析更方便。

```
➜  /tmp  jhat test.dat
Reading from test...
Dump file created Sun May 25 15:32:29 CST 2014
Snapshot read, resolving...
Resolving 40549 objects...
Chasing references, expect 8 dots........
Eliminating duplicate references........
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.

```

其中**Show heap histogram**最有用，因为可以显示占用内存最大的对象。
![](/public/images/jhat.png)


**Object Query Language (OQL)： OQL is SQL-like query language to query Java heap.**
![](/public/images/oql.png)


## jstack

- jstack用于抓取线程快照(thread dump)，主要目的就是看看当前线程都在做什么事情，以便定位僵死，响应慢，锁等问题。

    ```
    ➜  ~  jstack 30397
    2014-05-25 16:03:36
    Full thread dump Java HotSpot(TM) 64-Bit Server VM (20.45-b01 mixed mode):

    "RMI TCP Accept-0" daemon prio=10 tid=0x00007ff9340ac800 nid=0x17d runnable [0x00007ff93bdab000]
       java.lang.Thread.State: RUNNABLE
    	at java.net.PlainSocketImpl.socketAccept(Native Method)
    	at java.net.PlainSocketImpl.accept(PlainSocketImpl.java:408)
    	- locked <0x00000000ece49dd0> (a java.net.SocksSocketImpl)
    	at java.net.ServerSocket.implAccept(ServerSocket.java:462)
    	at java.net.ServerSocket.accept(ServerSocket.java:430)
    	at sun.management.jmxremote.LocalRMIServerSocketFactory$1.accept(LocalRMIServerSocketFactory.java:34)
    	at sun.rmi.transport.tcp.TCPTransport$AcceptLoop.executeAcceptLoop(TCPTransport.java:369)
    	at sun.rmi.transport.tcp.TCPTransport$AcceptLoop.run(TCPTransport.java:341)
    	at java.lang.Thread.run(Thread.java:662)

    "Attach Listener" daemon prio=10 tid=0x00007ff94c001000 nid=0x17c waiting on condition [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "DestroyJavaVM" prio=10 tid=0x00007ff96c005800 nid=0x76c2 waiting on condition [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "Thread-0" prio=10 tid=0x00007ff96c0a5000 nid=0x76cf waiting on condition [0x00007ff96831b000]
       java.lang.Thread.State: TIMED_WAITING (sleeping)
    	at java.lang.Thread.sleep(Native Method)
    	at com.zyg.test2.Main$1.run(Main.java:13)
    	at java.lang.Thread.run(Thread.java:662)

    "Low Memory Detector" daemon prio=10 tid=0x00007ff96c08c000 nid=0x76cc runnable [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "C2 CompilerThread1" daemon prio=10 tid=0x00007ff96c08a000 nid=0x76cb waiting on condition [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "C2 CompilerThread0" daemon prio=10 tid=0x00007ff96c087000 nid=0x76ca waiting on condition [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "Signal Dispatcher" daemon prio=10 tid=0x00007ff96c085000 nid=0x76c9 runnable [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE

    "Finalizer" daemon prio=10 tid=0x00007ff96c068800 nid=0x76c7 in Object.wait() [0x00007ff968960000]
       java.lang.Thread.State: WAITING (on object monitor)
    	at java.lang.Object.wait(Native Method)
    	- waiting on <0x00000000ecab1300> (a java.lang.ref.ReferenceQueue$Lock)
    	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:118)
    	- locked <0x00000000ecab1300> (a java.lang.ref.ReferenceQueue$Lock)
    	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:134)
    	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:171)

    "Reference Handler" daemon prio=10 tid=0x00007ff96c066800 nid=0x76c6 in Object.wait() [0x00007ff968a61000]
       java.lang.Thread.State: WAITING (on object monitor)
    	at java.lang.Object.wait(Native Method)
    	- waiting on <0x00000000ecab11d8> (a java.lang.ref.Reference$Lock)
    	at java.lang.Object.wait(Object.java:485)
    	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:116)
    	- locked <0x00000000ecab11d8> (a java.lang.ref.Reference$Lock)

    "VM Thread" prio=10 tid=0x00007ff96c05f800 nid=0x76c5 runnable

    "GC task thread#0 (ParallelGC)" prio=10 tid=0x00007ff96c018800 nid=0x76c3 runnable

    "GC task thread#1 (ParallelGC)" prio=10 tid=0x00007ff96c01a800 nid=0x76c4 runnable

    "VM Periodic Task Thread" prio=10 tid=0x00007ff96c096800 nid=0x76cd waiting on condition

    JNI global references: 919
    ```
