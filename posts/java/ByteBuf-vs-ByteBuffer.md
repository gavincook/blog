## 1. ByteBuffer

### 1.1 ByteBuffer是什么？
ByteBuffer是Java系统提供的字节缓冲区。在NIO中，无论是读取数据还是写入数据都需要经过ByteBuffer缓冲区。

### 1.2 ByteBuffer工作机制
ByteBuffer维护了三个“指针”，分别为：

* position：当前写/读操作的下标，每次读写操作后，position会进行递增
* limit：读写操作都不能超过该下标。
* capacity：缓冲区的容量

三个“指针”的关系为：0 <= position <= limit <= capacity，然后ByteBuffer通过`flip()`方法来切换模式。当对ByteBuffer进行操作时，各个下标的变化如下图：

![bytebuffer][1]
其中蓝色代表可写入的区域，橙色代表已写入的数据区。
从图中可以看到：

* 初始化时，position=0; limit = capacity
* 写入数据时，position递增，但最大不会超过limit
* 当要从缓冲区读取数据时，需要先将模式从写模式切换到读模式。调用`flip()`方法进行模式的切换，此时limit=position; position=0; 也即在读取的时候只能从缓冲区最开始的地方读取到写模式的position处

**注意：** 不同操作方式（读/写）间一定要切换模式，并且flip方法主要用于写模式切换到读模式。如果在读模式调用flip，然后进行写入操作，此时会将写入的范围限制到读模式的position坐标。当ByteBuffer需要从读模式切换到写模式，可以调用`clear()`方法来重置缓冲区。

另外，我们在读写操作时，可以通过`mark()`方法将当前的position设置为一个临时点，后续可以通过调用`reset()`将position重置到mark的值。但是`flip()`方法或position值小于mark值时，会将mark值清空（即设置为-1）。如下图：
![bytebuffer-mark][2]

mark和前面说的三个“指针”的关系为：0 <= mark <= position <= limit <= capacity

### 1.3 堆外内存
ByteBuffer提供了两种分配方式，堆内内存申请和堆外内存申请。顾名思义，堆内内存申请，则直接从jvm堆中申请空间，并且会接受jvm gc管理。而堆外内存，则不会受gc直接管理（可以通过管理堆内对象，来释放关联的堆外内存）。

#### 1.3.1 为什么要使用堆外内存？
在某些特殊的应用中有大量生命周期很长的对象，在应用运行的整个过程中它们都存在，不需要被GC回收。如果这类对象很多，总体占用内存比例高，那么他们的存在将给GC带来很多不必要的工作负担。
另外，目前JVM间没有很高效的内存/对象共享技术，堆外内存为在JVM间共享内存/对象提供了必要的基础。
在进行通信/IO的时候，即使我们使用ByteBuffer的堆内内存的方式，仍然会将数据拷贝一份到堆外，然后再进行通信/IO。这里可以将堆外内存理解为jvm堆和内核之间的缓冲区。因此在一些需要高效率的场景，如缓存，一般会使用堆外内存。

#### 1.3.2 如何申请堆外内存
在ByteBuffer之前，可以通过反射使用`Unsafe`来获取堆外内存（因为默认Unsafe不允许外部调用）。
```
allocateMemory(long)//申请堆外内存
freeMemory(long)//释放堆外内存空间
```
在使用`Unsafe`进行堆外内存操作时，一定要记得调用`freeMemory`进行内存释放，否则会出现内存泄漏。

使用反射还是不够优雅，从java1.4开始，我们可以通过ByteBuffer来申请堆外内存。
```
ByteBuffer.allocateDirect(int)
```
ByteBuffer并没有提供对堆外内存的释放api，那么是否就没办法回收通过ByteBuffer申请的堆外内存呢？
ByteBuffer有一个`cleaner`字段，GC的时候会扫描DirectByteBuffer，然后调用cleaner的回收api，将堆外内存进行回收。也就是说，虽然我们通过ByteBuffer申请了堆外内存，但是无需去关心回收该内存，GC会通过管理堆内的ByteBuffer来及时回收通过ByteBuffer申请的堆外内存。

## 2. ByteBuf

### 2.1 ByteBuf是什么？
ByteBuf是Netty提供的类似JDK自带的ByteBuffer的字节数据容器。Netty中，所有的数据读写都需要经过ByteBuf进行缓冲。

### 2.2 ByteBuf工作机制
ByteBuf提供了读索引(readerIndex)，写索引（writterIndex）。读和写可以同时进行，不用像ByteBuffer一样使用flip进行模式的切换。所有的writexx都会增加写索引，readxx方法增加读索引。而getxx和setxx会读取或写入值，但不会改变读索引和写索引。索引值和容量的关系如下：
0 <= readerIndex <= writterIndex <= capacity
![bytebuf][3]
另外，ByteBuf也提供了形如ByteBuffer的mark功能，但ByteBuffer只能有一个mark设置点，而ByteBuf提供了两个mark设置点，分别为：markedReaderIndex 和 markedWritterIndex，分别针对读和写操作。mark值不一定对应的索引值小，有时会比对应索引值大。只要满足重置到mark值时，读索引小于写索引即可。如当markedReaderIndex大于写索引时，在调用`resetReaderIndex()`时，会抛出异常。
![bytebuf-mark][4]

### 2.3 使用模式
* 堆内存模式：即为创建一个byte数组
* 堆外内存模式：使用ByteBuffer的堆外内存申请api实现
* 复合缓冲模式
在ByteBuffer下，要实现复合模式，如下：
    ```
    ByteBuffer[] message = { header, body };
    ByteBuffer message2 = ByteBuffer.allocate(
            header.remaining() + body.remaining());
    message2.put(header);
    message2.put(body);
    message2.flip();
    ```
在ByteBuf下：

    ```
    CompositeByteBuf messageBuf = ...;
    ByteBuf headerBuf = ...; 
    ByteBuf bodyBuf = ...; 
    messageBuf.addComponents(headerBuf, bodyBuf);
   ```
在ByteBuffer下，会新申请一个内存空间，然后复制缓冲区的数据到新的空间。比ByteBuf下的直接引用，效率会更低。这也是Netty ByteBuf的zero-copy特性。

### 2.4 几个API
* discardReadBytes
将已经读取过的数据丢弃，并把未读取的数据移动到缓冲区开始处。
* forEachByte
用于查询要寻找的字节是否包含在缓冲区内，该方法接收一个ByteBufProcessor参数，ByteBufProcessor中的`process`方法返回false时，则会停止继续寻找，表示找到要寻找的字节。比如要寻找换行符，可以使用：
```
forEachByte（ByteBufProcessor.FIND_CRLF）
```
* order
可以用于指定缓冲区数据是大端模式还是小端模式。

### 2.5 为何要重新发明轮子
* zero-copy
* 无需调用flip来切换模式
* ByteBuf池化
* 清晰的使用api
* 动态的ByteBuf容量
### 2.6 各种各样的ByteBuf
![bytebuf class diagram][5]


  [1]: http://7xqdsb.com1.z0.glb.clouddn.com/bytebuffer.png
  [2]: http://7xqdsb.com1.z0.glb.clouddn.com/bytebuffer-mark.png
  [3]: http://7xqdsb.com1.z0.glb.clouddn.com/ByteBuf.png
  [4]: http://7xqdsb.com1.z0.glb.clouddn.com/bytebuf-mark.png
  [5]: http://7xqdsb.com1.z0.glb.clouddn.com/bytebuf-class-diagram.png

