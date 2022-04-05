### 1. 用处
用于查看基于hotspot的jvm里面，所有具有访问权限的Java进程具体状态，包括进程ID、启动路径、启动参数等。

### 2. 用法
`jps [ options ] [ hostid ]`

**options**:

* -q : 只输出pid
* -m : 输出传递给main方法的参数
* -l : 输出程序主类的完整路径
* -v : 输出传递给jvm的参数
* -V : 输出通过标记的文件传递给JVM的参数（.hotspotrc文件，或者是通过参数-XX:Flags=<filename>指定的文件）
* -J : 用于传递jvm选项到由javac调用的java加载器中，例如，“-J-Xms48m”将把启动内存设置为48M，使用-J选项可以非常方便的向基于Java的开发的底层虚拟机应用程序传递参数。

