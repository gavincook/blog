<style type="text/css">table{margin-left:45px;}</style>
### 1. 用处
主要用于监控基于hotspot的jvm，包括：
* 类的加载和卸载情况
* 新生代、老年代、持久代（或元空间）的容量和使用情况
* 监控GC的情况，包括GC的次数和耗时

### 2. 用法
`jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]`

*注意：*笔者测试使用的Java版本为`jdk1.8.0_121`
#### 2.1. option:
可以使用`jstat -options`进行查看

* -class : 用于查看类的加载和卸载情况，具体如下：
    
    ```
    Loaded  Bytes  Unloaded  Bytes     Time
      3645  7788.1        0  0.0       1.22
    ```
    
| 字段 | 说明1 |
| --- | --- |
| loaded | 加载的类个数 |
| bytes | 加载的类的字节数 |
| unloaded| 卸载的类个数 |
| bytes | 卸载的类字节数 |
| Time | 加载和卸载类的耗时 |

* -compiler : 查看jvm及时编译器的执行情况

    ```
    Compiled Failed Invalid   Time   FailedType FailedMethod
    6519      0       0       29.05          0
    ```
    
|字段|说明|
|---|---|
|Compiled| 编译任务执行的次数|
|Failed| 编译任务失败的次数|
|Invalid| 编译任务非法执行次数|
|Time| 编译任务执行耗时|
|FailedType| 最后一次编译失败的任务类型|
|FailedMethod|最后一次编译失败的类名和方法名|
    
* -gc : 查看GC情况
    ```
    S0C     S1C      S0U    S1U      EC       EU        OC         OU       MC  
    14336.0 10752.0  0.0   6052.1 64000.0  62108.3   79360.0     8859.6   25472.0
    MU      CCSC   CCSU       YGC    YGCT  FGC     FGCT     GCT
    24800.3 3200.0 3019.2      5    0.047   1      0.031    0.079   
    ```
    
|字段|说明|备注|
|---|---|---|
|S0C| survivor区域中的s0 区域的容量（kb）||
|S1C| survivorq区域中的s1 区域的容量（kb）||
|S0U| survivor区域中的s0 区域的使用量（kb）||
|S1U| survivor区域中的s1 区域的使用量（kb）||
|EC| eden的容量（kb）||
|EU| eden的使用量（kb）||
|OC| 老年代的容量（kb）||
|OU| 老年代的使用量（kb）||
|MC|元空间的容量（kb）|jdk8之前为PC，持久代容量|
|MU|元空间的s使用量（kb）|jdk8之前为PU，持久代使用量|
|CCSC|类压缩空间容量（kb）|jdk8之前无该项|
|CCSU|类压缩空间使用量（kb）|jdk8之前无该项|
|YGC|年轻代GC次数||
|YGCT|年轻代GC耗时||
|FGC|full gc次数||
|FGCT|full gc耗时||
|GCT|gc总耗时|<br/>|
    
* -gccapacity : 查看堆内各代容量情况
    ```
     NGCMN     NGCMX     NGC      S0C     S1C       EC         OGCMN     OGCMX
     87040.0   262144.0  187392.0 22016.0 14336.0   143360.0   175104.0  524288.0
     OGC         OC          MCMN     MCMX       MC       CCSMN    CCSMX       CCSC      YGC    FGC
     103936.0   103936.0     0.0      1073152.0  27264.0  0.0      1048576.0   3200.0     19     3
    ```
    
|字段|说明|备注|
|---|---|---|
|NGCMN |新生代最小容量||
|NGCMX| 新生代最大容量||
|NGC| 当前新生代容量||
|S0C| survior 0的容量||
|S1C| survior 1的容量||
|EC| eden容量||
|OGCMN| 老年代最小容量||
|OGCMX| 老年代最大容量||
|OGC| 当前老年代容量|OGC=sum(OC)，一个OGC可能包含多个OC，就像新生代包含S0,S1,Eden一样。但在hotspot中只有一个，因此在hotspot中:OGC=OC|
|OC| 当前老年代空间容量||
|MCMN| 元空间最小容量|jdk8之前为PCMN，持久代最小容量|
|MCMX| 元空间最大容量|jdk8之前为PCMX，持久代最大容量|    
|MC| 当前元空间容量|jdk8之前为PC，当前持久代容量|
|CCSMN|类压缩空间最小容量（kb）|jdk8之前无该项|
|CCSMX|类压缩空间最大容量（kb）|jdk8之前无该项|
|CCSC|当前类压缩空间容量（kb）|jdk8之前无该项|
|YGC|年轻代GC次数||
|FGC|full gc次数|<br/>|

* -gcutil：查看各代GC的统计数据
    ```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  97.77  51.63  45.49  96.62  91.85  19      0.416    3     0.196    0.612
    ```
    
|字段|说明|备注|
|---|---|---|
|S0 |survior 0使用空间的百分比||
|S1 |survior 1使用空间的百分比||
|E |eden使用空间的百分比||
|O |老年代使用空间的百分比||
|M |元空间使用空间的百分比|jdk8之前为P：持久代使用的空间百分比|    
|CCS |类压缩空间使用空间的百分比|jdk8无该项|
|YGC | 新生代GC次数||
|YGCT|新生代GC耗时||
|FGC|full gc次数||
|FGCT|full gc耗时||
|GCT|GC耗时|<br/>|

* -gccause：和gcutil一样，查看GC统计数据，此外还会显示最后一次GC的原因，如果当前正在发生GC，还会展示当前GC发生的原因。
    ```
    LGCC                 GCC
    Allocation Failure   No GC
    ```
    
|字段|说明|
|---|---|
|LGCC |最后一次GC的原因|
|LGCC |当前GC的原因|

* -gcnew： 查看新生代垃圾回收情况
    ```
 S0C     S1C      S0U    S1U    TT MTT  DSS      EC        EU      YGC    YGCT
2112.0 2112.0    0.0    846.0  4   4   1056.0  17024.0  13279.5    129    0.354
    ```
    
|字段|说明|
|---|---|
|S0C |survior 0容量（kb）|
|S1C |survior 1容量（kb）|
|S0U |survior 0使用量（kb）|
|S1U |survior 1使用量（kb）|
|TT |Tenuring threshold，用于表示新生代对象经过多少次回收后，进入老年代|
|MTT|TT的最大值|
|DSS|Desired survivor size，具体参见表格后描述|
|EC|eden区容量|
|EU|eden区使用量|
|YGC|新生代GC次数|
|YGCT|新生代GC耗时|

    **Desired survivor size**：
    计算公式： (survivor_capacity * TargetSurvivorRatio) / 100：survivor_capacity（一个survivor space的大小）乘以TargetSurvivorRatio，
    表明所有age的survivor space对象的大小如果超过Desired survivor size，则重新计算threshold，以age和MaxTenuringThreshold的最小值为准，否则以MaxTenuringThreshold为准。threshold指对象经过多少次GC进入老年代。

* -gcnewcapacity：新生代空间容量统计
    ```
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
   87040.0   238592.0    87040.0  79360.0  10752.0  79360.0  10752.0   237568.0    65536.0     1     0
    ```
    
|字段|说明|备注|
|---|---|---|
|NGCMN |新生代最小容量||
|NGCMX| 新生代最大容量||
|NGC| 当前新生代容量||
|S0CMX| survior 0的最大容量||
|S0C| survior 0的容量||
|S1CMX| survior 1的最大容量||
|S1C| survior 1的容量||
|ECMX| eden最大容量||
|EC| eden容量||
|YGC|年轻代GC次数||
|FGC|full gc次数|<br/>|

* -gcold：查看老年代垃圾回收情况
    ```
MC       MU        CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
 15232.0  14766.1   1920.0   1758.8    175104.0     80.0      1     0      0.000    0.007
    ```
    
|字段|说明|备注|
|---|---|---|
|MC| 当前元空间容量|jdk8之前为PC，当前持久代容量|
|MU|元空间的s使用量（kb）|jdk8之前为PU，持久代使用量|
|CCSC|类压缩空间容量（kb）|jdk8之前无该项|
|CCSU|类压缩空间使用量（kb）|jdk8之前无该项|
|OC| 老年代的容量（kb）||
|OU| 老年代的使用量（kb）||
|YGC|年轻代GC次数||
|FGC|full gc次数||
|FGCT|full GC耗时||
|GCT|GC总耗时|<br/>|

* -gcoldcapacity：老年代空间容量统计
    ```
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
   175104.0    478208.0    175104.0    175104.0     1     0    0.000    0.007
    ```
    
|字段|说明|备注|
|---|---|---|
|OGCMN| 老年代最小容量||
|OGCMX| 老年代最大容量||
|OGC| 当前老年代容量|OGC=sum(OC)，一个OGC可能包含多个OC，就像新生代包含S0,S1,Eden一样。但在hotspot中只有一个，因此在hotspot中:OGC=OC|
|OC| 当前老年代空间容量||
|YGC|年轻代GC次数||
|FGC|full gc次数||
|FGCT|full GC耗时||
|GCT|GC总耗时|<br/>|

* -gcmetacapacity：查看元空间容量统计（jdk8之前为：-gcpermcapacity查看持久代容量）
    ```
   MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT
     0.0  1062912.0    15232.0        0.0  1048576.0     1920.0     1     0    0.000    0.007
    ```
    
|字段|说明|备注|
|---|---|---|
|MCMN| 元空间最小容量|jdk8之前为PCMN，持久代最小容量|
|MCMX| 元空间最大容量|jdk8之前为PCMX，持久代最大容量|    
|MC| 当前元空间容量|jdk8之前为PC，当前持久代容量|
|CCSMN|类压缩空间最小容量（kb）|jdk8之前无该项|
|CCSMX|类压缩空间最大容量（kb）|jdk8之前无该项|
|CCSC|当前类压缩空间容量（kb）|jdk8之前无该项|
|YGC|年轻代GC次数||
|FGC|full gc次数||
|FGCT|full GC耗时||
|GCT|GC总耗时|<br/>|

* -printcompilation： HotSpot编译方法统计
    ```
Compiled  Size  Type Method
    1107     49        1 io/netty/util/internal/shaded/org/jctools/queues/atomic/BaseLinkedAtomicQueue isEmpty
    ```
     
|字段|说明|
|---|---|
|Compiled| 编译任务执行的次数|
|Size| 方法的字节码所占的字节数|
|Type| 编译类型|
|Method| 确定被编译方法的类名及方法名，类名中使名“/”而不是“.”做为命名分隔符，方法名是被指定的类中的方法，这两个字段的格式是由HotSpot中的“-XX:+PrintComplation”选项确定的。|

