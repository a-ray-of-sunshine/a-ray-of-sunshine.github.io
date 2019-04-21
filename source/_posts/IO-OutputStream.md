---
title: OutputStream
date: 2016-10-2 16:36:01
---

## OutputStream

``` java
public abstract class OutputStream implements Closeable, Flushable{

	// 向流中写入 b 低8位字节，高 24 位字节被丢弃。
    public abstract void write(int b) throws IOException;

	// 将缓存的字节全部写入底层的流中。
    public void flush() throws IOException {
    }

	// 关闭流
    public void close() throws IOException {
    }

	// ...
}
```

### ByteArrayOutputStream

这个类内部持有一个字节数据，write 方法就是将字节数据写入到该字节数据中。注意底层的 ByteArray 会随着不断的 write 而增大，所以底层的 Array 是无限大的。当然一旦写入的数据个数超过：Integer.MAX_VALUE，则OutOfMemoryError()。

``` java

public class ByteArrayOutputStream extends OutputStream {

    /**
     * The buffer where data is stored.
     */
    protected byte buf[];

    /**
     * The number of valid bytes in the buffer.
     */
    protected int count;
}
```

* buf

	存储字节数据

* count

	buf 中写入的有效字节个数

### FileOutputStream

这个类的实现可以参考 openjdk1.7 的实现。

#### 构造函数

``` java
public FileOutputStream(File file, boolean append)
    throws FileNotFoundException
{
	// 获得文件的路径
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkWrite(name);
    }
    if (name == null) {
        throw new NullPointerException();
    }

	// 创建 fd 对象
    this.fd = new FileDescriptor();
    this.append = append;

    fd.incrementAndGetUseCount();
    // 打开文件。这是一个 native 方法
	// 在这个方法中将使用 name 打开文件，初始化 fd 中的文件描述符。
	open(name, append);
}
```

#### open 的实现


open 的实现在 jdk 中

``` c
// src/solaris/native/java/io
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_open(JNIEnv *env, jobject this,
                                   jstring path, jboolean append) {
    fileOpen(env, this, path, fos_fd,
             O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC));
}

// src/solaris/native/java/io/io_util_md.c
void
fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
    WITH_PLATFORM_STRING(env, path, ps) {
        FD fd;

#ifdef __linux__
        /* Remove trailing slashes, since the kernel won't */
        char *p = (char *)ps + strlen(ps) - 1;
        while ((p > ps) && (*p == '/'))
            *p-- = '\0';
#endif
	    // ps 是转换后的路径path.
		// 调用 JVM_Open 方法打开文件。
        fd = JVM_Open(ps, flags, 0666);

		// fd >= 0 表示打开成功，返回一个有效的 fd
        if (fd >= 0) {
			// 将 打开的文件描述符 fd 设置到 FileOutputStream.fd 对象的
			// fd 字段中。
            SET_FD(this, fd, fid);
        } else {
            throwFileNotFoundException(env, path);
        }
    } END_PLATFORM_STRING(env, ps);
}

```

JVM_Open 方法在 jvm 的实现中。

``` c
// src/share/vm/prims/jvm.cpp
JVM_LEAF(jint, JVM_Open(const char *fname, jint flags, jint mode))
  JVMWrapper2("JVM_Open (%s)", fname);

  //%note jvm_r6
  int result = os::open(fname, flags, mode);
  if (result >= 0) {
    return result;
  } else {
    switch(errno) {
      case EEXIST:
        return JVM_EEXIST;
      default:
        return -1;
    }
  }
JVM_END

// os::open 的 linux 实现在
// src/os/linux/vm/os_linux.cpp 中
// 其中参数:
// oflag == O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC)
// mode  == 0666
int os::open(const char *path, int oflag, int mode) {

  if (strlen(path) > MAX_PATH - 1) {
    errno = ENAMETOOLONG;
    return -1;
  }
  int fd;
  int o_delete = (oflag & O_DELETE);
  oflag = oflag & ~O_DELETE;

  // open64 等价于 open 方法，这是一个 linux 系统调用，
  // 用来打开文件。其中参数：
  // oflag = O_WRONLY | O_CREAT | O_APPEND
  // O_WRONLY: write-only
  // O_CREAT: If the file does not exist it will be created.
  // O_APPEND: The  file is opened in append mode.
  // mode specifies the permissions to use in case a new file is created.
  // mode = 0666 表示，如果要打开的文件存在，则新创建的文件的权限为 666
  // 也就是 rw-rw-rw-
  fd = ::open64(path, oflag, mode);
  if (fd == -1) return -1;

  //If the open succeeded, the file might still be a directory
  {
    struct stat64 buf64;
    int ret = ::fstat64(fd, &buf64);
    int st_mode = buf64.st_mode;

    if (ret != -1) {
	  // 可以看到，如果文件成功打开，但是如果是目录的话，则 close 文件。
      if ((st_mode & S_IFMT) == S_IFDIR) {
        errno = EISDIR;
        ::close(fd);
        return -1;
      }
    } else {
      ::close(fd);
      return -1;
    }
  }

  // ...

  return fd;
}
```

### write 过程


``` c

JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_write(JNIEnv *env, jobject this, jint byte, jboolean append) {
    writeSingle(env, this, byte, append, fos_fd);
}

void
writeSingle(JNIEnv *env, jobject this, jint byte, jboolean append, jfieldID fid) {
    // Discard the 24 high-order bits of byte. See OutputStream#write(int)
	// 可以看到，在这里直接将 int 类型的 byte, 直接强制转换成 char 类型
	// 注意这是 c 语言中的 char 类型，占一个字节，和 java 中的 char 完全不同。
    char c = (char) byte;
    jint n;
	// 获得文件描述符。
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return;
    }
	// 写入一个字节。
    if (append == JNI_TRUE) {
        n = (jint)IO_Append(fd, &c, 1);
    } else {
        n = (jint)IO_Write(fd, &c, 1);
    }
    if (n == JVM_IO_ERR) {
        JNU_ThrowIOExceptionWithLastError(env, "Write error");
    } else if (n == JVM_IO_INTR) {
        JNU_ThrowByName(env, "java/io/InterruptedIOException", NULL);
    }
}


// src/solaris/natvie/java/io/io_util_md.h
#define IO_Append JVM_Write
#define IO_Write JVM_Write

// jvm.cpp
JVM_LEAF(jint, JVM_Write(jint fd, char *buf, jint nbytes))
  JVMWrapper2("JVM_Write (0x%x)", fd);

  //%note jvm_r6
  return (jint)os::write(fd, buf, nbytes);
JVM_END

// os_linux.inline.hpp
inline size_t os::write(int fd, const void *buf, unsigned int nBytes) {
  size_t res;
  // 其中 write 方法为一个系统调用：
  // writes up to count bytes from the buffer pointed buf to the file referred to by the file descriptor fd.
  RESTARTABLE((size_t) ::write(fd, buf, (size_t) nBytes), res);
  return res;
}
```

### close 过程

``` c
void
fileClose(JNIEnv *env, jobject this, jfieldID fid)
{
	// 1. 获得 fd
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        return;
    }

    /* Set the fd to -1 before closing it so that the timing window
     * of other threads using the wrong fd (closed but recycled fd,
     * that gets re-opened with some other filename) is reduced.
     * Practically the chance of its occurance is low, however, we are
     * taking extra precaution over here.
     */
	// 2. 关闭底层的fd之前，将 FileOutputStream.fd 中的 fd 字段
	// 置为无效，即 -1.
    SET_FD(this, -1, fid);

    /*
     * Don't close file descriptors 0, 1, or 2. If we close these stream
     * then a subsequent file open or socket will use them. Instead we
     * just redirect these file descriptors to /dev/null.
     */
    if (fd >= STDIN_FILENO && fd <= STDERR_FILENO) {
        int devnull = open("/dev/null", O_WRONLY);
        if (devnull < 0) {
            SET_FD(this, fd, fid); // restore fd
            JNU_ThrowIOExceptionWithLastError(env, "open /dev/null failed");
        } else {
            dup2(devnull, fd);
            close(devnull);
        }
	// 调用 JVM_Close 进行关闭。
    } else if (JVM_Close(fd) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "close failed");
    }

JVM_LEAF(jint, JVM_Close(jint fd))
  JVMWrapper2("JVM_Close (0x%x)", fd);
  //%note jvm_r6
  return os::close(fd);
JVM_END

inline int os::close(int fd) {
  // close 方法为 linux 系统调用，用来关闭文件 fd.
  return ::close(fd);
}
```

> close()  closes  a  file  descriptor, so that it no longer refers to any file and may be reused.  Any record locks (see fcntl(2)) held on the file it was associated with, and owned by the  process,  are  removed  (regardless  of  the  file descriptor that was used to obtain the lock).
>
> If  fd is the last file descriptor referring to the underlying open file description (see open(2)), the resources associated with the open file description are freed; if the descriptor was the last reference to  a  file  which  has  been removed using unlink(2) the file is deleted.

以上引用来自，linux 对 close 方法的描述，可以看到 close 方法的重要性，所以当流使用完毕一定要及时关闭。否则，底层的文件将一直被占用，无法释放，造成了资源的浪费。

### BufferedOutputStream

``` java
public class BufferedOutputStream extends FilterOutputStream{

    /**
     * The internal buffer where data is stored.
     */
    protected byte buf[];

    public BufferedOutputStream(OutputStream out) {
        this(out, 8192);
    }

    /** Flush the internal buffer */
    private void flushBuffer() throws IOException {
        if (count > 0) {
            out.write(buf, 0, count);
            count = 0;
        }
    }

	// ...
}
```

内部持有一个默认的 8K 的缓冲区 buf, write 操作优先写入这个buffer. 直到 buf 满了，调用 flushBuffer 将缓冲区内部的数据写入到底层的流中。