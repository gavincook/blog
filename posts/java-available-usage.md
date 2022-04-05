### 1. 背景
最近在工作中需要实现一个文件下载器，在下载文件的时候，笔者开始使用了如下代码片段：

```java
byte[] data = new byte[10240];
try (BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(file, true))) {
    while (in.available() > 0) {
        int length = in.read(data);
        out.write(data, 0, length);
    }
    out.flush();
}
```
逻辑很简单有没有，就是不断检查是否有可读的内容，然后将内容写到输出流中。
尴尬的是，一运行，老是重复下载（由于有重试机制）。映射到这里也就是输入流里的数据并没有写入或者并没有完全写入到输出流中。究其原因，是`available`方法使用不当。

Java的InputStream类提供`available`方法，这个方法初看字面意思可能会理解为整个输入流数据的长度，但实际上并不是这样的用法。

### 2. 定义与用法
先来看下文档说明：

```java
    /**
     * Returns an estimate of the number of bytes that can be read (or
     * skipped over) from this input stream without blocking by the next
     * invocation of a method for this input stream. The next invocation
     * might be the same thread or another thread.  A single read or skip of this
     * many bytes will not block, but may read or skip fewer bytes.
     *
     * <p> Note that while some implementations of {@code InputStream} will return
     * the total number of bytes in the stream, many will not.  It is
     * never correct to use the return value of this method to allocate
     * a buffer intended to hold all data in this stream.
     *
     * <p> A subclass' implementation of this method may choose to throw an
     * {@link IOException} if this input stream has been closed by
     * invoking the {@link #close()} method.
     *
     * <p> The {@code available} method for class {@code InputStream} always
     * returns {@code 0}.
     *
     * <p> This method should be overridden by subclasses.
     *
     * @return     an estimate of the number of bytes that can be read (or skipped
     *             over) from this input stream without blocking or {@code 0} when
     *             it reaches the end of the input stream.
     * @exception  IOException if an I/O error occurs.
     */
    public int available() throws IOException {
        return 0;
    }
```
`avaliable`用于返回非阻塞情况下，一次性可读的字节数。其中提及到不要使用该方法作为申请用于存储整个流数据的缓冲区容量。简而言之，available方法不一定会返回整个流的数据长度，并且绝大部分时候返回的长度都比实际整个流的长度小。

到这，就明白了为什么用`available`会导致内容少写或者不写到指定的地方了。对于文件下载的场景而言，数据来源于网络，这SocketInputStream是阻塞的，`available`的数字每次取决于网络状况。

### 3. FileInputStream
这里有一种较特殊的输入流，与其他阻塞的文件流不同，对于本地文件而言，其输入流的`available`可以得到整个文件的大小。`available`是非阻塞的，但文件流又是阻塞的，那么是怎么得到整个文件的大小的呢？
FileInputStream#available为一个本地方法，其内容为：

```
JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_available(JNIEnv *env, jobject this) {
    jlong ret;
    FD fd = GET_FD(this, fis_fd);
    if (fd == -1) {
        JNU_ThrowIOException (env, "Stream Closed");
        return 0;
    }
    if (IO_Available(fd, &ret)) {
        if (ret > INT_MAX) {
            ret = (jlong) INT_MAX;
        } else if (ret < 0) {
            ret = 0;
        }
        return jlong_to_jint(ret);
    }
    JNU_ThrowIOExceptionWithLastError(env, NULL);
    return 0;
}
```
可以看到最终通过`IO_Available`去获取可读的数据长度。而`IO_Available`在`io_util_md.h`中被定义为：`#define IO_Available handleAvailable`，继续看下`handleAvailable`的定义：

```
int
handleAvailable(FD fd, jlong *pbytes) {
    HANDLE h = (HANDLE)fd;
    DWORD type = 0;

    type = GetFileType(h);
    /* Handle is for keyboard or pipe */
    if (type == FILE_TYPE_CHAR || type == FILE_TYPE_PIPE) {
        int ret;
        long lpbytes;
        HANDLE stdInHandle = GetStdHandle(STD_INPUT_HANDLE);
        if (stdInHandle == h) {
            ret = handleStdinAvailable(fd, &lpbytes); /* keyboard */
        } else {
            ret = handleNonSeekAvailable(fd, &lpbytes); /* pipe */
        }
        (*pbytes) = (jlong)(lpbytes);
        return ret;
    }
    /* Handle is for regular file */
    if (type == FILE_TYPE_DISK) {
        jlong current, end;

        LARGE_INTEGER filesize;
        current = handleLseek(fd, 0, SEEK_CUR);
        if (current < 0) {
            return FALSE;
        }
        if (GetFileSizeEx(h, &filesize) == 0) {
            return FALSE;
        }
        end = long_to_jlong(filesize.QuadPart);
        *pbytes = end - current;
        return TRUE;
    }
    return FALSE;
}
```
其中FILE_TYPE_DISK处就是对本地文件的处理，调用`GetFileSizeEx`去获取文件本身的大小，而不是从输入流中获得相关的数据大小。对于非本地文件的输入流，依然通过输入流获取可读数据长度，也即不一定得到整个流的数据长度。

### 4. 修正
如果我们的输入流来源于本地文件，我们可以像文章最开始那样通过`available`来判断是否还有数据读取，然后再做数据处理操作。但最好不要这样使用，因为对于文件流本身来说，除了本地文件还有网络文件等场景。如果基于本地文件的`available`的特性来使用`FileInputStream`，会造成`FileInputStream`的不可移植性，不适用去非本地文件的其他模式。

那么对于最开始的场景，我们可以使用如下方式来处理：

```java
byte[] data = new byte[10240];
try (BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(file, true))) {
    int length;
    while ((length = in.read(data)) != -1) {
        out.write(data, 0, length);
    }
    out.flush();
} finally {
    in.close();
}
```
每次阻塞读取数据，直到读取到流末尾。这样就能保证将输入流的数据进行完全的处理。


