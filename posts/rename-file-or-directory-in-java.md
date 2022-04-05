在文件操作中，会涉及到重命名文件，或者移动文件。我们至少有如下几种方式来处理文件的重命名。

### 1. File#renameTo(dest)
Java IO中的File类，提供了`renameTo`方法用于重命名文件。`renameTo`方法会在没有目标文件的写入权限时抛出`SecurityException`异常；在目标文件为`null`时，抛出`NullPointerException`。而其他原因重命名失败时，仅仅只返回`false`，成功时返回`true`。

#### 1.1 重命名文件
* 同目录

    ```java
@Test
public void testIORenameFile() throws IOException {
    File src = new File("/test/file/src");
    if (!src.exists()) {
        src.createNewFile();
    }
    boolean ret = src.renameTo(new File("/test/file/dest"));
    assert ret;
}
    ```
    此时如果在已经存在目标文件，则会覆盖。但如果目标文件和源文件类型不同，比如源文件是文件类型，而已经存在的目标文件是目录类型，此时会返回`false`，且无任何异常。

* 不同目录

    ```java
    @Test
    public void testIORenameFileInDiffDir() throws IOException {
        File src = new File("/test/file/src");
        if (!src.exists()) {
            src.createNewFile();
        }
        File targetFile = new File("/test/file-another/dest");
        if(!targetFile.getParentFile().exists()){
            targetFile.getParentFile().mkdirs();
        }
        boolean ret = src.renameTo(targetFile);
        assert ret;
    }
    ```
此时，如果目标目录不存在，则会返回`false`，断言无法通过，且无任何异常。

#### 1.2 重命名目录

```java
@Test
public void testIORenameDirectory() throws IOException {
    File src = new File("/test/file/src");
    deleteUnEmptyDir(src);
    src.mkdirs();

    File subFile = new File(src, "subFile");
    if(!subFile.exists()){
        subFile.createNewFile();
    }

    boolean ret = src.renameTo(new File("/test/file/dest"));
    assert ret;
}
```
如果目标目录存在，则必须原目录和存在的目标目录都为空目录则可以重命名（覆盖）成功，否则都会返回`false`，切无异常。
_注意_：`File#delete`方法只能删除文件或非空目录。

### 2. Files#move(src,dest)
java1.7增加了`java.nio.file.Files`，可以使用其中的`move`方法来实现重命名，并能在重命名或移动失败时，抛出错误信息，而非`File#rename`仅仅返回`false`。
   
### 2.1 重命名文件

* 同目录

  ```java
    @Test
    public void testNIORename() throws IOException {
        File src = new File("/test/file/src");
        if (!src.exists()) {
            src.createNewFile();
        }
        Path path = Files.move(src.toPath(), Paths.get("/test/file/dest"), StandardCopyOption.REPLACE_EXISTING);
        assert path.equals(Paths.get("/test/file/dest"));
    }
    ```
    
    其中指定了移动方式为：`StandardCopyOption.REPLACE_EXISTING`，表示目标对象存在时，进行覆盖。和`File#renameTo`不同的是，如果目标对象存在，且是目录时，只有当是非空目录时会抛出`DirectoryNotEmptyException`，而如果是空目录时，也会替换成原文件。

* 不同目录

    ```java
    @Test
    public void testNIORenameInDiffDirs() throws IOException {
        File src = new File("/test/file/src");
        if (!src.exists()) {
            src.createNewFile();
        }
        Path path = Files.move(src.toPath(), Paths.get("/test/file-another/dest"), StandardCopyOption.REPLACE_EXISTING);
        assert path.equals(Paths.get("/test/file-another/dest"));
    }
    ```
和`File#renameTo`一样，当目标文件所在的目录不存在时，会重命名失败。但`Files.move`会抛出异常，说明失败原因，如：`java.nio.file.NoSuchFileException: /test/file/src -> /test/file-another/dest`

#### 2.2 重命名目录

```java
@Test
public void testNIORenameDirectory() throws IOException {
    File src = new File("/test/file/src");
    deleteUnEmptyDir(src);
    src.mkdirs();

    File subFile = new File(src, "subFile");
    if(!subFile.exists()){
        subFile.createNewFile();
    }
    Path path = Files.move(src.toPath(), Paths.get("/test/file/dest"), StandardCopyOption.REPLACE_EXISTING);
    assert path.equals(Paths.get("/test/file/dest"));
}
```
和`File#renameTo`一样，如果目标目录存在，则必须原目录和存在的目标目录都为空目录则可以重命名（覆盖）成功，否则抛出重命名失败的原因。

### 3. guava的`Files.move(srcFile,destFile)`

#### 3.1 重命名文件

* 同目录

    ```java
    @Test
    public void testGuavaRenameFileInSameDir() throws IOException {
        File src = new File("/test/file/src");
        if (!src.exists()) {
            src.createNewFile();
        }
        com.google.common.io.Files.move(src, new File("/test/file/dest"));
    }
    ```
guava的`Files#move`先使用`File#renameTo`进行重命名，如果失败了，则使用复制文件流的方式，将文件复制到目标文件中。

* 不同目录
    
    ```java
@Test
public void testGuavaRenameInDiffDirs() throws IOException {
    File src = new File("/test/file/src");
    if (!src.exists()) {
        src.createNewFile();
    }
    com.google.common.io.Files.move(src, new File("/test/file-another/dest"));
}
    ```
如果目标文件所在的文件夹不存在，也即`File#renameTo`会返回`false`，那么就会使用复制的方式进行处理，最终会抛出文件不存在的异常。

#### 3.2 重命名目录
```java
public void testGuavaRenameDirectory() throws IOException {
    File src = new File("/test/file/src");
    deleteUnEmptyDir(src);
    src.mkdirs();

    File subFile = new File(src, "subFile");
    if(!subFile.exists()){
        subFile.createNewFile();
    }
    com.google.common.io.Files.move(src, new File("/test/file/dest"));
}
```
和`File#renameTo`一样，如果目标目录存在，则必须原目录和存在的目标目录都为空目录则可以重命名（覆盖）成功，否则抛出`java.io.FileNotFoundException: /test/file/src (Is a directory)`，因为此时guava又使用文件流进行复制（对文件夹无效）。

### 4. 总结
这里对比了三种重命名文件(目录)的方式。

* 对于java1.7版本之前的重命名操作，如果使用`File#renameTo`，则一定要检测其返回值，并针对不同的返回值做相应的处理。否则可能会出现程序无任何异常，但出现的结果却和程序逻辑不一致。可尽量使用类似guava之类的工具类来处理此类逻辑。
* java1.7或更新版本，尽量使用`Files#move`来完成重命名操作。`File#renameTo`除了重命名失败时，不抛出异常，并且在不同的平台上也不能保证其一致性。

    > The rename method didn't work consistently across platforms.

    具体参考：[Legacy File I/O Code](http://docs.oracle.com/javase/tutorial/essential/io/legacy.html)

