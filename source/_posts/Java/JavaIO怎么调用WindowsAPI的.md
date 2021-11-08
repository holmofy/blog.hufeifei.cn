---
title: JavaIO怎么调用WindowsAPI的——从Native层剖析JavaIO文件读写
date: 2017-08-13
categories: JAVA
---

[上一篇文章](http://blog.csdn.net/holmofy/article/details/75269866)中列举了JavaIO中FileDescriptor和File类提供的一些文件操作，这些操作还只是对文件系统中的文件进行创建或删除操作。鉴于大一玩过Window编程(对Linux API不是很熟悉)，所以这篇文章会从Windows C API去分析一下Java提供给我们的几个文件读写类。

> 建议结合着源代码看这篇文章(这篇文章就是记录我看源代码的过程，这里的java版本是1.8.0_131)

# RandomAccessFile

这个类就是完全模仿C语言的文件读写操作，允许随机读取，想读文件的哪个部分就可以把文件流指针指到哪儿。下面会列一张表将这个类中的常用方法和标准C语言API进行对比，然后再看一下Java在Native层是怎么实现这个类的：

|                   Java                   |                    C                     |
| :--------------------------------------: | :--------------------------------------: |
| `public int read(byte b[], int off, int len)`<br/>`public int read(byte b[])`<br/>`public final void readFully(byte b[])`<br/>`public final void readFully(byte b[], int off, int len)` | `size_t fread( void *buffer, size_t size, size_t count, FILE *stream );` |
|           `public int read()`            | `int fgetc( FILE *stream );`<br/>`int getc( FILE *stream );` |
| `public void write(byte b[])`<br/>`public void write(byte b[], int off, int len)` | `size_t fwrite( const void *buffer, size_t size, size_t count, FILE *stream );` |
|        `public void write(int b)`        | `int fputc( int ch, FILE *stream );`<br/> `int putc( int ch, FILE *stream );` |
| `public void seek(long pos)`<br/>`public int skipBytes(int n)` | `int fseek( FILE *stream, long offset, int origin );`<br/>`int fsetpos( FILE *stream, const fpos_t *pos );`<br/>` void rewind(FILE *stream);` |
|  `public native long getFilePointer()`   | `long ftell( FILE *stream );`<br/>`int fgetpos( FILE *stream, fpos_t *pos );` |

RandomAccessFile还同时实现了`DataOutput, DataInput`两个接口，所以同时拥有了`DataInputStream`和`DataOutputStream`两个类的基本方法。

![RandomAccessFile](http://img-blog.csdn.net/20170820162027077?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### open

```java
    // 首先从构造方法开始看
    public RandomAccessFile(String name, String mode)
        throws FileNotFoundException
    {
        this(name != null ? new File(name) : null, mode);
    }
    // mode参数指定文件的打开模式
    public RandomAccessFile(File file, String mode)
        throws FileNotFoundException
    {
        String name = (file != null ? file.getPath() : null);
        int imode = -1;
        // r 表示“只读”,调用写操作将会抛出IO异常
        if (mode.equals("r"))
            imode = O_RDONLY;
        // rw 表示“可读可写”,如果文件不存在就会创建它
        else if (mode.startsWith("rw")) {
            imode = O_RDWR;
            rw = true;
            if (mode.length() > 2) {
                // rws 表示每一次写入操作文件内容(content)或元数据(metadata)，
                // 底层存储设备也会同步写入
                if (mode.equals("rws"))
                    imode |= O_SYNC;
                // rws 表示每一次写入操作文件内容(content)
                // 底层存储设备也会同步写入
                else if (mode.equals("rwd"))
                    imode |= O_DSYNC;
                else
                    imode = -1;
                // “rwd”模式可用于减少执行的I / O操作的数量。
                // 使用“rwd”更新时只要写入文件内容;
                // 使用“rws”更新时要写入文件内容以及文件元数据(文件大小,文件名等信息)，
                // 而文件元数据的更新,通常需要至少一个底层IO操作
            }
        }
        if (imode < 0)
            throw new IllegalArgumentException("Illegal mode \"" + mode
                                               + "\" must be one of "
                                               + "\"r\", \"rw\", \"rws\","
                                               + " or \"rwd\"");
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
            if (rw) {
                security.checkWrite(name);
            }
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);
        path = name;
        // 最终辗转调用native方法
        open(name, imode);
    }
    private void open(String name, int mode)
        throws FileNotFoundException {
        open0(name, mode);
    }
    // 最终的native方法
    private native void open0(String name, int mode)
        throws FileNotFoundException;
```

下面看一下native层是怎么打开文件的：

```c
/////////////////////////////////////////////////////////////////////////
// RandomAccessFile.c文件
JNIEXPORT void JNICALL
Java_java_io_RandomAccessFile_open(JNIEnv *env,
                                   jobject this, jstring path, jint mode)
{
    int flags = 0;
    // 构造标志位
    if (mode & java_io_RandomAccessFile_O_RDONLY)
        flags = O_RDONLY;
    else if (mode & java_io_RandomAccessFile_O_RDWR) {
        flags = O_RDWR | O_CREAT;
        if (mode & java_io_RandomAccessFile_O_SYNC)
            flags |= O_SYNC;
        else if (mode & java_io_RandomAccessFile_O_DSYNC)
            flags |= O_DSYNC;
    }
    // fileOpen
    fileOpen(env, this, path, raf_fd, flags);
}


/////////////////////////////////////////////////////////////////////////
// io_util_md.c文件
// 这里看的windows上的实现
void
fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
    FD h = winFileHandleOpen(env, path, flags);
    if (h >= 0) {
        SET_FD(this, h, fid);
    }
}

FD
winFileHandleOpen(JNIEnv *env, jstring path, int flags)
{
    // 将标志位解析成Windows API中规定的标志位
    const DWORD access =
        (flags & O_WRONLY) ?  GENERIC_WRITE :
        (flags & O_RDWR)   ? (GENERIC_READ | GENERIC_WRITE) :
        GENERIC_READ;
    const DWORD sharing =
        FILE_SHARE_READ | FILE_SHARE_WRITE;
    const DWORD disposition =
        /* Note: O_TRUNC overrides O_CREAT */
        (flags & O_TRUNC) ? CREATE_ALWAYS :
        (flags & O_CREAT) ? OPEN_ALWAYS   :
        OPEN_EXISTING;
    const DWORD  maybeWriteThrough =
        (flags & (O_SYNC | O_DSYNC)) ?
        FILE_FLAG_WRITE_THROUGH :
        FILE_ATTRIBUTE_NORMAL;
    const DWORD maybeDeleteOnClose =
        (flags & O_TEMPORARY) ?
        FILE_FLAG_DELETE_ON_CLOSE :
        FILE_ATTRIBUTE_NORMAL;
    const DWORD flagsAndAttributes = maybeWriteThrough | maybeDeleteOnClose;

    // 文件句柄
    HANDLE h = NULL;

    // 转成Windows路径
    WCHAR *pathbuf = pathToNTPath(env, path, JNI_TRUE);
    if (pathbuf == NULL) {
        /* Exception already pending */
        return -1;
    }
    // 调用windows API
    h = CreateFileW(
        pathbuf,            /* Wide char path name */
        access,             /* Read and/or write permission */
        sharing,            /* File sharing flags */
        NULL,               /* Security attributes */
        disposition,        /* creation disposition */
        flagsAndAttributes, /* flags and attributes */
        NULL);
    free(pathbuf);

    if (h == INVALID_HANDLE_VALUE) {
        throwFileNotFoundException(env, path);
        return -1;
    }
    // 最后将文件句柄作为FileDescriptor返回
    return (jlong) h;
}

//////////////////////////////////////////////////////////////////////////
// io_util.md.h文件
#define FD jlong

#define SET_FD(this, fd, fid) \
    if ((*env)->GetObjectField(env, (this), (fid)) != NULL) \
        (*env)->SetLongField(env, (*env)->GetObjectField(env, (this), (fid)), IO_handle_fdID, (fd))
```

[点击这里](https://msdn.microsoft.com/EN-US/library/windows/desktop/aa363858.aspx)可以查看`CreateFile`的API文档：

```c
// 返回文件句柄
HANDLE WINAPI CreateFile(
  // 创建(或打开)的文件名(或设备名)
  _In_     LPCTSTR               lpFileName,
  // 访问属性,这个值通常是GENERIC_READ,GENERIC_WRITE,或者(GENERIC_READ | GENERIC_WRITE).
  _In_     DWORD                 dwDesiredAccess,
  // 文件共享模式: FILE_SHARE_READ,FILE_SHARE_READ,FILE_SHARE_WRITE
  _In_     DWORD                 dwShareMode,
  // SECURITY_ATTRIBUTES结构体指针，该参数可以为NULL
  _In_opt_ LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  // 文件创建的配置:
  //   CREATE_ALWAYS,总是创建新的文件
  //   CREATE_NEW,不存在就创建新文件
  //   OPEN_ALWAYS,总是打开文件
  //   OPEN_EXISTING,仅当文件存在时打开文件
  //   TRUNCATE_EXISTING,如果文件存在,删除原来的数据
  _In_     DWORD                 dwCreationDisposition,
  // 文件的属性标记:
  // FILE_ATTRIBUTE_ARCHIVE,归档文件
  // FILE_ATTRIBUTE_ENCRYPTED,加密文件
  // FILE_ATTRIBUTE_HIDDEN,隐藏文件
  // FILE_ATTRIBUTE_NORMAL,普通文件
  // FILE_ATTRIBUTE_READONLY,只读文件
  // FILE_ATTRIBUTE_TEMPORARY,临时文件
  // ...
  _In_     DWORD                 dwFlagsAndAttributes,
  // 使用某个文件的属性作为模板
  _In_opt_ HANDLE                hTemplateFile
);
```

### read

```java
// 所有的read方法最终都会辗转调用这两个方法
private native int read0() throws IOException; // 读取一个字节
private native int readBytes(byte b[], int off, int len) throws IOException; // 读取多个字节
```

Native层代码：

```c
/////////////////////////////////////////////////////////////////////
// RandomAccessFile.c文件
JNIEXPORT jint JNICALL
Java_java_io_RandomAccessFile_read(JNIEnv *env, jobject this) {
    return readSingle(env, this, raf_fd);
}

JNIEXPORT jint JNICALL
Java_java_io_RandomAccessFile_readBytes(JNIEnv *env,
    jobject this, jbyteArray bytes, jint off, jint len) {
    return readBytes(env, this, bytes, off, len, raf_fd);
}

////////////////////////////////////////////////////////////////////
// io_util.c文件
jint
readSingle(JNIEnv *env, jobject this, jfieldID fid) {
    jint nread;
    char ret;
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return -1;
    }
    // 调用IO_Read
    nread = IO_Read(fd, &ret, 1);
    if (nread == 0) { /* EOF */
        return -1;
    } else if (nread == -1) { /* error */
        JNU_ThrowIOExceptionWithLastError(env, "Read error");
    }
    return ret & 0xFF;
}

/* 缓存大小 */
// 估计这也是为什么BufferedInputStream缓存大小为8192的原因
#define BUF_SIZE 8192

/* 越界判断 */
static int
outOfBounds(JNIEnv *env, jint off, jint len, jbyteArray array) {
    return ((off < 0) ||
            (len < 0) ||
            // We are very careful to avoid signed integer overflow,
            // the result of which is undefined in C.
            ((*env)->GetArrayLength(env, array) - off < len));
}

jint
readBytes(JNIEnv *env, jobject this, jbyteArray bytes,
          jint off, jint len, jfieldID fid)
{
    jint nread;
    // 在栈中申请的默认缓存空间
    char stackBuf[BUF_SIZE];
    char *buf = NULL;
    FD fd;

    if (IS_NULL(bytes)) {
        JNU_ThrowNullPointerException(env, NULL);
        return -1;
    }

    if (outOfBounds(env, off, len, bytes)) {
        JNU_ThrowByName(env, "java/lang/IndexOutOfBoundsException", NULL);
        return -1;
    }

    // 根据想要读取的字节数，申请对应大小的缓冲区
    if (len == 0) {
        return 0;
    } else if (len > BUF_SIZE) {
        // 申请缓冲区内存
        buf = malloc(len);
        if (buf == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            return 0;
        }
    } else {
        // 如果需要读取的字节数小于默认缓存大小
        // 则使用栈缓存区
        buf = stackBuf;
    }

    // 获取文件句柄
    fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        nread = -1;
    } else {
        // 也是调用IO_Read方法读取文件
        nread = IO_Read(fd, buf, len);
        if (nread > 0) {
            (*env)->SetByteArrayRegion(env, bytes, off, nread, (jbyte *)buf);
        } else if (nread == -1) {
            JNU_ThrowIOExceptionWithLastError(env, "Read error");
        } else { /* EOF */
            nread = -1;
        }
    }

    // 释放缓冲区内存
    if (buf != stackBuf) {
        free(buf);
    }
    return nread;
}

////////////////////////////////////////////////////////////////////
// io_util_md.h文件
#define IO_Read handleRead

////////////////////////////////////////////////////////////////////
// io_util_md.c文件
JNIEXPORT
jint
handleRead(FD fd, void *buf, jint len)
{
    DWORD read = 0;
    BOOL result = 0;
    // Windows上FileDescriptor就是Handle文件句柄
    HANDLE h = (HANDLE)fd;
    if (h == INVALID_HANDLE_VALUE) {
        return -1;
    }
    // 调用Windows API
    result = ReadFile(h,          /* File handle to read */
                      buf,        /* address to put data */
                      len,        /* number of bytes to read */
                      &read,      /* number of bytes read */
                      NULL);      /* no overlapped struct */
    if (result == 0) {
        int error = GetLastError();
        if (error == ERROR_BROKEN_PIPE) {
            return 0; /* EOF */
        }
        return -1;
    }
    return (jint)read;
}
```

[点击这里](https://msdn.microsoft.com/EN-US/library/4ad4580d-c002-44a4-a5f6-757e83ed8732.aspx)查看ReadFile的API文档

```c
BOOL WINAPI ReadFile(
  // 文件句柄
  _In_        HANDLE       hFile,
  // 接收数据的字节缓冲区
  _Out_       LPVOID       lpBuffer,
  // 读取数据的长度
  _In_        DWORD        nNumberOfBytesToRead,
  // 接受数据读取的长度，当lpOverlapped参数不为NULL，该参数可以为NULL
  _Out_opt_   LPDWORD      lpNumberOfBytesRead,
  // OVERLAPPED结构体指针，用于使用FILE_FLAG_OVERLAPPED标记打开的文件
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
```

### write

```java
// 所有的write方法最终都会调用这两个方法
private native void write0(int b) throws IOException; // 写入一个字节
private native void writeBytes(byte b[], int off, int len) throws IOException; // 写入多个字节
```

Native层源码：

```c
//////////////////////////////////////////////////////////////////////////////
// RandomAccessFile.c文件
JNIEXPORT void JNICALL
Java_java_io_RandomAccessFile_write(JNIEnv *env, jobject this, jint byte) {
    writeSingle(env, this, byte, JNI_FALSE, raf_fd);
}

JNIEXPORT void JNICALL
Java_java_io_RandomAccessFile_writeBytes(JNIEnv *env,
    jobject this, jbyteArray bytes, jint off, jint len) {
    writeBytes(env, this, bytes, off, len, JNI_FALSE, raf_fd);
}

//////////////////////////////////////////////////////////////////////////////
// io_util.c文件
void
writeSingle(JNIEnv *env, jobject this, jint byte, jboolean append, jfieldID fid) {
    // Discard the 24 high-order bits of byte. See OutputStream#write(int)
    // 在OutputStream.write(int)方法中已经通过位运算处理了int的高24位字节
    // 所以这里可以直接强转
    char c = (char) byte;
    jint n;
    // 获取文件句柄(Windows)
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return;
    }
    if (append == JNI_TRUE) {
        // 向后追加
        n = IO_Append(fd, &c, 1);
    } else {
        // 覆盖写入
        n = IO_Write(fd, &c, 1);
    }
    if (n == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Write error");
    }
}


void
writeBytes(JNIEnv *env, jobject this, jbyteArray bytes,
           jint off, jint len, jboolean append, jfieldID fid)
{
    jint n;
    // 栈内存缓冲区
    char stackBuf[BUF_SIZE];
    char *buf = NULL;
    FD fd;

    if (IS_NULL(bytes)) {
        JNU_ThrowNullPointerException(env, NULL);
        return;
    }

    if (outOfBounds(env, off, len, bytes)) {
        JNU_ThrowByName(env, "java/lang/IndexOutOfBoundsException", NULL);
        return;
    }

    if (len == 0) {
        return;
    } else if (len > BUF_SIZE) {
        buf = malloc(len);
        if (buf == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            return;
        }
    } else {
        buf = stackBuf;
    }

    // 将Java层字节数组，转成Native层字节数组
    // Java中数组是一个对象，对象头部有部分其他信息(比如数组的长度)
    // 不像C语言的字节数组那么单纯
    (*env)->GetByteArrayRegion(env, bytes, off, len, (jbyte *)buf);

    if (!(*env)->ExceptionOccurred(env)) {
        off = 0;
        // 这里要循环，IO_Append,IO_Write底层调用的WriteFile API
        // 不是让写多少就写多少
        while (len > 0) {
            fd = GET_FD(this, fid);
            if (fd == -1) {
                JNU_ThrowIOException(env, "Stream Closed");
                break;
            }
            if (append == JNI_TRUE) {
                // 向后追加
                n = IO_Append(fd, buf+off, len);
            } else {
                // 覆盖写入
                n = IO_Write(fd, buf+off, len);
            }
            if (n == -1) {
                JNU_ThrowIOExceptionWithLastError(env, "Write error");
                break;
            }
            // 已写入n个字节
            off += n;
            len -= n;
        }
    }
    // 释放内存
    if (buf != stackBuf) {
        free(buf);
    }
}

////////////////////////////////////////////////////////////////////////////
// io_util_md.h
#define IO_Append handleAppend
#define IO_Write handleWrite

////////////////////////////////////////////////////////////////////////////
// io_util_md.c
jint handleWrite(FD fd, const void *buf, jint len) {
    return writeInternal(fd, buf, len, JNI_FALSE);
}

jint handleAppend(FD fd, const void *buf, jint len) {
    return writeInternal(fd, buf, len, JNI_TRUE);
}

static jint writeInternal(FD fd, const void *buf, jint len, jboolean append)
{
    BOOL result = 0;
    DWORD written = 0;
    HANDLE h = (HANDLE)fd;// Windows中文件句柄就是FileDescriptor
    if (h != INVALID_HANDLE_VALUE) {
        OVERLAPPED ov;
        LPOVERLAPPED lpOv;
        if (append == JNI_TRUE) {
            // 构造OVERLAPPED结构体
            ov.Offset = (DWORD)0xFFFFFFFF;
            ov.OffsetHigh = (DWORD)0xFFFFFFFF;
            ov.hEvent = NULL;
            lpOv = &ov;
        } else {
            lpOv = NULL;
        }
        // 调用Windows API
        result = WriteFile(h,                /* File handle to write */
                           buf,              /* pointers to the buffers */
                           len,              /* number of bytes to write */
                           &written,         /* receives number of bytes written */
                           lpOv);            /* overlapped struct */
    }
    if ((h == INVALID_HANDLE_VALUE) || (result == 0)) {
        return -1;
    }
    return (jint)written;
}
```

[点击这里](https://msdn.microsoft.com/EN-US/library/9d6fa723-fe3e-4052-b0b3-2686eee076a7.aspx)查看WriteFile的API文档

```c
BOOL WINAPI WriteFile(
  // 文件句柄
  _In_        HANDLE       hFile,
  // 要写入的字节数据
  _In_        LPCVOID      lpBuffer,
  // 要写入多少数据
  _In_        DWORD        nNumberOfBytesToWrite,
  // 实际写入了多少数据，作为返回值
  _Out_opt_   LPDWORD      lpNumberOfBytesWritten,
  // OVERLAPPED结构体指针，对FILE_FLAG_OVERLAPPED方式打开的文件有效
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
```

### seek

```java
private native void seek0(long pos) throws IOException;
```

Native层源码：

```c
/////////////////////////////////////////////////////////////////
// RandomAccessFile.c文件
JNIEXPORT void JNICALL
Java_java_io_RandomAccessFile_seek0(JNIEnv *env,
                    jobject this, jlong pos) {

    FD fd;

    fd = GET_FD(this, raf_fd);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return;
    }
    if (pos < jlong_zero) {
        JNU_ThrowIOException(env, "Negative seek offset");
    } else if (IO_Lseek(fd, pos, SEEK_SET) == -1) {
        // 调用IO_Lseek函数
        JNU_ThrowIOExceptionWithLastError(env, "Seek failed");
    }
}

///////////////////////////////////////////////////////////////
// io_util_md.h文件
#define IO_Lseek handleLseek

///////////////////////////////////////////////////////////////
// io_util_md.c文件
jlong
handleLseek(FD fd, jlong offset, jint whence)
{
    LARGE_INTEGER pos, distance;
    DWORD lowPos = 0;
    long highPos = 0;
    DWORD op = FILE_CURRENT;
    HANDLE h = (HANDLE)fd;

    if (whence == SEEK_END) {
        op = FILE_END;
    }
    if (whence == SEEK_CUR) {
        op = FILE_CURRENT;
    }
    if (whence == SEEK_SET) {
        op = FILE_BEGIN;
    }

    distance.QuadPart = offset;
    // 调用SetFilePointerEx API
    if (SetFilePointerEx(h, distance, &pos, op) == 0) {
        return -1;
    }
    // 返回移动后的位置
    return long_to_jlong(pos.QuadPart);
}
```

[点击这里](https://msdn.microsoft.com/EN-US/library/a6fdfa00-626d-425d-b00e-c174b19ea4b9.aspx)查看SetFilePointEx的API文档

```c
BOOL WINAPI SetFilePointerEx(
  // 文件句柄
  _In_      HANDLE         hFile,
  // 要移动的距离
  _In_      LARGE_INTEGER  liDistanceToMove,
  // 移动后的文件指针位置
  _Out_opt_ PLARGE_INTEGER lpNewFilePointer,
  // 移动的相对起点：
  // FILE_BEGIN 文件开头
  // FILE_END 文件结尾
  // FILE_CURRENT 当前位置
  _In_      DWORD          dwMoveMethod
);
```

### getFilePointer

```java
public native long getFilePointer() throws IOException;
```

Native层源码：

```c
JNIEXPORT jlong JNICALL
Java_java_io_RandomAccessFile_getFilePointer(JNIEnv *env, jobject this) {
    FD fd;
    jlong ret;

    fd = GET_FD(this, raf_fd);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return -1;
    }
    // 仍然是调用IO_Lseek方法，但是最后参数(相对位置),设置为当前位置
    if ((ret = IO_Lseek(fd, 0L, SEEK_CUR)) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Seek failed");
    }
    return ret;
}
```



RandomAccessFile类最终调用的是Windows的四个API：OpenFile，ReadFile，WriteFile，GetFilePointer



# FileInputStream和FileOutputStream

FileInputStream和FileOutputStream与C++的STL中的文件流API类似：面向对象，RandomAccessFile仅仅以面向对象方式封装了文件读写。Java的文件流功能上肯定不如C++，要知道C++的运算符重载，模板类等语言特性让C++的文件操作简单了很多(相反调试也变得更加困难)。

C++中`std::basic_ifstream`代表了文件输入流，`std::basic_ofstream`代表了文件输出流，又因为C++有多继承的特性所以还有一个`std::basic_fstream`融合了前两者的功能(实际上并非继承自前两个类而是继承自basic_iostream类)。

![C++中的IO流](http://img-blog.csdn.net/20170820162211680?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

不过C++虽好，但Java的native实现仍使用C语言(实际上Java API几乎都是C语言实现，而JVM的实现Hotspot才使用到了C++)。

事实上FileInputStream和FileOutputStream的实现和RandomAccessFile几近一致：

### FileInputStream的native方法

```java
private native void open0(String name) throws FileNotFoundException;
private native int read0() throws IOException;
private native int readBytes(byte b[], int off, int len) throws IOException;
public native long skip(long n) throws IOException;
public native int available() throws IOException;
```

Native层源码：

```c
JNIEXPORT void JNICALL
Java_java_io_FileInputStream_open(JNIEnv *env, jobject this, jstring path) {
    // fileOpen，这和RandomAccessFile打开文件调用同一个函数
    fileOpen(env, this, path, fis_fd, O_RDONLY);
}

JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_read(JNIEnv *env, jobject this) {
    // readSingle，读取单个字节
    return readSingle(env, this, fis_fd);
}

JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_readBytes(JNIEnv *env, jobject this,
        jbyteArray bytes, jint off, jint len) {
    // readBytes，读取多个字节
    return readBytes(env, this, bytes, off, len, fis_fd);
}

JNIEXPORT jlong JNICALL
Java_java_io_FileInputStream_skip(JNIEnv *env, jobject this, jlong toSkip) {
    jlong cur = jlong_zero;
    jlong end = jlong_zero;
    FD fd = GET_FD(this, fis_fd);
    if (fd == -1) {
        JNU_ThrowIOException (env, "Stream Closed");
        return 0;
    }
    // IO_Lseek函数，和RandomAccessFile的seek方法实现类似
    if ((cur = IO_Lseek(fd, (jlong)0, (jint)SEEK_CUR)) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Seek error");
    } else if ((end = IO_Lseek(fd, toSkip, (jint)SEEK_CUR)) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Seek error");
    }
    return (end - cur);
}

JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_available(JNIEnv *env, jobject this) {
    jlong ret;
    FD fd = GET_FD(this, fis_fd);
    if (fd == -1) {
        JNU_ThrowIOException (env, "Stream Closed");
        return 0;
    }
    // 唯一出了点新花样的可能就是这个IO_Available函数
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

///////////////////////////////////////////////////////////////////////////////
// io_util_md.h文件
#define IO_Available handleAvailable

int
handleAvailable(FD fd, jlong *pbytes) {
    HANDLE h = (HANDLE)fd;
    DWORD type = 0;

    // 获取文件类型，调用Windows API
    type = GetFileType(h);
    /* Handle is for keyboard or pipe */
    if (type == FILE_TYPE_CHAR || type == FILE_TYPE_PIPE) {
        int ret;
        long lpbytes;
        // 获取标准输入流
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
        // 调用Windows API获取文件大小
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

查看[GetFileType](https://msdn.microsoft.com/EN-US/library/windows/desktop/aa364960.aspx)和[GetFileSizeEx](https://msdn.microsoft.com/EN-US/library/windows/desktop/aa364957.aspx)的API文档

```c
// FILE_TYPE_CHAR  字符文件，典型的如：打印设备或控制台
// FILE_TYPE_DISK  磁盘文件
// FILE_TYPE_PIPE  管道文件，如Socket，命名管道，匿名管道
// FILE_TYPE_REMOTE 未使用
// FILE_TYPE_UNKNOWN  未知设备，或者函数调用出错
DWORD WINAPI GetFileType(
  // 文件句柄
  _In_ HANDLE hFile
);

BOOL WINAPI GetFileSizeEx(
  // 文件句柄
  _In_  HANDLE         hFile,
  // 接收文件大小的长整型指针
  _Out_ PLARGE_INTEGER lpFileSize
);
```

### FileOutputStream的native方法

FileOutputStream的native实现就更简单了

```java
private native void open0(String name, boolean append) throws FileNotFoundException;
private native void write(int b, boolean append) throws IOException;
private native void writeBytes(byte b[], int off, int len, boolean append) throws IOException;
```

Native源码：

```c
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_open(JNIEnv *env, jobject this,
                                   jstring path, jboolean append) {
    // fileOpen:打开文件
    fileOpen(env, this, path, fos_fd,
             O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC));
}
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_write(JNIEnv *env, jobject this, jint byte, jboolean append) {
    // writeSingle:写入单个字节
    writeSingle(env, this, byte, append, fos_fd);
}

JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_writeBytes(JNIEnv *env,
    jobject this, jbyteArray bytes, jint off, jint len, jboolean append)
{
    // writeBytes:写入多个字节
    writeBytes(env, this, bytes, off, len, append, fos_fd);
}
```

