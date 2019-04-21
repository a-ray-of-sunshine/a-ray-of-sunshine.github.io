---
title: FileChannel
date: 2016-10-17 15:31:25
---

## FileChannel

FileChannel对象的获得：

* FileInputStream.getChannel
* FileOutputStream.getChannel
* RandomAccessFile.getChannel
* FileChannel.open
* Files.newByteChannel

### FileChannelImpl

openjdk\jdk\src\share\classes\sun\nio\ch\FileChannelImpl.java

``` java

private FileChannelImpl(FileDescriptor fd, boolean readable,
                        boolean writable, boolean append, Object parent)
{
    this.fd = fd;
    this.readable = readable;
    this.writable = writable;
    this.append = append;
    this.parent = parent;
    this.nd = new FileDispatcherImpl(append);
}

// Used by FileInputStream.getChannel() and RandomAccessFile.getChannel()
public static FileChannel open(FileDescriptor fd,
                               boolean readable, boolean writable,
                               Object parent)
{
    return new FileChannelImpl(fd, readable, writable, false, parent);
}

// Used by FileOutputStream.getChannel
public static FileChannel open(FileDescriptor fd,
                               boolean readable, boolean writable,
                               boolean append, Object parent)
{
    return new FileChannelImpl(fd, readable, writable, append, parent);
}
```
	
### FileDispatcherImpl

不同的操作系统有不同的实现。

* windows

	openjdk\jdk\src\windows\classes\sun\nio\ch\FileDispatcherImpl.java
	
	openjdk\jdk\src\windows\native\sun\nio\ch\FileDispatcherImpl.c

* linux

	openjdk\jdk\src\solaris\classes\sun\nio\ch\FileDispatcherImpl.java
	
	openjdk\jdk\src\solaris\native\sun\nio\ch\FileDispatcherImpl.c


``` java
// FileChannelImpl
public int read(ByteBuffer dst) throws IOException {
    ensureOpen();
    if (!readable)
        throw new NonReadableChannelException();
    synchronized (positionLock) {
        int n = 0;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return 0;
            do {
                n = IOUtil.read(fd, dst, -1, nd, positionLock);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            return IOStatus.normalize(n);
        } finally {
            threads.remove(ti);
            end(n > 0);
            assert IOStatus.check(n);
        }
    }
}

// IOUtil.read
static int read(FileDescriptor fd, ByteBuffer dst, long position,
                NativeDispatcher nd, Object lock)
    throws IOException
{
    if (dst.isReadOnly())
        throw new IllegalArgumentException("Read-only buffer");
    if (dst instanceof DirectBuffer)
		// 如果Buffer是DirectBuffer，则调用下面的进行
        return readIntoNativeBuffer(fd, dst, position, nd, lock);

	// 如果不是 DirectBuffer， 则临时创建一个。
    // Substitute a native buffer
    ByteBuffer bb = Util.getTemporaryDirectBuffer(dst.remaining());
    try {
        int n = readIntoNativeBuffer(fd, bb, position, nd, lock);
		// bb 中存储数据
		// 读取到 dst 中。
        bb.flip();
        if (n > 0)
            dst.put(bb);
        return n;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(bb);
    }
}

private static int readIntoNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                        long position, NativeDispatcher nd, Object lock)
    throws IOException
{
    int pos = bb.position();
    int lim = bb.limit();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

	// 注意，如果 Buffer 的剩余空间不够了，直接返回0
	// 而不会抛出异常。
    if (rem == 0)
        return 0;
    int n = 0;
    if (position != -1) {
        n = nd.pread(fd, ((DirectBuffer)bb).address() + pos,
                     rem, position, lock);
    } else {
		// 进入到这个分支
		// 调用 NativeDispatcher 的 read 方法读取数据。
		// 这是个抽象类，由不同的平台提供实现。
        n = nd.read(fd, ((DirectBuffer)bb).address() + pos, rem);
    }
    if (n > 0)
        bb.position(pos + n);
    return n;
}

/**
 * Allows different platforms to call different native methods
 * for read and write operations.
 */
// abstract class FileDispatcher extends NativeDispatcher
// FileDispatcher 不同的平台提供不同的实现 子类 FileDispatcherImpl
// sun\nio\ch\FileDispatcherImpl.java

// openjdk\jdk\src\windows\classes\sun\nio\ch\FileDispatcherImpl.java
int read(FileDescriptor fd, long address, int len)
    throws IOException
{
    return read0(fd, address, len);
}

// openjdk\jdk\src\windows\native\sun\nio\ch\FileDispatcherImpl.c
// 这个文件是实现整个FileChannel的核心，FileChannel中与操作系统相关的
// 所以操作全部都在这个文件中实现。
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_read0(JNIEnv *env, jclass clazz, jobject fdo,
                                      jlong address, jint len)
{
    DWORD read = 0;
    BOOL result = 0;
    HANDLE h = (HANDLE)(handleval(env, fdo));

    if (h == INVALID_HANDLE_VALUE) {
        JNU_ThrowIOExceptionWithLastError(env, "Invalid handle");
        return IOS_THROWN;
    }
	// 调用 win32 API ReadFile 读取数据
	// result 返回非0值，表示读取成功。
    result = ReadFile(h,          /* File handle to read */
                      (LPVOID)address,    /* address to put data */
                      len,        /* number of bytes to read */
                      &read,      /* number of bytes read */
                      NULL);      /* no overlapped struct */
    if (result == 0) {
        int error = GetLastError();
        if (error == ERROR_BROKEN_PIPE) {
            return IOS_EOF;
        }
        if (error == ERROR_NO_DATA) {
            return IOS_UNAVAILABLE;
        }
        JNU_ThrowIOExceptionWithLastError(env, "Read failed");
        return IOS_THROWN;
    }
    return convertReturnVal(env, (jint)read, JNI_TRUE);
}


// FileDispatcherImpl 的 linux 实现：
// openjdk\jdk\src\solaris\classes\sun\nio\ch\FileDispatcherImpl.java
int read(FileDescriptor fd, long address, int len) throws IOException {
    return read0(fd, address, len);
}
// openjdk\jdk\src\solaris\native\sun\nio\ch\FileDispatcherImpl.c
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_read0(JNIEnv *env, jclass clazz,
                             jobject fdo, jlong address, jint len)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);
	// 调用 linux 系统调用 read 
	// read() attempts to read up to count bytes 
	// from file descriptor fd into the buffer starting at buf.
	// On success, the number of bytes read is returned
    return convertReturnVal(env, read(fd, buf, len), JNI_TRUE);
}
```

#### FileDispatcherImpl 实现的写操作

``` c
// windows
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_write0(JNIEnv *env, jclass clazz, jobject fdo,
                                          jlong address, jint len, jboolean append)
{
    BOOL result = 0;
    DWORD written = 0;
    HANDLE h = (HANDLE)(handleval(env, fdo));

    if (h != INVALID_HANDLE_VALUE) {
        OVERLAPPED ov;
        LPOVERLAPPED lpOv;
        // append 为 true,表示从文件最后面进行写入
        if (append == JNI_TRUE) {
        	// To write to the end of file, specify both the 
        	// Offset and OffsetHigh members of the OVERLAPPED structure as 0xFFFFFFFF. 
            ov.Offset = (DWORD)0xFFFFFFFF;
            ov.OffsetHigh = (DWORD)0xFFFFFFFF;
            ov.hEvent = NULL;
            lpOv = &ov;
        } else {
        	// 写入到当前文件指针的位置。
            lpOv = NULL;
        }
        result = WriteFile(h,           /* File handle to write */
                      (LPCVOID)address, /* pointers to the buffers */
                      len,              /* number of bytes to write */
                      &written,         /* receives number of bytes written */
                      lpOv);            /* overlapped struct */
    }

    if ((h == INVALID_HANDLE_VALUE) || (result == 0)) {
        JNU_ThrowIOExceptionWithLastError(env, "Write failed");
    }

    return convertReturnVal(env, (jint)written, JNI_FALSE);
}


// linux
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_write0(JNIEnv *env, jclass clazz,
                              jobject fdo, jlong address, jint len)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);

	// write() writes up to count bytes from the buffer pointed buf
	// to the file referred to by the file descriptor fd.
	// On success, the number of bytes written is returned
    return convertReturnVal(env, write(fd, buf, len), JNI_FALSE);
}
```

###	Channel的关闭

``` java
// FileChannelImpl.java
protected void implCloseChannel() throws IOException {

	// 释放 channel 持有的锁
    // Release and invalidate any locks that we still hold
    if (fileLockTable != null) {
        for (FileLock fl: fileLockTable.removeAll()) {
            synchronized (fl) {
                if (fl.isValid()) {
                    nd.release(fd, fl.position(), fl.size());
                    ((FileLockImpl)fl).invalidate();
                }
            }
        }
    }

    nd.preClose(fd);
    threads.signalAndWait();

    if (parent != null) {
		// 如果 channel 是由 FileInputStream.getChannel 获得的
		// 则 parent 就是 FileInputStream，所以 fd 的关闭操作
		// 应该委托给 parent 进行。
        // Close the fd via the parent stream's close method.  The parent
        // will reinvoke our close method, which is defined in the
        // superclass AbstractInterruptibleChannel, but the isOpen logic in
        // that method will prevent this method from being reinvoked.
        //
        ((java.io.Closeable)parent).close();
    } else {
    	// 调用 FileDispatcherImpl 关闭 fd.
        nd.close(fd);
    }

}
```

FileDispatcherImpl 关闭 fd 的实现：

``` c
// windows
static void closeFile(JNIEnv *env, jlong fd) {
    HANDLE h = (HANDLE)fd;
    if (h != INVALID_HANDLE_VALUE) {
    	// Closes an open object handle.
        int result = CloseHandle(h);
        if (result < 0)
            JNU_ThrowIOExceptionWithLastError(env, "Close failed");
    }
}
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_close0(JNIEnv *env, jclass clazz, jobject fdo)
{
    jlong fd = handleval(env, fdo);
    closeFile(env, fd);
}

// linux
static void closeFileDescriptor(JNIEnv *env, int fd) {
    if (fd != -1) {
    	// close() closes a file descriptor, so that it no longer 
    	// refers to any file and may be reused.
        int result = close(fd);
        if (result < 0)
            JNU_ThrowIOExceptionWithLastError(env, "Close failed");
    }
}
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_close0(JNIEnv *env, jclass clazz, jobject fdo)
{
    jint fd = fdval(env, fdo);
    closeFileDescriptor(env, fd);
}
```

### lock 方法

* public abstract FileLock lock(long position, long size, boolean shared)
* public final FileLock lock()
* public abstract FileLock tryLock(long position, long size, boolean shared)
* public final FileLock tryLock()

> Acquires a lock on the given region of this channel's file. 
	
``` c
// FileChannelImpl
public FileLock lock(long position, long size, boolean shared)
    throws IOException
{
    ensureOpen();
    if (shared && !readable)
        throw new NonReadableChannelException();
    if (!shared && !writable)
        throw new NonWritableChannelException();
    FileLockImpl fli = new FileLockImpl(this, position, size, shared);
    FileLockTable flt = fileLockTable();
    flt.add(fli);
    boolean completed = false;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return null;
        int n;
        do {
        	// 以 block 形式不断尝试获取锁
            n = nd.lock(fd, true, position, size, shared);
        } while ((n == FileDispatcher.INTERRUPTED) && isOpen());
        if (isOpen()) {
            if (n == FileDispatcher.RET_EX_LOCK) {
                assert shared;
                // 锁获得成功，创建 FileLock 对象。
                FileLockImpl fli2 = new FileLockImpl(this, position, size,
                                                     false);
                flt.replace(fli, fli2);
                fli = fli2;
            }
            completed = true;
        }
    } finally {
        if (!completed)
            flt.remove(fli);
        threads.remove(ti);
        try {
            end(completed);
        } catch (ClosedByInterruptException e) {
            throw new FileLockInterruptionException();
        }
    }
    return fli;
}

// windows
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_lock0(JNIEnv *env, jobject this, jobject fdo,
                                      jboolean block, jlong pos, jlong size,
                                      jboolean shared)
{
    HANDLE h = (HANDLE)(handleval(env, fdo));
    DWORD lowPos = (DWORD)pos;
    long highPos = (long)(pos >> 32);
    DWORD lowNumBytes = (DWORD)size;
    DWORD highNumBytes = (DWORD)(size >> 32);
    BOOL result;
    DWORD flags = 0;
    OVERLAPPED o;
    o.hEvent = 0;
    o.Offset = lowPos;
    o.OffsetHigh = highPos;
    if (block == JNI_FALSE) {
        flags |= LOCKFILE_FAIL_IMMEDIATELY;
    }
    if (shared == JNI_FALSE) {
        flags |= LOCKFILE_EXCLUSIVE_LOCK;
    }
    // Locks the specified file for exclusive access by the calling process. 
    result = LockFileEx(h, flags, 0, lowNumBytes, highNumBytes, &o);
    if (result == 0) {
        int error = GetLastError();
        if (error != ERROR_LOCK_VIOLATION) {
            JNU_ThrowIOExceptionWithLastError(env, "Lock failed");
            return sun_nio_ch_FileDispatcherImpl_NO_LOCK;
        }
        if (flags & LOCKFILE_FAIL_IMMEDIATELY) {
            return sun_nio_ch_FileDispatcherImpl_NO_LOCK;
        }
        JNU_ThrowIOExceptionWithLastError(env, "Lock failed");
        return sun_nio_ch_FileDispatcherImpl_NO_LOCK;
    }
    return sun_nio_ch_FileDispatcherImpl_LOCKED;
}

// linux
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_lock0(JNIEnv *env, jobject this, jobject fdo,
                                      jboolean block, jlong pos, jlong size,
                                      jboolean shared)
{
    jint fd = fdval(env, fdo);
    jint lockResult = 0;
    int cmd = 0;
    struct flock64 fl;

	// Specifies the starting offset of the lock 
	// segment in the file. Valid settings are:
	// 0. l_whence 参数表明：偏移量 l_start 相对于文件的起始位置。
    fl.l_whence = SEEK_SET;
    
    // 1. l_len 表明：被锁segment的长度。
    if (size == (jlong)java_lang_Long_MAX_VALUE) {
        fl.l_len = (off64_t)0;
    } else {
        fl.l_len = (off64_t)size;
    }
    
    // 2. l_start 被锁方法的偏移量
    fl.l_start = (off64_t)pos;
    
    // 3. l_type 表示操作类型
    // 如果 l_type 是 F_UNLCK，表示：Remove a lock. 释放锁。 
    if (shared == JNI_TRUE) {
    	// F_RDLCK: Create a shared lock.
        fl.l_type = F_RDLCK;
    } else {
    	// F_WRLCK: Create an exclusive lock.
        fl.l_type = F_WRLCK;
    }
    
    // 要执行的命令
    // Set or clear a file segment lock for the file 
    // to which the specified file descriptor refers.
    if (block == JNI_TRUE) {
    	// 和 F_SETLK64 功能相同，不过，它是block的：
    	// if a shared or exclusive lock is blocked by other locks, 
    	// the thread waits until the request can be satisfied.
    	// 阻塞到获得锁，或者线程被中断
        cmd = F_SETLKW64;
    } else {
    	// If the lock cannot be immediately obtained, 
    	// fcntl() returns -1 with errno set to EACCES.
    	// 如果不能获得锁，则立即返回-1
        cmd = F_SETLK64;
    }
    
    lockResult = fcntl(fd, cmd, &fl);
    if (lockResult < 0) {
        if ((cmd == F_SETLK64) && (errno == EAGAIN))
            return sun_nio_ch_FileDispatcherImpl_NO_LOCK;
        if (errno == EINTR)
            return sun_nio_ch_FileDispatcherImpl_INTERRUPTED;
        JNU_ThrowIOExceptionWithLastError(env, "Lock failed");
    }
    return 0;
}

```

通过 lock 方法获得锁成功之后，会创建一个 FileLock 对象来表示这个锁:

`FileLockImpl fli2 = new FileLockImpl(this, position, size, false);`

使用这个对象可以用来释放锁。

``` java
// FileLockImpl.release
public synchronized void release() throws IOException {
    Channel ch = acquiredBy();
    if (!ch.isOpen())
        throw new ClosedChannelException();
    if (valid) {
        if (ch instanceof FileChannelImpl)
            ((FileChannelImpl)ch).release(this);
        else if (ch instanceof AsynchronousFileChannelImpl)
            ((AsynchronousFileChannelImpl)ch).release(this);
        else throw new AssertionError();
        valid = false;
    }
}

// FileChannelImpl.release
void release(FileLockImpl fli) throws IOException {
    int ti = threads.add();
    try {
        ensureOpen();
        nd.release(fd, fli.position(), fli.size());
    } finally {
        threads.remove(ti);
    }
    assert fileLockTable != null;
    fileLockTable.remove(fli);
}

// windows
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_release0(JNIEnv *env, jobject this, jobject fdo, jlong pos, jlong size)
{
    HANDLE h = (HANDLE)(handleval(env, fdo));
    DWORD lowPos = (DWORD)pos;
    long highPos = (long)(pos >> 32);
    DWORD lowNumBytes = (DWORD)size;
    DWORD highNumBytes = (DWORD)(size >> 32);
    jint result = 0;
    OVERLAPPED o;
    o.hEvent = 0;
    o.Offset = lowPos;
    o.OffsetHigh = highPos;
    // Unlocks a region in the specified file.
    result = UnlockFileEx(h, 0, lowNumBytes, highNumBytes, &o);
    if (result == 0 && GetLastError() != ERROR_NOT_LOCKED) {
        JNU_ThrowIOExceptionWithLastError(env, "Release failed");
    }
}

// linux
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_release0(JNIEnv *env, jobject this,
                                         jobject fdo, jlong pos, jlong size)
{
    jint fd = fdval(env, fdo);
    jint lockResult = 0;
    struct flock64 fl;
    int cmd = F_SETLK64;

    fl.l_whence = SEEK_SET;
    if (size == (jlong)java_lang_Long_MAX_VALUE) {
        fl.l_len = (off64_t)0;
    } else {
        fl.l_len = (off64_t)size;
    }
    fl.l_start = (off64_t)pos;
    // F_UNLCK: Remove a lock.
    fl.l_type = F_UNLCK;
    lockResult = fcntl(fd, cmd, &fl);
    if (lockResult < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "Release failed");
    }
}
```

### map 内存映射文件

``` java
public MappedByteBuffer map(MapMode mode, long position, long size)
        throws IOException
{
    ensureOpen();
    if (position < 0L)
        throw new IllegalArgumentException("Negative position");
    if (size < 0L)
        throw new IllegalArgumentException("Negative size");
    if (position + size < 0)
        throw new IllegalArgumentException("Position + size overflow");
    if (size > Integer.MAX_VALUE)
        throw new IllegalArgumentException("Size exceeds Integer.MAX_VALUE");
    int imode = -1;
    if (mode == MapMode.READ_ONLY)
        imode = MAP_RO;
    else if (mode == MapMode.READ_WRITE)
        imode = MAP_RW;
    else if (mode == MapMode.PRIVATE)
        imode = MAP_PV;
    assert (imode >= 0);
    if ((mode != MapMode.READ_ONLY) && !writable)
        throw new NonWritableChannelException();
    if (!readable)
        throw new NonReadableChannelException();

    long addr = -1;
    int ti = -1;
    try {
        begin();
        ti = threads.add();
        if (!isOpen())
            return null;
        if (size() < position + size) { // Extend file size
            if (!writable) {
                throw new IOException("Channel not open for writing " +
                    "- cannot extend file to required size");
            }
            int rv;
            do {
                rv = nd.truncate(fd, position + size);
            } while ((rv == IOStatus.INTERRUPTED) && isOpen());
        }
        if (size == 0) {
            addr = 0;
            // a valid file descriptor is not required
            FileDescriptor dummy = new FileDescriptor();
            if ((!writable) || (imode == MAP_RO))
                return Util.newMappedByteBufferR(0, 0, dummy, null);
            else
                return Util.newMappedByteBuffer(0, 0, dummy, null);
        }

        int pagePosition = (int)(position % allocationGranularity);
        long mapPosition = position - pagePosition;
        long mapSize = size + pagePosition;
        try {
            // If no exception was thrown from map0, the address is valid
            // 调用native方法进行内存映射
            addr = map0(imode, mapPosition, mapSize);
        } catch (OutOfMemoryError x) {
            // An OutOfMemoryError may indicate that we've exhausted memory
            // so force gc and re-attempt map
            System.gc();
            try {
                Thread.sleep(100);
            } catch (InterruptedException y) {
                Thread.currentThread().interrupt();
            }
            try {
                addr = map0(imode, mapPosition, mapSize);
            } catch (OutOfMemoryError y) {
                // After a second OOME, fail
                throw new IOException("Map failed", y);
            }
        }

        // On Windows, and potentially other platforms, we need an open
        // file descriptor for some mapping operations.
        FileDescriptor mfd;
        try {
            mfd = nd.duplicateForMapping(fd);
        } catch (IOException ioe) {
            unmap0(addr, mapSize);
            throw ioe;
        }

        assert (IOStatus.checkAll(addr));
        assert (addr % allocationGranularity == 0);
        int isize = (int)size;
        Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
        // 映射成功，将 addr 作为 MappedByteBuffer 的基址
        if ((!writable) || (imode == MAP_RO)) {
        // 相当于： return new DirectByteBufferR(isize,addr + pagePosition,mfd,um)
            return Util.newMappedByteBufferR(isize,
                                             addr + pagePosition,
                                             mfd,
                                             um);
        } else {
        // 相当于 return new DirectByteBuffer(isize,addr + pagePosition,mfd,um);
            return Util.newMappedByteBuffer(isize,
                                            addr + pagePosition,
                                            mfd,
                                            um);
        }
    } finally {
        threads.remove(ti);
        end(IOStatus.checkAll(addr));
    }
}

// map0 的实现
// winodws
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                               jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jint lowOffset = (jint)off;
    jint highOffset = (jint)(off >> 32);
    jlong maxSize = off + len;
    jint lowLen = (jint)(maxSize);
    jint highLen = (jint)(maxSize >> 32);
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    HANDLE fileHandle = (HANDLE)(handleval(env, fdo));
    HANDLE mapping;
    DWORD mapAccess = FILE_MAP_READ;
    DWORD fileProtect = PAGE_READONLY;
    DWORD mapError;
    BOOL result;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        fileProtect = PAGE_READONLY;
        mapAccess = FILE_MAP_READ;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        fileProtect = PAGE_READWRITE;
        mapAccess = FILE_MAP_WRITE;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        fileProtect = PAGE_WRITECOPY;
        mapAccess = FILE_MAP_COPY;
    }

	// Creates or opens a named or unnamed file mapping object for a specified file.
    mapping = CreateFileMapping(
        fileHandle,      /* Handle of file */
        NULL,            /* Not inheritable */
        fileProtect,     /* Read and write */
        highLen,         /* High word of max size */
        lowLen,          /* Low word of max size */
        NULL);           /* No name for object */

    if (mapping == NULL) {
        JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }

	// Maps a view of a file mapping into the address space of a calling process.
    mapAddress = MapViewOfFile(
        mapping,             /* Handle of file mapping object */
        mapAccess,           /* Read and write access */
        highOffset,          /* High word of offset */
        lowOffset,           /* Low word of offset */
        (DWORD)len);         /* Number of bytes to map */
    mapError = GetLastError();

    result = CloseHandle(mapping);
    if (result == 0) {
        JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }

    if (mapAddress == NULL) {
        if (mapError == ERROR_NOT_ENOUGH_MEMORY)
            JNU_ThrowOutOfMemoryError(env, "Map failed");
        else
            JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }

    return ptr_to_jlong(mapAddress);
}

// linux
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                                     jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    jint fd = fdval(env, fdo);
    int protections = 0;
    int flags = 0;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        protections = PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        protections = PROT_WRITE | PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        protections =  PROT_WRITE | PROT_READ;
        flags = MAP_PRIVATE;
    }

	// mmap() creates a new mapping in the virtual address space of the calling process.
    mapAddress = mmap64(
        0,                    /* Let OS decide location */
        len,                  /* Number of bytes to map */
        protections,          /* File permissions */
        flags,                /* Changes are shared */
        fd,                   /* File descriptor of mapped file */
        off);                 /* Offset into file */

    if (mapAddress == MAP_FAILED) {
        if (errno == ENOMEM) {
            JNU_ThrowOutOfMemoryError(env, "Map failed");
            return IOS_THROWN;
        }
        return handle(env, -1, "Map failed");
    }

    return ((jlong) (unsigned long) mapAddress);
}

```

### 可中断机制

FileChannel在实现I/O操作，其基本形式如下：

``` java
public int Operator(...) throws IOException {
    synchronized (positionLock) {
        try {
            begin();
            // Operator...
        } finally {
            end(n > 0);
        }
    }
}
```

其中 begin 将会在在设置当前线程线程中的 blocker 对象，这个对象将会在 Thread.interrupt 对象中被调用：

``` java
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0(); // Just to set the interrupt flag
            // blocker 对象的 interrupt 方法
            // 会被调用
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```

blocker 对象的实现如下：

* open 设置为 false
* 调用 implCloseChannel 来清除资源，关闭 channel

在 end 方法中，会检测是否被中断或者是被关闭，相应地抛出异常。

所以可以看到，其实对于任何一个线程来说，都是可以中断的，但是，由于 channel 对注册了 blocker 对象，使得对于阻塞在 channel 上进行 IO 操作的线程，当执行异步关闭或者中断时 channel 会将其所占用的资源全部释放。**所以可以说 channel 是可以安全地进行异步关闭和中断。** 

``` java
// -- Interruption machinery --
private Interruptible interruptor;
private volatile Thread interrupted;

/** Set thread's blocker field. */
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {
        
        		// 线程被中断时会调用这个函数
                public void interrupt(Thread target) {
                    synchronized (closeLock) {
                    
                    	// 清除状态位
                        if (!open)
                            return;
                        open = false;
                        interrupted = target;
                        try {
        // 调用implCloseChannel清理资源
		AbstractInterruptibleChannel.this.implCloseChannel();
                        } catch (IOException x) { }
                    }
                }};
    }
    
    // 将上面的 interruptor 设置到当前线程的 blocker 对象上
    // private volatile Interruptible blocker;
    // 这个方法的最终实现就是调用 Thread.blockedOn 方法,类似于：
    // Thread.currentThread().blockedOn(interruptor);
    blockedOn(interruptor);
    
    // 其实下面的代码并不是必须的，因为上面已经将 interruptor 设置到
    // 当前线程 me 的 blocker 字段中，如果有其它线程调用了 me.interrupt
    // 方法使得线程被中断，则 interruptor.interrupt(me) 方法也会被调用
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me);
}

protected final void end(boolean completed)
    throws AsynchronousCloseException
{
	// 将当前线程的 blocker 清除
    blockedOn(null);
    
    // 如果线程被中断，则this.interrupted会被设置，那个中断当前线程的线程
    Thread interrupted = this.interrupted;
    if (interrupted != null && interrupted == Thread.currentThread()) {
    	// 当前线程自己中断了自己
        interrupted = null;
        throw new ClosedByInterruptException();
    }
    
    // 如果操作失败（!completed） 且 
    // 当前线程被其它线程中断（!open，代码执行到这里，open变成false的惟一的原因就是被其它线程中断或者是当前channel被其它线程关闭）
    // 则表明当前线程被异步中断了
    if (!completed && !open)
        throw new AsynchronousCloseException();
}
```

### 随机访问

* public abstract int read(ByteBuffer dst, long position)

* public abstract int write(ByteBuffer src, long position)

这两个方法可以直接从指定的 position 处读取和写入数据，它们不会改变 file pointer 的值。

其底层由下面的方法实现：

``` c
// linux
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_pwrite0(JNIEnv *env, jclass clazz, jobject fdo,
                            jlong address, jint len, jlong offset)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);
	// pwrite() writes up to count bytes from the buffer starting at 
	// buf to the file descriptor fd at offset offset. The file 
	// offset is not changed.
    return convertReturnVal(env, pwrite64(fd, buf, len, offset), JNI_FALSE);
}
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_pread0(JNIEnv *env, jclass clazz, jobject fdo,
                            jlong address, jint len, jlong offset)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);
	// pread() reads up to count bytes from file descriptor fd at 
	// offset offset (from the start of the file) into the buffer 
	// starting at buf. The file offset is not changed.
    return convertReturnVal(env, pread64(fd, buf, len, offset), JNI_TRUE);
}

// windows
// windows 并没有像 pwrite 和 pread 类方法，可以在不改变 file potioner 
//的情况下进行读写，所以使用 SetFilePointer 查询并保存操作之前的位置，
// 操作完成之后，将其恢复，从而实现相同的目的。
```

### FileChannel.open

实现过程：

1. 调用 CreateFile 获得文件句柄(或者open/openat获得文件描述符)
2. 创建 FileDescriptor 对象
3. 调用 FileChannelImpl.open 方法（使用上面的fd），创建 FileChannel

openjdk\jdk\src\windows\classes\sun\nio\fs\WindowsChannelFactory.java

由 WindowsChannelFactory.newFileChannel 实现

``` java

static FileChannel newFileChannel(String pathForWindows,
                                  String pathToCheck,
                                  Set<? extends OpenOption> options,
                                  long pSecurityDescriptor)
    throws WindowsException
{
    Flags flags = Flags.toFlags(options);

    // default is reading; append => writing
    if (!flags.read && !flags.write) {
        if (flags.append) {
            flags.write = true;
        } else {
            flags.read = true;
        }
    }

    // validation
    if (flags.read && flags.append)
        throw new IllegalArgumentException("READ + APPEND not allowed");
    if (flags.append && flags.truncateExisting)
        throw new IllegalArgumentException("APPEND + TRUNCATE_EXISTING not allowed");

	// 调用 open 获得文件描述符
    FileDescriptor fdObj = open(pathForWindows, pathToCheck, flags, pSecurityDescriptor);
    // 创建 FileChannelImpl 对象
    return FileChannelImpl.open(fdObj, flags.read, flags.write, flags.append, null);
}

/**
 * Opens file based on parameters and options, returning a FileDescriptor
 * encapsulating the handle to the open file.
 */
private static FileDescriptor open(String pathForWindows,
                                   String pathToCheck,
                                   Flags flags,
                                   long pSecurityDescriptor)
    throws WindowsException
{
    // set to true if file must be truncated after open
    boolean truncateAfterOpen = false;

    // map options
    int dwDesiredAccess = 0;
    if (flags.read)
        dwDesiredAccess |= GENERIC_READ;
    if (flags.write)
        dwDesiredAccess |= GENERIC_WRITE;

    int dwShareMode = 0;
    if (flags.shareRead)
        dwShareMode |= FILE_SHARE_READ;
    if (flags.shareWrite)
        dwShareMode |= FILE_SHARE_WRITE;
    if (flags.shareDelete)
        dwShareMode |= FILE_SHARE_DELETE;

    int dwFlagsAndAttributes = FILE_ATTRIBUTE_NORMAL;
    int dwCreationDisposition = OPEN_EXISTING;
    if (flags.write) {
        if (flags.createNew) {
            dwCreationDisposition = CREATE_NEW;
            // force create to fail if file is orphaned reparse point
            dwFlagsAndAttributes |= FILE_FLAG_OPEN_REPARSE_POINT;
        } else {
            if (flags.create)
                dwCreationDisposition = OPEN_ALWAYS;
            if (flags.truncateExisting) {
                // Windows doesn't have a creation disposition that exactly
                // corresponds to CREATE + TRUNCATE_EXISTING so we use
                // the OPEN_ALWAYS mode and then truncate the file.
                if (dwCreationDisposition == OPEN_ALWAYS) {
                    truncateAfterOpen = true;
                } else {
                    dwCreationDisposition = TRUNCATE_EXISTING;
                }
            }
        }
    }

    if (flags.dsync || flags.sync)
        dwFlagsAndAttributes |= FILE_FLAG_WRITE_THROUGH;
    if (flags.overlapped)
        dwFlagsAndAttributes |= FILE_FLAG_OVERLAPPED;
    if (flags.deleteOnClose)
        dwFlagsAndAttributes |= FILE_FLAG_DELETE_ON_CLOSE;

    // NOFOLLOW_LINKS and NOFOLLOW_REPARSEPOINT mean open reparse point
    boolean okayToFollowLinks = true;
    if (dwCreationDisposition != CREATE_NEW &&
        (flags.noFollowLinks ||
         flags.openReparsePoint ||
         flags.deleteOnClose))
    {
        if (flags.noFollowLinks || flags.deleteOnClose)
            okayToFollowLinks = false;
        dwFlagsAndAttributes |= FILE_FLAG_OPEN_REPARSE_POINT;
    }

    // permission check
    if (pathToCheck != null) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            if (flags.read)
                sm.checkRead(pathToCheck);
            if (flags.write)
                sm.checkWrite(pathToCheck);
            if (flags.deleteOnClose)
                sm.checkDelete(pathToCheck);
        }
    }

    // open file
    long handle = CreateFile(pathForWindows,
                             dwDesiredAccess,
                             dwShareMode,
                             pSecurityDescriptor,
                             dwCreationDisposition,
                             dwFlagsAndAttributes);

    // make sure this isn't a symbolic link.
    if (!okayToFollowLinks) {
        try {
            if (WindowsFileAttributes.readAttributes(handle).isSymbolicLink())
                throw new WindowsException("File is symbolic link");
        } catch (WindowsException x) {
            CloseHandle(handle);
            throw x;
        }
    }

    // truncate file (for CREATE + TRUNCATE_EXISTING case)
    if (truncateAfterOpen) {
        try {
            SetEndOfFile(handle);
        } catch (WindowsException x) {
            CloseHandle(handle);
            throw x;
        }
    }

    // make the file sparse if needed
    if (dwCreationDisposition == CREATE_NEW && flags.sparse) {
        try {
            DeviceIoControlSetSparse(handle);
        } catch (WindowsException x) {
            // ignore as sparse option is hint
        }
    }

    // create FileDescriptor and return
    FileDescriptor fdObj = new FileDescriptor();
    fdAccess.setHandle(fdObj, handle);
    return fdObj;
}

// 类似地 linux上的实现由 UnixChannelFactory 完成
// 打开文件，获得文件描述符fd,然后创建 FileChannelImpl 对象。
```