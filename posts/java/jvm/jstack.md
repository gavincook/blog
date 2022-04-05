### 1. 用处
用于打印指定进程的线程堆栈信息或远程调试服务的堆栈信息.

### 2. 用法
`jstack [ option ] pid`
**option**:
-F : 当`jstack [-l] pid`没有相应的时候强制打印栈信息
-l : 长列表，包括锁信息
-m : 打印Java和native框架的所有栈信息

对于线程转储文件的分析，可以使用IBM的[IBM Thread and Monitor Dump Analyzer]；同样，jvisualvm也可以分析线程转储文件。


