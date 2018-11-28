---
title: NIO中的File文件操作
date: 2017-08-21 14:50
categories: JAVA
---

之前的两篇文章
http://blog.csdn.net/holmofy/article/details/75269866
http://blog.csdn.net/holmofy/article/details/77429957
分别对传统IO对文件的基本操作以及读写进行了介绍，从Java1.7开始Java在NIO中引入了新的文件操作API。这些API在`java.nio.file`以及它的子包`java.nio.file.attribute`中。

而整个包中最核心的一个类就是Files，这个类包含了所有的所有的文件操作。

涉及到文件操作就难免要对文件路径进行各种拼接和解析等操作，但是由于操作系统路径分隔符的不同，为了跨平台于是把Path定义成接口，再由Paths这个工具类根据平台的不同选用不同的Path实现类来创建对象。

Files的大部分功能实际上由FileSystemProvider实现，而获取FileSystemProvider必须通过FileSystem，FileSystem也因为平台的原因被定义成抽象类，所以需要通过FileSystems工具类创建FileSystem实例。


![NIOFile](NIO中的File操作\NIOFile.svg)

[TOC]

# Path—路径操作

[上一篇文章中](http://blog.csdn.net/holmofy/article/details/75269866)也说过File中是一个大杂烩：路径操作，文件属性访问，文件增删操作，磁盘属性访问等全部混在了File类中。NIO中将这些功能划分开来，其中Path类就定义了路径操作的功能。Path也是`java.nio.file`包中的核心接口之一，如果你的应用使用NIO API来操作文件，那就很有必要了解一下这个类的强大功能了。

> Java1.7为了向后兼容，在传统IO`java.io.File`类中添加了`File.toPath`方法用于将File对象转成Path。
>
> 同样的`java.nio.file.Path`中也定义了一个`Path.toFile`方法向前兼容

一个Path对象可用于构建目录或文件的路径，但是Path只是一个接口，而他的实现类都在`sun.nio.fs`包下：

![Path](NIO中的File操作\Path.svg)

为了平台的无关性，Java提供了一个Paths工厂类让不同平台的JDK创建对应的Path对象。

## 使用Paths工具类创建Path对象

Paths类只有两个方法，实现也很简单：

```java
public final class Paths {
    private Paths() { }
    // 通过FileSystem根据字符串构建Path对象
    public static Path get(String first, String... more) {
        return FileSystems.getDefault().getPath(first, more);
    }
    // 通过FileSystem根据URI构建Path对象
    public static Path get(URI uri) {
        String scheme =  uri.getScheme();
        if (scheme == null)
            throw new IllegalArgumentException("Missing scheme");
        // check for default provider to avoid loading of installed providers
        if (scheme.equalsIgnoreCase("file"))
            return FileSystems.getDefault().provider().getPath(uri);
        // try to find provider
        for (FileSystemProvider provider: FileSystemProvider.installedProviders()) {
            if (provider.getScheme().equalsIgnoreCase(scheme)) {
                return provider.getPath(uri);
            }
        }
        throw new FileSystemNotFoundException("Provider \"" + scheme + "\" not installed");
    }
}
```

> FileSystem类和以及它的工厂类FileSystems会在文章的后半部分讲到。

## 1. 简单路径操作(字符操作)

Path路径本质上就是一个字符串，在没有访问文件系统之前，大部分操作只是进行字符串的操作。

```java
// 判断路径是否为绝对路径
boolean isAbsolute();
// 该路径对应的根路径(如果根路径不存在会返回null)
Path getRoot();
// 获取该路径的对应的文件(或文件夹)的名字
Path getFileName();
// 获取父路径
Path getParent();
// 获取路径中文件夹名(或文件名)的个数
int getNameCount();
// 根据索引获取路径中的文件夹名(或文件名)
Path getName(int index);
// 根据索引范围获取子路径
Path subpath(int beginIndex, int endIndex);
// 判断当前路径是否以other路径作为开头
boolean startsWith(Path other);
// 判断当前路径是否以other字符串开头
boolean startsWith(String other);
// 判断当前路径是否以other路径结尾
boolean endsWith(Path other);
// 判断当前路径是否以other字符串结尾
boolean endsWith(String other);
// 将路径名常规化去除冗余：去掉中间出现的“..”代表的上级目录，“.”代表的当前目录
// 相当于File类中的getCanonicalPath方法
Path normalize();
// 如果原路径是相对路径则将路径转为绝对路径
Path toAbsolutePath();
```

示例：

```java
public static void main(String[] args) {
   Path[] paths = {
      Paths.get(URI.create("file:///D:/dir/test.txt")), // URI路径
      Paths.get("D:/dir/test.txt"),                     // 正常路径
      Paths.get("D:/dir/../dir/test.txt"),              // 冗余路径
      Paths.get("../dir/test.txt")                      // 相对路径
   };
   for (Path path : paths) {
      System.out.println("-------" + path.toString() + "-------");
      System.out.println("normalize: " + path.normalize());
      System.out.println("getFileName: " + path.getFileName());
      System.out.println("getParent: " + path.getParent());
      System.out.println("getRoot: " + path.getRoot());
      System.out.println("isAbsolute: " + path.isAbsolute());
      System.out.println("toAbsolutePath: " + path.toAbsolutePath());
      int nameCount = path.getNameCount();
      System.out.println("getNameCount: " + nameCount);
      System.out.println("subpath(0, NameCount): " + path.subpath(0, nameCount));
      // 两种子路径名迭代方式
      for (int i = 0; i < nameCount; i++) {
         System.out.println("count-" + i + ": " + path.getName(i));
      }
      for (Path sub : path) {
         System.out.println("iterate subpath: " + sub);
      }
   }
}
```

运行结果：

```shell
-------D:\dir\test.txt-------    # URI构造路径
normalize: D:\dir\test.txt
getFileName: test.txt
getParent: D:\dir
getRoot: D:\
isAbsolute: true
toAbsolutePath: D:\dir\test.txt
getNameCount: 2
subpath(0, NameCount): dir\test.txt
count-0: dir
count-1: test.txt
iterate subpath: dir
iterate subpath: test.txt
-------D:\dir\test.txt-------    # 正常路径
normalize: D:\dir\test.txt
getFileName: test.txt
getParent: D:\dir
getRoot: D:\
isAbsolute: true
toAbsolutePath: D:\dir\test.txt
getNameCount: 2
subpath(0, NameCount): dir\test.txt
count-0: dir
count-1: test.txt
iterate subpath: dir
iterate subpath: test.txt
-------D:\dir\..\dir\test.txt-------    # 冗余路径
normalize: D:\dir\test.txt
getFileName: test.txt
getParent: D:\dir\..\dir
getRoot: D:\
isAbsolute: true
toAbsolutePath: D:\dir\..\dir\test.txt
getNameCount: 4
subpath(0, NameCount): dir\..\dir\test.txt
count-0: dir
count-1: ..
count-2: dir
count-3: test.txt
iterate subpath: dir
iterate subpath: ..
iterate subpath: dir
iterate subpath: test.txt
-------..\dir\test.txt-------    # 相对路径
normalize: ..\dir\test.txt
getFileName: test.txt
getParent: ..\dir
getRoot: null
isAbsolute: false
toAbsolutePath: D:\java\Apache Commons\commons-io2.5\..\dir\test.txt
getNameCount: 3
subpath(0, NameCount): ..\dir\test.txt
count-0: ..
count-1: dir
count-2: test.txt
iterate subpath: ..
iterate subpath: dir
iterate subpath: test.txt
```

## 2. 路径的解析与处理

```java
// 根据当前路径解析other路径
// 如果other为绝对路径则直接返回other,如果other为空路径则返回this
// 否则根据当前路径解析other路径,
// 比如:当前路径为"foo/bar",other为"gus",返回"foo/bar/gus"
Path resolve(Path other);
Path resolve(String other);
// 构造兄弟路径,即根据当前路径的父路径构造other的兄弟路径
// 比如当前路径为"/a/b",other为"c",构造得到兄弟路径"/a/c"
Path resolveSibling(Path other);
Path resolveSibling(String other);
// 根据当前路径构造other的相对路径
// 比如当前路径为"/a/b",other为"/a/b/c/d",构造得到相对路径"c/d"
Path relativize(Path other);
// 返回一个已存在的文件的路径(文件不存在会抛IO异常)
Path toRealPath(LinkOption... options) throws IOException;
// 将路径转成URI
URI toUri();
// 返回该路径对应的File对象(File类中也有一个toPath方法与之相对应)
File toFile();
```

示例：

```java
public static void main(String[] args) {
   Path path = Paths.get("D:/a/b");
   System.out.println(path.resolve(Paths.get("c")));
   System.out.println(path.resolveSibling(Paths.get("c")));
   System.out.println(path.relativize(Paths.get("D:/a/b/c")));
}
```

运行结果：

```shell
D:\a\b\c
D:\a\c
c
```

## 3. 路径(文件或目录)的监测

```java
// 注册一个监听器观察路径对应的文件或目录是否有事件(创建,删除,修改)
WatchKey register(WatchService watcher,
                  WatchEvent.Kind<?>[] events,
                  WatchEvent.Modifier... modifiers)
        throws IOException;
WatchKey register(WatchService watcher,
                      WatchEvent.Kind<?>... events)
        throws IOException;
```

![Watchable](NIO中的File操作\Watchable.svg)

例子：

```java
public class PathTest {
    public static void main(String[] args) throws IOException {
        Path path = Paths.get("D:/");
        WatchService watcher = FileSystems.getDefault().newWatchService();
        // 如果"D:/"目录下有新文件创建,删除或修改操作都会被观测到
        WatchKey token = path.register(watcher,
                StandardWatchEventKinds.ENTRY_CREATE,
                StandardWatchEventKinds.ENTRY_DELETE,
                StandardWatchEventKinds.ENTRY_MODIFY);
        while (true) {
            List<WatchEvent<?>> pollEvents = token.pollEvents();
            for (WatchEvent<?> e : pollEvents) {
                System.out.println(e.kind().name());
            }
        }
    }
}
```

# Files工具类——文件操作

Files工具类负责文件的各种操作：

## 1. 文件内容的读写操作

使用这些方法我们可以直接获取文件的IO流对象(传统IO)或文件管道(NIO)，这很大程度方便了我们的编程。

```java
// 创建文件输入流
public static InputStream newInputStream(Path path, OpenOption... options) throws IOException
{
    return provider(path).newInputStream(path, options);
}
// 创建文件输出流
public static OutputStream newOutputStream(Path path, OpenOption... options) throws IOException
{
    return provider(path).newOutputStream(path, options);
}
// 创建字节管道(FileChannel)
public static SeekableByteChannel newByteChannel(Path path, Set<? extends OpenOption> options, FileAttribute<?>... attrs) throws IOException
{
    return provider(path).newByteChannel(path, options, attrs);
}
public static SeekableByteChannel newByteChannel(Path path, OpenOption... options) throws IOException
{
    Set<OpenOption> set = new HashSet<OpenOption>(options.length);
    Collections.addAll(set, options);
    return newByteChannel(path, set);
}
```

## 2. 拷贝和移动

```java
// 将源文件拷贝到指定路径
public static Path copy(Path source, Path target, CopyOption... options);
// 将源文件移动到指定路径
public static Path move(Path source, Path target, CopyOption... options);
```

### OpenOption与CopyOption

OpenOption：用于配置如何打开或创建一个文件

CopyOption：用于配置如何拷贝或移动一个文件

LinkOption：定义如何处理符号链接

![Option](NIO中的File操作\Option.svg)

## 3. 列举目录下的文件

```java
// 非递归迭代目录下的文件(一层)
public static DirectoryStream<Path> newDirectoryStream(Path dir);
// glob：可使用通配符，如：*,?等。
public static DirectoryStream<Path> newDirectoryStream(Path dir, String glob);
// filter：过滤器
public static DirectoryStream<Path> newDirectoryStream(Path dir, DirectoryStream.Filter<? super Path> filter);
```

示例：

```java
    public static void main(String[] args) {
        Path path = Paths.get("C:/");
        try {
            DirectoryStream<Path> ds;
            ds = Files.newDirectoryStream(path);  // 默认
            for (Path p : ds) {
                System.out.println(p);
            }
            System.out.println("-------------------");
            ds = Files.newDirectoryStream(path, "*Win*");  // 通配符
            for (Path p : ds) {
                System.out.println(p);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

运行结果：

```
C:\$GetCurrent
C:\$Recycle.Bin
C:\$WINRE_BACKUP_PARTITION.MARKER
C:\AVScanner.ini
C:\Boot
C:\bootmgr
C:\BOOTNXT
C:\BOOTSECT.BAK
C:\Documents and Settings
C:\hiberfil.sys
C:\inetpub
C:\Intel
C:\pagefile.sys
C:\Program Files
C:\Program Files (x86)
C:\ProgramData
C:\Recovery
C:\swapfile.sys
C:\System Volume Information
C:\Users
C:\Windows
C:\Windows10Upgrade
-------------------
C:\$WINRE_BACKUP_PARTITION.MARKER
C:\Windows
C:\Windows10Upgrade
```

## 4. 创建与删除

```java
// 创建文件，并指定相应的文件属性
public static Path createFile(Path path, FileAttribute<?>... attrs);
// 创建目录(父目录不存在会抛出异常)
public static Path createDirectory(Path dir, FileAttribute<?>... attrs);
// 创建多级目录(父目录不存在会创建父目录)
public static Path createDirectories(Path dir, FileAttribute<?>... attrs);
// 创建临时文件
// dir指定目录;
// prefix指定前缀,可以为空;
// suffix指定后缀,可以为空，默认为.tmp
public static Path createTempFile(Path dir, String prefix, String suffix, FileAttribute<?>... attrs);
public static Path createTempFile(String prefix, String suffix, FileAttribute<?>... attrs);
// 创建临时目录
public static Path createTempDirectory(Path dir, String prefix, FileAttribute<?>... attrs);
public static Path createTempDirectory(String prefix, FileAttribute<?>... attrs)
// 创建一个符号链接，如果JVM权限不够可能会抛出异常
public static Path createSymbolicLink(Path link, Path target, FileAttribute<?>... attrs);
// 创建一个硬链接，如果JVM权限不够可能会抛出异常
public static Path createLink(Path link, Path existing);
// 删除文件或目录，如果目录不为空会抛出异常
public static void delete(Path path);
// 如果文件存在就删除
public static boolean deleteIfExists(Path path);
```

## 5. 其他

```java
// 读取符号链接指向的路径
public static Path readSymbolicLink(Path link);
// 获取该路径的FileStore对象
public static FileStore getFileStore(Path path);
// 判断两个路径是否指向同一个文件
public static boolean isSameFile(Path path, Path path2);
// 该路径所指向的文件是否为隐藏文件
public static boolean isHidden(Path path);
// 判断文件类型，无法判断则返回null
public static String probeContentType(Path path);
```

FileStore示例：

```java
    public static void main(String[] args) {
        Path path = Paths.get("C:/");
        try {
            FileStore fs = Files.getFileStore(path);
            System.out.println("TotalSpace: " + fs.getTotalSpace());
            System.out.println("UnallocatedSpace: " + fs.getUnallocatedSpace());
            System.out.println("UsableSpace: " + fs.getUsableSpace());
            System.out.println("Name: " + fs.name());
            System.out.println("Type: " + fs.type());
            System.out.println("ReadOnly: " + fs.isReadOnly());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

运行结果：

```true
TotalSpace: 117369143296
UnallocatedSpace: 49184604160
UsableSpace: 49184604160
Name: 操作系统
Type: NTFS
ReadOnly: false
```

ContentType示例：

```java
    public static void main(String[] args) {
        Path path = Paths.get("D:/sort.txt");
        try {
            // 对于大部分文件该方法还是无法判断文件内容类型
            System.out.println(Files.probeContentType(path));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

运行结果：

```shell
text/plain
```

## 6. 属性访问

```java
/**读取FileAttributeView的子类的相关属性**/
public static <V extends FileAttributeView> V getFileAttributeView(Path path, Class<V> type, LinkOption... options);
/**读取BasicAttributeView的子类相关属性**/
public static <A extends BasicFileAttributes> A readAttributes(Path path, Class<A> type, LinkOption... options);
/**设置文件的相关属性**/
public static Path setAttribute(Path path, String attribute, Object value, LinkOption... options);
/**获取文件的相关属性**/
public static Object getAttribute(Path path, String attribute, LinkOption... options);
/**读取文件的相关属性(多个值)**/
public static Map<String,Object> readAttributes(Path path, String attributes, LinkOption... options);
/**获取Posix规定的文件属性，Owner|Group|Other 的 Read|Write|Execute 权限**/
public static Set<PosixFilePermission> getPosixFilePermissions(Path path, LinkOption... options);
/**设置Posix规定的文件属性**/
public static Path setPosixFilePermissions(Path path, Set<PosixFilePermission> perms);
/**获取文件的所有者信息**/
public static UserPrincipal getOwner(Path path, LinkOption... options);
/**设置文件的所有者信息**/
public static Path setOwner(Path path, UserPrincipal owner);
/**该文件是否为符号链接文件**/
public static boolean isSymbolicLink(Path path);
/**该文件是否为目录文件**/
public static boolean isDirectory(Path path, LinkOption... options);
/**是否为常规文件**/
public static boolean isRegularFile(Path path, LinkOption... options);
/**查看文件上次修改的时间**/
public static FileTime getLastModifiedTime(Path path, LinkOption... options);
/**设置文件上次修改的时间**/
public static Path setLastModifiedTime(Path path, FileTime time);
/**获取文件的长度大小**/
public static long size(Path path);
```

### AttributeView

所有的Attribute相关类都定义在`java.nio.file.attribute`包下，这个包下的类提供了访问文件(或文件系统)的属性。该包下关键接口的关系图如下：

![AttributeView](NIO中的File操作\AttributeView.svg)

|              接口              |         功能          |
| :--------------------------: | :-----------------: |
|        AttributeView         |  可读写文件系统中对象的非透明属性   |
|      FileAttributeView       |       可读写文件属性       |
|    BasicFileAttributeView    |     可读写基本的文件属性      |
|    PosixFileAttributeView    |   可读写POSIX标准定义的属性   |
|     DosFileAttributeView     | 可读写DOS FAT文件系统定义的属性 |
|    FileOwnerAttributeView    |      可读写文件的所有者      |
| UserDefinedFileAttributeView |     可读写用户自定义属性      |
|     AclFileAttributeView     |   可读写ACL访问控制列表属性    |
|    FileStoreAttributeView    |     可读写文件系统的属性      |

Attribute示例：

```java

    public static void main(String[] args) {
        Path path = Paths.get("D:/test.txt");
        try {
            BasicFileAttributes basicAttrs = Files.readAttributes(path,
                    BasicFileAttributes.class);
            AclFileAttributeView aclView = Files.getFileAttributeView(path,
                    AclFileAttributeView.class);

            System.out.println("--------BasicFileAttributes--------");
            System.out.println("CreateTime: " + basicAttrs.creationTime());
            System.out.println("FileKey: " + basicAttrs.fileKey());
            System.out.println("IsDirectory: " + basicAttrs.isDirectory());
            System.out.println("IsRegularFile: " + basicAttrs.isRegularFile());
            System.out.println("IsSymbolicLink: " + basicAttrs.isSymbolicLink());
            System.out.println("LastAccessTime: " + basicAttrs.lastAccessTime());
            System.out.println("Size: " + basicAttrs.size());

            System.out.println("--------ACL--------");
            List<AclEntry> acl = aclView.getAcl();
            for (AclEntry entry : acl) {
                System.out.print(entry.principal().getName() + ":");
                System.out.println(entry.permissions());
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

运行结果：

```shell
--------BasicFileAttributes--------
CreateTime: 2017-07-26T08:25:48.577378Z
FileKey: null
IsDirectory: false
IsRegularFile: true
IsSymbolicLink: false
LastAccessTime: 2017-07-26T08:25:48.577378Z
Size: 1921
--------ACL--------
BUILTIN\Administrators:[READ_NAMED_ATTRS, WRITE_NAMED_ATTRS, APPEND_DATA, READ_ACL, WRITE_OWNER, DELETE_CHILD, SYNCHRONIZE, WRITE_DATA, WRITE_ATTRIBUTES, READ_DATA, DELETE, WRITE_ACL, READ_ATTRIBUTES, EXECUTE]
NT AUTHORITY\SYSTEM:[READ_NAMED_ATTRS, WRITE_NAMED_ATTRS, APPEND_DATA, READ_ACL, WRITE_OWNER, DELETE_CHILD, SYNCHRONIZE, WRITE_DATA, WRITE_ATTRIBUTES, READ_DATA, DELETE, WRITE_ACL, READ_ATTRIBUTES, EXECUTE]
NT AUTHORITY\Authenticated Users:[READ_DATA, READ_NAMED_ATTRS, WRITE_NAMED_ATTRS, APPEND_DATA, DELETE, READ_ACL, READ_ATTRIBUTES, SYNCHRONIZE, WRITE_DATA, WRITE_ATTRIBUTES, EXECUTE]
BUILTIN\Users:[READ_DATA, READ_NAMED_ATTRS, READ_ACL, READ_ATTRIBUTES, SYNCHRONIZE, EXECUTE]
```

## 7. 可访问性

```java
/**文件是否存在**/
public static boolean exists(Path path, LinkOption... options);
/**文件是否不存在**/
public static boolean notExists(Path path, LinkOption... options);
/**文件是否可读**/
public static boolean isReadable(Path path);
/**文件是否可写**/
public static boolean isWritable(Path path);
/**文件是否可执行**/
public static boolean isExecutable(Path path);
```

## 8. 递归操作

```java
/**遍历文件树**/
public static Path walkFileTree(Path start, Set<FileVisitOption> options, int maxDepth, FileVisitor<? super Path> visitor);
/**遍历文件树**/
public static Path walkFileTree(Path start, FileVisitor<? super Path> visitor);
```

递归示例：

```java
    public static void main(String[] args) {
        try {
            String javaHome = System.getenv("JAVA_HOME");
            Path path = Paths.get(javaHome,"include");
            Files.walkFileTree(path, new SimpleFileVisitor<Path>() {
                @Override
                public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
                        throws IOException {
                    System.out.println(file);
                    // 如果这里返回TERMINATE将会立即终止目录的遍历
                    return FileVisitResult.CONTINUE;
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

输出结果：

```shell
E:\JDevTools\jdk1.8.0_131\include\classfile_constants.h
E:\JDevTools\jdk1.8.0_131\include\jawt.h
E:\JDevTools\jdk1.8.0_131\include\jdwpTransport.h
E:\JDevTools\jdk1.8.0_131\include\jni.h
E:\JDevTools\jdk1.8.0_131\include\jni.h.gch
E:\JDevTools\jdk1.8.0_131\include\jvmti.h
E:\JDevTools\jdk1.8.0_131\include\jvmticmlr.h
E:\JDevTools\jdk1.8.0_131\include\win32\bridge\AccessBridgeCallbacks.h
E:\JDevTools\jdk1.8.0_131\include\win32\bridge\AccessBridgeCalls.c
E:\JDevTools\jdk1.8.0_131\include\win32\bridge\AccessBridgeCalls.h
E:\JDevTools\jdk1.8.0_131\include\win32\bridge\AccessBridgePackages.h
E:\JDevTools\jdk1.8.0_131\include\win32\jawt_md.h
E:\JDevTools\jdk1.8.0_131\include\win32\jni_md.h
```

## 9. 文件内容读写的简单包装

```java
// 打开一个文本文件，并以BufferedReader进行读取
public static BufferedReader newBufferedReader(Path path, Charset cs);
public static BufferedReader newBufferedReader(Path path);
// 打开一个文本文件，并以BufferedWriter进行写入
public static BufferedWriter newBufferedWriter(Path path, Charset cs, OpenOption... options);
public static BufferedWriter newBufferedWriter(Path path, OpenOption... options);

// 将输入流写入到指定路径
public static long copy(InputStream in, Path target, CopyOption... options);
// 将指定路径的文件写入一个输出流
public static long copy(Path source, OutputStream out);

// 读取指定路径文件的所有字节
public static byte[] readAllBytes(Path path);
// 读取指定路径文本文件的所有行
public static List<String> readAllLines(Path path, Charset cs);
public static List<String> readAllLines(Path path);

// 将字节数组写入指定路径的文件中
public static Path write(Path path, byte[] bytes, OpenOption... options);
// 将多个字符串写入指定路径的文件中
public static Path write(Path path, Iterable<? extends CharSequence> lines, Charset cs, OpenOption... options);
public static Path write(Path path, Iterable<? extends CharSequence> lines, OpenOption... options);
```

## 10. Stream函数式编程

Lambda与函数式编程可参考[这篇文章](http://blog.csdn.net/holmofy/article/details/77481304)。

```java
// newDirectoryStream方法的链式调用方式
public static Stream<Path> list(Path dir);
// walkFileTree方法的链式调用方式
public static Stream<Path> walk(Path start, int maxDepth, FileVisitOption... options);
public static Stream<Path> walk(Path start, FileVisitOption... options);
// walkFileTree方法遍历查找的链式调用方式
public static Stream<Path> find(Path start, int maxDepth, BiPredicate<Path, BasicFileAttributes> matcher, FileVisitOption... options);
// readAllLines方法的链式调用方式
public static Stream<String> lines(Path path, Charset cs);
public static Stream<String> lines(Path path);
```

链式调用示例：

```java
public class FilesTest {
    public static void main(String[] args) {
        try {
            // 列出JDK安装目录下的文件或文件夹
            String javaHome = System.getenv("JAVA_HOME");
            Path path = Paths.get(javaHome);
            Files.list(path).map(p -> p.getFileName())
                    .sorted()
                    .forEach(System.out::println);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```shell
bin
classes
classes.zip
COPYRIGHT
db
docs
include
javafx-src.zip
jre
lib
LICENSE
README.html
release
src.zip
THIRDPARTYLICENSEREADME-JAVAFX.txt
THIRDPARTYLICENSEREADME.txt
```

# FileSystem与FileSystems

FileSystem是访问文件或其他文件系统对象的工厂类。它是个抽象类，我们可以使用FileSystems这个工具类来创建FileSystem对象。

FileSystem使用实例：

```java
public class FileSystemTest {
    public static void main(String[] args) {
        // 使用FileSystems工具类获取FileSystem对象
        FileSystem fs = FileSystems.getDefault();

        // 获取文件系统的文件名路径分隔符
        System.out.println("Separator: " + fs.getSeparator());
        // Windows为"\",*nix为"/"

        // 获取文件系统支持的FileAttributeView
        Set<String> attrSet = fs.supportedFileAttributeViews();
        for (String attr : attrSet) {
            System.out.println("Attr: " + attr);
        }
        // 获取文件存储(存储设备,磁盘分区)
        // FileStore在Files工具类的其他类方法中有个例子
        Iterable<FileStore> fstore = fs.getFileStores();
        for (FileStore s : fstore) {
            System.out.println("FileStore: " + s);
        }
        // 获取根目录
        Iterable<Path> rootDirectories = fs.getRootDirectories();
        for (Path p : rootDirectories) {
            System.out.println("RootDirectorie: " + p);
        }
        try {
            // 路径匹配器
            PathMatcher glob = fs.getPathMatcher("glob:**.java");
            Files.walk(Paths.get(".")).filter(glob::matches).forEach(System.out::println);
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            // 获取用户信息
            UserPrincipalLookupService princLookup = fs.getUserPrincipalLookupService();
            UserPrincipal princepal = princLookup.lookupPrincipalByName("Administrator");
            System.out.println(princepal.getName());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```shell
Separator: \
Attr: owner
Attr: dos
Attr: acl
Attr: basic
Attr: user
FileStore: 系统 (C:)
FileStore: 代码 (D:)
FileStore: 软件 (E:)
FileStore: 虚拟机 (F:)
RootDirectorie: C:\
RootDirectorie: D:\
RootDirectorie: E:\
RootDirectorie: F:\
.\src\main\java\cn\hff\io\FilesTest.java
.\src\main\java\cn\hff\io\FileSystemTest.java
.\src\main\java\cn\hff\io\FileTest.java
.\src\main\java\cn\hff\io\PathTest.java
.\src\main\java\cn\hff\io\PipeTest.java
.\src\main\java\cn\hff\io\PrintTest.java
.\src\main\java\cn\hff\io\RecurseFile.java
.\src\main\java\cn\hff\io\ScannerTest.java
.\src\main\java\cn\hff\io\SystemIO.java
.\src\main\java\cn\hff\nio\SelectorTest.java
.\src\main\java\cn\hff\nio\ServerChannelTest.java
DESKTOP-DQJB4BI\Administrator
```

上面的例子方法都很简单，不过PathMatcher需要详细介绍一下

## PathMatcher

在传统IO的File类中我们通常使用FileFilter或FilenameFilter来过滤文件，Common-IO中提供了这两个接口大量的实现类，其中有两个实现类就是`WildcardFileFilter`和`RegexFileFilter`分别代表了使用通配符过滤文件名、使用正则表达式过滤文件名，而在Java7提供的`FileSystem.getPathMatcher`方法返回了一个PathMatcher对象，这个PathMatcher对象就能实现`WildcardFileFilter`和`RegexFileFilter`两者的功能。

PathMatcher是一个[函数式接口](http://blog.csdn.net/holmofy/article/details/77481304)，它只有一个方法`boolean matches(Path path)`，这个方法和FilenameFilter接口的`boolean accept(File dir, String name)`功能类似——用于检测路径是否符合指定规则。`FileSystem.getPathMatcher`返回的PathMatcher实现对象实现了一个规则，下面就让我们看一下这个方法：

```java
public abstract class FileSystem
    implements Closeable {
   ...
   /**
    * syntaxAndPattern用于制定匹配规则
    */
   public abstract PathMatcher getPathMatcher(String syntaxAndPattern);
   ...
}
```

syntaxAndPattern参数语法如下：

`<syntax>:<pattern>`

syntax可以有两个选择："glob"或"regex"

* 当为glob时，表示使用通配符模式匹配路径，后面的pattern将会以通配符的形式解析成过滤规则
* 当为regex时，表示使用正则表达式模式匹配路径，后面的pattern将会以正则表达式的形式解析成过滤规则

>  正则表达式模式这里就不介绍了，具体可以参考[这里的几篇文章](http://blog.csdn.net/Holmofy/article/category/7015920)。

glob是一种比正则表达式更简单的字符串匹配的语法规则，通常用来匹配文件名或文件路径，它的语法规则如下：

`*`：匹配0个到多个任意字符，和正则里面的`*`功能类似，但正则的`*`只表示数量不表示匹配字符。

`?`：匹配一个任意字符，正则里的`?`表示0个或1个，正则中的`?`也只是表示数量不表示匹配字符。

`[abc]`：方括号中的任意一个字符，和正则里的语法一致。

`[a-z]`：匹配方括号中指定范围的一个字符，和正则语法一致。

`[!abc]`：`[abc]`取反，正则使用`^`表示取反。

除了上面几种通用的写法，Java还支持以下几种写法：

`{str1,str2,...}`：匹配花括号内的任意一个字符串，和正则中的`str1|str2`语法类似。

`**`：两个`*`可以穿过文件(夹)分隔符。

下面是官方文档中给出的例子：

| 例子               | 功能                                       |
| ---------------- | ---------------------------------------- |
| `*.java`         | 匹配所有以`.java`结尾的文件                        |
| `*.*`            | 匹配文件名中包含`.`的文件                           |
| `*.{java,class}` | 匹配以 `.java`或 `.class`结尾的文件名              |
| `foo.?`          | 匹配以 `foo.`开头且任意单个字符结尾的文件名                |
| `/home/*/*`      | 匹配home目录下的任意两级子目录，如`/home/gus/data`      |
| `/home/**`       | 匹配home目录下的任意子目录，如 `/home/gus` 和`/home/gus/data` |
| `C:\\*`          | 匹配C盘下的任意一级子目录，如 `C:\foo` and `C:\bar`  (前一个反斜线是用来转义，所以在Java语言中该匹配串应该写成:  `"C:\\\\*"`) |

> glob的更多内容可以参考Wiki：https://en.wikipedia.org/wiki/Glob_(programming)