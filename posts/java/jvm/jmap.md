### 1. 用处
查看堆内存情况，包括对象数量、占用空间等。

### 2. 用法
```bash
jmap [option] pid
```
* `-dump:[live,]format=b,file=<filename>`: 使用hprof二进制形式,输出jvm的heap内容到文件=. live子选项是可选的，假如指定live选项,那么只输出活的对象到文件.
* `-finalizerinfo`:打印等待回收的对象信息
* `-heap`: 打印堆的垃圾回收算法，已经各个代的容量、使用情况

    ```
Attaching to process ID 31748, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13
using thread-local object allocation.
Mark Sweep Compact GC
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 20971520 (20.0MB)
   NewSize                  = 10485760 (10.0MB)
   MaxNewSize               = 10485760 (10.0MB)
   OldSize                  = 10485760 (10.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 1
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)
Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 7012352 (6.6875MB)
   used     = 4194352 (4.0000457763671875MB)
   free     = 2818000 (2.6874542236328125MB)
   59.81376861857477% used
Eden Space:
   capacity = 3538944 (3.375MB)
   used     = 2097168 (2.0000152587890625MB)
   free     = 1441776 (1.3749847412109375MB)
   59.25971137152778% used
From Space:
   capacity = 3473408 (3.3125MB)
   used     = 2097184 (2.000030517578125MB)
   free     = 1376224 (1.312469482421875MB)
   60.3782797759434% used
To Space:
   capacity = 3473408 (3.3125MB)
   used     = 0 (0.0MB)
   free     = 3473408 (3.3125MB)
   0.0% used
tenured generation:
   capacity = 10485760 (10.0MB)
   used     = 9292184 (8.861717224121094MB)
   free     = 1193576 (1.1382827758789062MB)
   88.61717224121094% used
743 interned Strings occupying 49944 bytes.   
    ```

* `-histo[:live]`: 打印每个class的实例数目,内存占用,类全名信息. VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量. 

    ```
    num     #instances         #bytes  class name
    ----------------------------------------------
   1:            44       13132984  [B
   2:          1341         138568  [C
   3:           111          76304  [I
   4:           490          55856  java.lang.Class
   5:          1329          31896  java.lang.String
   6:           555          28848  [Ljava.lang.Object;
   7:           104           6656  java.net.URL
   8:            79           5688  java.lang.reflect.Field
   9:           170           5440  java.util.HashMap$Node
10:           184           4416  java.util.LinkedList$Node
  11:            11           4136  java.lang.Thread
  12:           256           4096  java.lang.Integer
  13:            94           3760  java.lang.ref.SoftReference
  14:           114           3648  java.util.Hashtable$Entry
    ```
* `-clstats`: 打印classloader统计信息，包括：加载的类数目、字节数、父classloader、是否存活以及类型。注意：jdk8之前无该选项，对应的选项为：`-permstat`（类信息存放在持久代，jdk8存放在元空间）。
    
    ```
Attaching to process ID 32297, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.........done.
class_loader	classes	bytes	  parent_loader	  alive?  type
<bootstrap>	       423	813217	  null  	       live	  <internal>
0x00000007bf600000	12	54554	0x00000007bf612138 live	  sun/misc/Launcher$AppClassLoader@0x00000007c000f6a0
0x00000007bf612138   0	0	  null             live	  sun/misc/Launcher$ExtClassLoader@0x00000007c000fa48
total = 3	       435	867771	    N/A    	alive=3, dead=0	    N/A
    ```

* `-F`: 强制输出，在`jmap -dump`或 `jmap -histo`无响应时可使用，另外在强制模式下，`live`子选项无效。

