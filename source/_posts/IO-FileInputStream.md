---
title: IO-FileInputStream
date: 2016-9-8 22:07:26
---

## FileInputStream 的初始化

``` java
public FileInputStream(File file) throws FileNotFoundException {
	// FileInputStream 类并没有存储 file 引用，只是使用了
	// file.getPath() 来获得文件的存储路径。
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(name);
    }
    if (name == null) {
        throw new NullPointerException();
    }
    if (file.isInvalid()) {
        throw new FileNotFoundException("Invalid file path");
    }
    
    // 创建 FileDescriptor 对象，
    fd = new FileDescriptor();
    fd.incrementAndGetUseCount();
    
    this.path = name;
    
    // 打开文件
    open(name);
}

private native void open(String name) throws FileNotFoundException
```

``` c++
// FileInputStream.c
JNIEXPORT void JNICALL
Java_java_io_FileInputStream_open(JNIEnv *env, jobject this, jstring path) {
    fileOpen(env, this, path, fis_fd, O_RDONLY);
}

// jdk/windows/native/java/io/io_utils_md.c
// io 在 windows 上的实现
void
fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
	// winFileHandleOpen 将以读模式打开文件，并且其共享模式是
	// 可读可写，也就是其它线程也可以并发的对该文件进行读写
	// 如果文件不存在，则抛异常：FileNotFoundException
    jlong h = winFileHandleOpen(env, path, flags);
    
    if (h >= 0) {
    	// h >= 0 表示文件句柄有效
    	// 初始化 java.io.FileInputStream.fd 对象
    	// fd 的类型是 FileDescriptor
    	// 这里的参数： 
    	// 		this 表示当前 FileInputStream 对象对应的 oop
    	// 		h 是 打开的文件的 句柄
    	//		fid 是 全局变量:fis_fd 在 FileInputStream 加载的时候，
    	//			初始化，表示 FileInputStream 的 字段 fd 的 fieldId 
        SET_FD(this, h, fid);
    }
}


jlong
winFileHandleOpen(JNIEnv *env, jstring path, int flags)
{
	// flags == O_RDONLY
	// access = GENERIC_READ
	// 当前进程，对文件的访问模式：
	// 读模式，规定了当前线程可以对该文件进行的操作
    const DWORD access = 
        (flags & O_WRONLY) ?  GENERIC_WRITE :
        (flags & O_RDWR)   ? (GENERIC_READ | GENERIC_WRITE) :
        GENERIC_READ;
        
    // FILE_SHARE_READ 文件读共享
    // FILE_SHARE_WRITE 文件写共享
    // 表示其它进程，可以打开这个文件，并进行读写操作。
    // 规定当前线程对该文件的占用模式
    const DWORD sharing =
        FILE_SHARE_READ | FILE_SHARE_WRITE;
        
    // disposition = OPEN_EXISTING
    // 表示 如果文件不存在时，如何处理
	// OPEN_EXISTING 表示：如果文件不存在
	// CreateFile 将返回 INVALID_HANDLE_VALUE，
	// 并用 last-error code == ERROR_FILE_NOT_FOUND
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
    // 用来确定文件具有的属性
    // 一般情况下是：FILE_ATTRIBUTE_NORMAL
    // 表示：The file does not have other attributes set.
    const DWORD flagsAndAttributes = maybeWriteThrough | maybeDeleteOnClose;
    
    HANDLE h = NULL;
	
	// path 是文件所在的路径
    if (onNT) {
        WCHAR *pathbuf = pathToNTPath(env, path, JNI_TRUE);
        if (pathbuf == NULL) {
            /* Exception already pending */
            return -1;
        }
        // win32api CreateFileW 用来创建文件
        // 返回文件的句柄
        h = CreateFileW(
            pathbuf,            /* Wide char path name */
            access,             /* Read and/or write permission */
            sharing,            /* File sharing flags */
            NULL,               /* Security attributes */
            disposition,        /* creation disposition */
            flagsAndAttributes, /* flags and attributes */
            NULL);
        free(pathbuf);
    } else {
        WITH_PLATFORM_STRING(env, path, _ps) {
            h = CreateFile(_ps, access, sharing, NULL, disposition,
                           flagsAndAttributes, NULL);
        } END_PLATFORM_STRING(env, _ps);
    }
    
    // 如果打开文件失败，抛异常，返回 -1
    if (h == INVALID_HANDLE_VALUE) {
        int error = GetLastError();
        if (error == ERROR_TOO_MANY_OPEN_FILES) {
            JNU_ThrowByName(env, JNU_JAVAIOPKG "IOException",
                            "Too many open files");
            return -1;
        }
        throwFileNotFoundException(env, path);
        return -1;
    }
    
    // 返回文件的句柄
    return (jlong) h;
}

```

## 初始化 java.io.FileInputStream.fd 对象

``` c++
// windows/native/java/io/io_util_md.h
/*
 * Macros to use the right data type for file descriptors
 */
#define FD jlong

/*
 * Macros to set/get fd from the java.io.FileDescriptor.
 * If GetObjectField returns null, SET_FD will stop and GET_FD
 * will simply return -1 to avoid crashing VM.
 */
#define SET_FD(this, fd, fid) \
    if ((*env)->GetObjectField(env, (this), (fid)) != NULL) \
        (*env)->SetLongField(env, (*env)->GetObjectField(env, (this), (fid)), IO_handle_fdID, (fd))

#define GET_FD(this, fid) \
    ((*env)->GetObjectField(env, (this), (fid)) == NULL) ? \
      -1 : (*env)->GetLongField(env, (*env)->GetObjectField(env, (this), (fid)), IO_handle_fdID)
      
// SET_FD 在 fileOpen 中被调用，由于其是宏，所以在编译阶段将被展开，所以
// fileOpen 就变成：
void fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
    jlong h = winFileHandleOpen(env, path, flags);
    if (h >= 0) {
    	// GetObjectField 和 SetLongField 都是 JNI 函数，
    	// 用来操作 Java对象 FileInputStream，对应的底层的 oop 对象 this
    	// GetObjectField 应该是获得 this 对象的 fd 字段
    	// SetLongField 设置 fd(FileDescriptor) 所以对应的对象的
    	// 字段：
    	// 		private int fd;
    	//		private long handle;
    	// fid 表示 this 对象的 fd 字段的id,
    	// IO_handle_fdID 表示 
    	//		java.io.FileDescriptor.handle 的 fieldId
    	// 所以 GetObjectField 返回 this 对象的 fd 对象。
    	// 所以下面的 SetLongField 调用将 创建的文件 句柄 h 设置到
    	// FileDescriptor 对象的 handle 字段中。
    	// 在 win32 的实现中将 创建好的 文件句柄 设置到 handle 字段
		// 在 linux 版本中则使用的是 FileDescriptor 的 fd 字段
		// 由此，可知 handle 和 fd 是共存的但并不同时在使用，
		// 在 win32 平台上使用 handle 字段
		// 在 linux 平台上使用 fd 字段
        if ((*env)->GetObjectField(env, (this), (fid)) != NULL)
        (*env)->SetLongField(env, (*env)->GetObjectField(env, (this), (fid)), IO_handle_fdID, (h))
    }
}
```

FileInputStream 类加载的会执行下面的代码：

``` java
// java.io.FileInputStream
/* File Descriptor - handle to the open file */
private final FileDescriptor fd;

private static native void initIDs();

static {
    initIDs();
}
```

initIDs的实现：

``` c++
// share/native/java/io/FileInputStream.c

jfieldID fis_fd; /* id for jobject 'fd' in java.io.FileInputStream */

/**************************************************************
 * static methods to store field ID's in initializers
 */
JNIEXPORT void JNICALL
Java_java_io_FileInputStream_initIDs(JNIEnv *env, jclass fdClass) {
    fis_fd = (*env)->GetFieldID(env, fdClass, "fd", "Ljava/io/FileDescriptor;");
}
```

initIDs 在 FileInputStream 被加载的时候，将其字段 fd 的 FieldID进行初始化，存储到全局变量 fis_fd 中。

同样的得到，fd 对象之后要操作，其字段：
``` java
// java.io.FileDescriptor
private int fd;

private long handle;

/* This routine initializes JNI field offsets for the class */
private static native void initIDs();

static {
    initIDs();
}
```

对应的 native 实现:

``` c++
/* field id for jint 'fd' in java.io.FileDescriptor */
jfieldID IO_fd_fdID;

/* field id for jlong 'handle' in java.io.FileDescriptor */
jfieldID IO_handle_fdID;

/**************************************************************
 * static methods to store field IDs in initializers
 */

JNIEXPORT void JNICALL
Java_java_io_FileDescriptor_initIDs(JNIEnv *env, jclass fdClass) {
    IO_fd_fdID = (*env)->GetFieldID(env, fdClass, "fd", "I");
    IO_handle_fdID = (*env)->GetFieldID(env, fdClass, "handle", "J");
}
```

### 打开文件的流程总结

``` java
FileInputStream fis = new FileInputStream("1.txt");
```

这个调用的过程如下：

1. 创建 FileDescriptor 对象

	每一个 FileInputStream 有一个 FileDescriptor，代表这个流底层的文件的handle

2. 调用 native 方法 open, 打开文件

	内部调用 CreateFile 打开文件，返回文件句柄 handle
	
3. 初始化 FileDescriptor 对象

	将 文件句柄 handle 设置到，FileDescriptor 对象的 handle 中

## read 的实现

其实现在 FileInputStream.c 和

src/windows/native/java/io/Io_util_md.h 中

``` c++
JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_read(JNIEnv *env, jobject this) {
	// fis_fd ： java.io.FileInputStream.fd 的 fieldId
    return readSingle(env, this, fis_fd);
}

jint
readSingle(JNIEnv *env, jobject this, jfieldID fid) {
    jint nread;
    char ret;
    // 获得 fd
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return -1;
    }
    // fd 文件句柄，
    // ret 读取到数据
    // 读取数据的数量
    nread = (jint)IO_Read(fd, &ret, 1);
    if (nread == 0) { /* EOF */
        return -1;
    } else if (nread == JVM_IO_ERR) { /* error */
        JNU_ThrowIOExceptionWithLastError(env, "Read error");
    } else if (nread == JVM_IO_INTR) {
        JNU_ThrowByName(env, "java/io/InterruptedIOException", NULL);
    }
    return ret & 0xFF;
}

/*
 * Route the routines away from VM
 */
#define IO_Append handleAppend
#define IO_Write handleWrite
#define IO_Sync handleSync
#define IO_Read handleRead
#define IO_Lseek handleLseek
#define IO_Available handleAvailable
#define IO_SetLength handleSetLength

// src/windows/native/java/io/Io_util_md.c
// IO_Read(fd, &ret, 1);
JNIEXPORT
size_t
handleRead(jlong fd, void *buf, jint len)
{
    DWORD read = 0;
    BOOL result = 0;
    HANDLE h = (HANDLE)fd;
    if (h == INVALID_HANDLE_VALUE) {
        return -1;
    }
    // h 文件句柄，len = 1
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
    return read;
}
```

ReadFile win32 api 用来读取文件。

### 文件的读取模型

在windows 和 linux 中都提供了读取文件的操作

``` c++
// win32 API
BOOL WINAPI ReadFile(
  _In_        HANDLE       hFile,
  _Out_       LPVOID       lpBuffer,
  _In_        DWORD        nNumberOfBytesToRead,
  _Out_opt_   LPDWORD      lpNumberOfBytesRead,
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
DWORD WINAPI SetFilePointer(
  _In_        HANDLE hFile,
  _In_        LONG   lDistanceToMove,
  _Inout_opt_ PLONG  lpDistanceToMoveHigh,
  _In_        DWORD  dwMoveMethod
);

// linux API
ssize_t read(int fd, void *buf, size_t count);
off_t lseek(int fildes, off_t offset, int whence);
```

对于这两个API，都没有提供读取的位置，那么当第一次调用的时候，应该从文件的什么位置去读取数据呢？

* linux

> open & read & lseek:
>
> The file offset is set to the beginning of the file
>
> On success, the number of bytes read is returned (zero indicates end of file), and the file position is advanced by this number.

* win32

> ReadFile & SetFilePointer
> 
>If hFile is opened with FILE_FLAG_OVERLAPPED, it is an asynchronous file handle; otherwise it is synchronous.
>
>File pointer position for a synchronous handle is maintained by the system as data is read or written and can also be updated using the SetFilePointer or SetFilePointerEx function.
>
>Reads data from the specified file or input/output (I/O) device. Reads occur at the position specified by the file pointer if supported by the device.
>
>If lpOverlapped is NULL, the read operation starts at the current file position and ReadFile does not return until the operation is complete, and the system updates the file pointer before ReadFile returns.
>
>When an application calls CreateFile to open a file for the first time, Windows places the file pointer at the beginning of the file. As bytes are read from or written to the file, Windows advances the file pointer the number of bytes read or written.

[File Pointers](https://msdn.microsoft.com/zh-cn/library/windows/desktop/aa364397(v=vs.85).aspx)

[Positioning a File Pointer](https://msdn.microsoft.com/zh-cn/library/windows/desktop/aa365457(v=vs.85).aspx)

msdn: System Services > File Services > File Systems > File System Components > Files and Clusters

可以看到当使用 CreateFile 和 open 打开一个文件的时候，系统会将文件的 offset 放置到文件的开头。offset = 0, 后续的 ReadFile 和 read 的时候，offset 会被移动到下一次读取的文件的位置。所以，读取的位置 offset 是由系统来维护的。

但是，可以使用 SetFilePointer 和 lseek 来设置文件的 offset. 然后，下次的读取将使用新的 offset 来，读取数据。

## FileInputStream 关闭

``` c++
// jdk\src\windows\native\java\io\FileInputStream_md.c
JNIEXPORT void JNICALL
Java_java_io_FileInputStream_close0(JNIEnv *env, jobject this) {
    handleClose(env, this, fis_fd);
}


// jdk\src\windows\native\java\io\io_util_md.c
jint handleClose(JNIEnv *env, jobject this, jfieldID fid)
{
    // 获得 FileInputStream.fd.handle 对象
    // 其存储，当前文件的句柄
    FD fd = GET_FD(this, fid);
    HANDLE h = (HANDLE)fd;

    if (h == INVALID_HANDLE_VALUE) {
        return 0;
    }

    /* Set the fd to -1 before closing it so that the timing window
     * of other threads using the wrong fd (closed but recycled fd,
     * that gets re-opened with some other filename) is reduced.
     * Practically the chance of its occurance is low, however, we are
     * taking extra precaution over here.
     */
	// 将 FileInputStream.fd.handle 设置成 -1,
	// 表示 fd 已经无效，如果后续调用 在当前文件流上 继续调用 read 方法
	// 将导致 IOException "Stream Closed"
    SET_FD(this, -1, fid);

	// 使用 win32 API CloseHandle 来关闭文件
	// CloseHandle invalidates the specified object handle, decrements the object's handle count, 
	// and performs object retention checks. After the last handle to an object is closed, 
	// the object is removed from the system.
	// CloseHandle 最终将释放 打开文件时所创建的内核对象。
    if (CloseHandle(h) == 0) { /* Returns zero on failure */
        JNU_ThrowIOExceptionWithLastError(env, "close failed");
    }
    return 0;
}
```

## skip & available

### skip

``` c++
JNIEXPORT jlong JNICALL
Java_java_io_FileInputStream_skip(JNIEnv *env, jobject this, jlong toSkip) {
    jlong cur = jlong_zero;
    jlong end = jlong_zero;
    FD fd = GET_FD(this, fis_fd);
    if (fd == -1) {
        JNU_ThrowIOException (env, "Stream Closed");
        return 0;
    }
    if ((cur = IO_Lseek(fd, (jlong)0, (jint)SEEK_CUR)) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Seek error");
    } else if ((end = IO_Lseek(fd, toSkip, (jint)SEEK_CUR)) == -1) {
        JNU_ThrowIOExceptionWithLastError(env, "Seek error");
    }
    return (end - cur);
}

#define IO_Lseek handleLseek

jlong
handleLseek(jlong fd, jlong offset, jint whence)
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
    if (SetFilePointerEx(h, distance, &pos, op) == 0) {
        return -1;
    }
    return long_to_jlong(pos.QuadPart);
}

BOOL SetFilePointerEx(
  HANDLE hFile,
  LARGE_INTEGER liDistanceToMove,
  PLARGE_INTEGER lpNewFilePointer,
  DWORD dwMoveMethod
);
```

SetFilePointerEx win32 API:

The SetFilePointerEx function moves the file pointer of the specified file.

### available

``` c++
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
        }
        return jlong_to_jint(ret);
    }
    JNU_ThrowIOExceptionWithLastError(env, NULL);
    return 0;
}

#define IO_Available handleAvailable

int
handleAvailable(jlong fd, jlong *pbytes) {
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

* GetFileType

	``` c++
	DWORD GetFileType(
	  HANDLE hFile
	);
	```

	The GetFileType function retrieves the file type of the specified file.

	FILE_TYPE_DISK: The specified file is a disk file.

* GetFileSizeEx

	``` c++
	BOOL GetFileSizeEx(
	  HANDLE hFile,
	  PLARGE_INTEGER lpFileSize
	);
	```

	The GetFileSizeEx function retrieves the size of the specified file.

## FileOutputStream

* write

``` c++
// src/share/native/java/io/io_util.c

void
writeSingle(JNIEnv *env, jobject this, jint byte, jboolean append, jfieldID fid) {
    // Discard the 24 high-order bits of byte. See OutputStream#write(int)
    char c = (char) byte;
    jint n;
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return;
    }
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

// src/windows/native/java/io/io_util_md.h
#define IO_Append handleAppend
#define IO_Write handleWrite

// src/windows/native/java/io/io_util_md.c
JNIEXPORT
size_t handleWrite(jlong fd, const void *buf, jint len) {
    return writeInternal(fd, buf, len, JNI_FALSE);
}

JNIEXPORT
size_t handleAppend(jlong fd, const void *buf, jint len) {
    return writeInternal(fd, buf, len, JNI_TRUE);
}

static size_t writeInternal(jlong fd, const void *buf, jint len, jboolean append)
{
    BOOL result = 0;
    DWORD written = 0;
	// 获得文件句柄
    HANDLE h = (HANDLE)fd;
    if (h != INVALID_HANDLE_VALUE) {
        OVERLAPPED ov;
        LPOVERLAPPED lpOv;
        if (append == JNI_TRUE) {
            ov.Offset = (DWORD)0xFFFFFFFF;
            ov.OffsetHigh = (DWORD)0xFFFFFFFF;
            ov.hEvent = NULL;
            lpOv = &ov;
        } else {
            lpOv = NULL;
        }
        result = WriteFile(h,                /* File handle to write */
                           buf,              /* pointers to the buffers */
                           len,              /* number of bytes to write */
                           &written,         /* receives number of bytes written */
                           lpOv);            /* overlapped struct */
    }
    if ((h == INVALID_HANDLE_VALUE) || (result == 0)) {
        return -1;
    }
    return (size_t)written;
}
```

在 win32 平台下调用 WriteFile 来写入文件。

``` c++
BOOL WriteFile(
  HANDLE hFile,
  LPCVOID lpBuffer,
  DWORD nNumberOfBytesToWrite,
  LPDWORD lpNumberOfBytesWritten,
  LPOVERLAPPED lpOverlapped
);
```

>If hFile was not opened with FILE_FLAG_OVERLAPPED and lpOverlapped is NULL, the write operation starts at the current file position and WriteFile does not return until the operation has been completed. The system updates the file pointer upon completion.
>
>If hFile was not opened with FILE_FLAG_OVERLAPPED and lpOverlapped is not NULL, the write operation starts at the offset specified in the OVERLAPPED structure and WriteFile does not return until the write operation has been completed. The system updates the file pointer upon completion.

文件在 open 的时候，并没有使用 FILE_FLAG_OVERLAPPED 标志，所以在调用 WriteFile 的时候，遵循上面的行为。同时，由上面的描述可知，这种写入是 block 的，直到操作完成，才返回。同时，系统也会维护文件指针，使其指向下次写入的位置。

FileOutputStream 的构造函数，提供一个 append 参数，用来表示，文件的写入是从头开始（相当于覆盖），还是从文件的最后 apped 数据。这个参数，默认是 false, 也就是文件默认是覆盖写入的。

``` java
FileOutputStream(String name, boolean append) 
```