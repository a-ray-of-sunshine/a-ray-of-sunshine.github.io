---
title: java.nio.file
date: 2016-10-20 16:26:19
---

## java.nio.file的使用

这个包中最核心的概念就是 `Path`， 可以说 `Path` 是整个java.nio.file包和java.nio.file.attribute包中操作的目标，几乎所有的API都是围绕`Path`进行的。

### Path的获取

* 通过 File 类

	``` java
	File file = new File("/file.txt");
	Path path = file.toPath();
	```

* 通过 Paths 类

	``` java
	Path path = Paths.getPath("/file.txt");
	```

* 通过 FileSystem 类

	``` java
	FileSystem fileSystem = FileSystems.getDefault();
	Path path = fileSystem.getPath("/file.txt");
	```
	
其中前两种方式，获得的是当前JVM默认的FileSystem来获取Path. 

default file system：

> The default file system creates objects that provide access to the file systems accessible to the Java virtual machine.

java.io.File 使用是就是JVM运行环境所在的文件系统。所以通过将两种方式获得的 Path 可以和 File 对象进行互操作。

FileSystems 类可以获得默认的文件系统，同时也提供了创建文件系统的方法。例如：

``` java
// 获得 zip 文件系统
Map<String, String> env = new HashMap<>(); 
env.put("create", "true");
URI uri = URI.create("jar:file:/tmp/fszip.zip");

// 通过下面的方式获得 zip 文件系统。
FileSystem zipfs = FileSystems.newFileSystem(uri, env)
// 注意：此时获得的 Path 是 /tmp/fszip.zip 这个zip
// 压缩包中的 demo.txt 文件，而不是当前文件系统中的
// 文件。
Path pathInZipfile = zipfs.getPath("/demo.txt");
```

### FileSystem的获取

* FileSystems.newFileSystem
* path.getFileSystem

第二种方式是通过 Path 类的实现，来获得其所在的文件系统的抽象 FileSystem

## FileSystem & FileSystemProvider

wiki 中关于 File System 的描述：

> In computing, a file system or filesystem is used to control how data is stored and retrieved. Without a file system, information placed in a storage medium would be one large body of data with no way to tell where one piece of information stops and the next begins. By separating the data into pieces and giving each piece a name, the information is easily isolated and identified. Taking its name from the way paper-based information systems are named, each group of data is called a "file". **The structure and logic rules used to manage the groups of information and their names is called a "file system".**

可以看到文件系统，其实就是对数据如何存储的一种机制。从而有了目录和文件的概念，目录和文件就可以使用 Path 来表示。

同时对于一个目录和文件，文件系统可以提供一种抽象，就是对它们进行属性设置，例如：文件A 不能被执行。目录B 不可以被某个用户访问。这些信息，本身和文件数据（data）是没有任何关系，只是文件系统提供的一种属性，而这种属性则是存储在文件系统中的，所以 FileStore 可以用来表示这种抽象，通过FileStore可以查询到文件的属性。

基于目录层次文件系统抽象，还有一个概念就是根目录（RootDirectory）。所有新创建的文件和目录必然是从根目录开始的。

对于操作系统而言，文件系统提供了创建目录及文件的功能，自然对于文件的任何变化，例如文件属性的变化，文件内容的变化，都由文件系统来完成，则文件系统就可以提供一种机制，让用户（其它程序）监测文件的变化，如果文件发生变化，则这个程序可以得到通知。所以文件系统也应该能够提供 WatchService 的功能。

而上面的这些功能就可以接口的形式提供给用户，所以对于 java 中 FileSystem 类就像一个工厂一样，将一个文件系统所能提供的功能都以接口的形式提供出来：Path, PathMatcher,FileStore,UserPrincipalLookupService,WatchService 等等。

所以说 FileSystemProvider 中提供的仅仅是一个 文件系统 提供给用户的服务，例如：创建目录，创建文件等。

但 文件系统 并不等同于上面的服务。还有其存储元数据的信息等等。

所以说 将文件系统抽象为 FileSystem & FileSystemProvider 这两个类，是比较清晰的表达的文件系统的概念。

其实可以把 FileSystemProvider 类的方法移到 FileSystem 中，从逻辑上讲也是没有问题的。可以参考 Hadoop 对 文件系统 的抽象，就是这么做的。

FileSystem:

> Provides an interface to a file system and is the factory for objects to access files and other objects in the file system.

FileSystemProvider:

> Service-provider class for file systems. 
> 
> A file system provider is a concrete implementation of this class that implements the abstract methods defined by this class.
> 
> A provider is identified by a URI scheme.
> 
> A provider is a factory for one or more FileSystem instances.


* Path

	`FileSystem.getPath`
	
* PathMatcher

	`FileSystem.getPathMatcher`
	
* FileStore

	`FileSystem.getFileStores`
	
* UserPrincipalLookupService

	`FileSystem.getUserPrincipalLookupService`
	
* WatchService

	`FileSystem.newWatchService`

## JVM如何动态发现已经安装的Provider

通过 java.util.ServiceLoader

> A service provider is identified by placing a provider-configuration file in the resource directory META-INF/services. The file's name is the fully-qualified binary name of the service's type. The file contains a list of fully-qualified binary names of concrete provider classes, one per line.

## WatchService & Watchable

WatchService 可以用来检测文件系统中目录的变化。其使用方法如下。

``` java

FileSystem fs = FileSystems.getDefault();

Path path = Paths.get("/temp");
WatchService ws = fs.newWatchService();

// Path 是一个 Watchable 对象。
// 所以可以注册一个需要关注的事件，
// 当事件发生时，就可以通过
path.register(ws, StandardWatchEventKinds.ENTRY_CREATE);

WatchKey take = ws.take();
```

### WindowsWatchService

windows 平台如何实现 WatchService：

#### 创建 WatchService

``` java
/*
 * Win32 implementation of WatchService based on ReadDirectoryChangesW.
 */

class WindowsWatchService
    extends AbstractWatchService
{
	// background thread to service I/O completion port
	private final Poller poller;
	
	/**
	 * Creates an I/O completion port and a daemon thread to service it
	 */
	WindowsWatchService(WindowsFileSystem fs) throws IOException {
	    // create I/O completion port
	    long port = 0L;
	    try {
	    	// 创建 IO 完成端口
	    	// If INVALID_HANDLE_VALUE is specified, the function creates an I/O completion port without associating it with a file handle. In this case, 
	    	// the ExistingCompletionPort parameter must be NULL and the CompletionKey parameter is ignored.
	    	// 此时这个还没有还任何 handle(文件句柄) 关联。
	        port = CreateIoCompletionPort(INVALID_HANDLE_VALUE, 0, 0);
	    } catch (WindowsException x) {
	        throw new IOException(x.getMessage());
	    }
	
		// 创建一个 轮询线程（Poller）
	    this.poller = new Poller(fs, this, port);
	    this.poller.start();
	}
	
	// ...
}

// Poller的 run 方法
/**
 * Poller main loop
 */
@Override
public void run() {
    for (;;) {
        CompletionStatus info = null;
        try {
        	// Attempts to dequeue an I/O completion packet from the specified I/O completion port. 
        	// If there is no completion packet queued, the function waits for a pending I/O operation associated with the completion port to complete.
        	// 在完成端口处等待，直到有完成数据包返回。
            info = GetQueuedCompletionStatus(port);
        } catch (WindowsException x) {
            // this should not happen
            x.printStackTrace();
            return;
        }

        // wakeup
        if (info.completionKey() == 0) {
        	// 有新的事件需要注册
        	// 处理 requestList 对象
        	// 如果轮询器（Poller）被关闭（close）
        	// 则返回的 shutdown 为ture
        	// 从而轮询结束
            boolean shutdown = processRequests();
            if (shutdown) {
                return;
            }
            continue;
        }

        // map completionKey to get WatchKey
        WindowsWatchKey key = int2key.get(info.completionKey());
        if (key == null) {
            // We get here when a registration is changed. In that case
            // the directory is closed which causes an event with the
            // old completion key.
            continue;
        }

        // ReadDirectoryChangesW failed
        if (info.error() != 0) {
            // buffer overflow
            if (info.error() == ERROR_NOTIFY_ENUM_DIR) {
                key.signalEvent(StandardWatchEventKinds.OVERFLOW, null);
            } else {
                // other error so cancel key
                implCancelKey(key);
                key.signal();
            }
            continue;
        }

        // process the events
        if (info.bytesTransferred() > 0) {
            processEvents(key, info.bytesTransferred());
        } else {
            // insufficient buffer size
            key.signalEvent(StandardWatchEventKinds.OVERFLOW, null);
        }

        // start read for next batch of changes
        try {
            ReadDirectoryChangesW(key.handle(),
                                  key.buffer().address(),
                                  CHANGES_BUFFER_SIZE,
                                  key.watchSubtree(),
                                  ALL_FILE_NOTIFY_EVENTS,
                                  key.countAddress(),
                                  key.overlappedAddress());
        } catch (WindowsException x) {
            // no choice but to cancel key
            implCancelKey(key);
            key.signal();
        }
    }
}


// 为目录的变化注册io完成端口。
/**
 * Register a directory for changes as follows:
 *
 * 1. Open directory
 * 2. Read its attributes (and check it really is a directory)
 * 3. Assign completion key and associated handle with completion port
 * 4. Call ReadDirectoryChangesW to start (async) read of changes
 * 5. Create or return existing key representing registration
 */
@Override
Object implRegister(Path obj,
                    Set<? extends WatchEvent.Kind<?>> events,
                    WatchEvent.Modifier... modifiers)
{
    WindowsPath dir = (WindowsPath)obj;
    boolean watchSubtree = false;

    // FILE_TREE modifier allowed
    for (WatchEvent.Modifier modifier: modifiers) {
        if (modifier == ExtendedWatchEventModifier.FILE_TREE) {
            watchSubtree = true;
            continue;
        } else {
            if (modifier == null)
                return new NullPointerException();
            if (modifier instanceof com.sun.nio.file.SensitivityWatchEventModifier)
                continue; // ignore
            return new UnsupportedOperationException("Modifier not supported");
        }
    }

    // open directory
    // 1. 打开目录
    long handle = -1L;
    try {
        handle = CreateFile(dir.getPathForWin32Calls(),
                            FILE_LIST_DIRECTORY,
                            (FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE),
                            OPEN_EXISTING,
                            FILE_FLAG_BACKUP_SEMANTICS | FILE_FLAG_OVERLAPPED);
    } catch (WindowsException x) {
        return x.asIOException(dir);
    }

    boolean registered = false;
    try {
        // read attributes and check file is a directory
        // 检测文件是否是目录，如果不是目录则抛出异常
        WindowsFileAttributes attrs = null;
        try {
            attrs = WindowsFileAttributes.readAttributes(handle);
        } catch (WindowsException x) {
            return x.asIOException(dir);
        }
        if (!attrs.isDirectory()) {
            return new NotDirectoryException(dir.getPathForExceptionMessage());
        }

		// 检测目录是否已经注册
        // check if this directory is already registered
        FileKey fk = new FileKey(attrs.volSerialNumber(),
                                 attrs.fileIndexHigh(),
                                 attrs.fileIndexLow());
        WindowsWatchKey existing = fk2key.get(fk);

        // if already registered and we're not changing the subtree
        // modifier then simply update the event and return the key.
        if (existing != null && watchSubtree == existing.watchSubtree()) {
        	// 如果目录已经注册，则将当前需要 registe 的事件
        	// 添加到 existing
            existing.setEvents(events);
            return existing;
        }

        // unique completion key (skip 0)
        // IO 完成端口的 completionKey 
        int completionKey = ++lastCompletionKey;
        if (completionKey == 0)
            completionKey = ++lastCompletionKey;

        // associate handle with completion port
        try {
        	// Poller 对象创建的时候，会创建一个 port 对象。
        	// 这个 IO完成端口对象并没有和任何 handle 关联
        	// 调用下面的方法将 handle 和 port 关联起来。
            CreateIoCompletionPort(handle, port, completionKey);
        } catch (WindowsException x) {
            return new IOException(x.getMessage());
        }

        // allocate memory for events, including space for other structures
        // needed to do overlapped I/O
        int size = CHANGES_BUFFER_SIZE + SIZEOF_DWORD + SIZEOF_OVERLAPPED;
        NativeBuffer buffer = NativeBuffers.getNativeBuffer(size);

        long bufferAddress = buffer.address();
        long overlappedAddress = bufferAddress + size - SIZEOF_OVERLAPPED;
        long countAddress = overlappedAddress - SIZEOF_DWORD;

        // start async read of changes to directory
        // 启动目录异步读
        try {
            ReadDirectoryChangesW(handle,
                                  bufferAddress,
                                  CHANGES_BUFFER_SIZE,
                                  watchSubtree,
                                  ALL_FILE_NOTIFY_EVENTS,
                                  countAddress,
                                  overlappedAddress);
        } catch (WindowsException x) {
            buffer.release();
            return new IOException(x.getMessage());
        }

        WindowsWatchKey watchKey;
        if (existing == null) {
            // not registered so create new watch key
            // 创建一个 WindowsWatchKey
            watchKey = new WindowsWatchKey(dir, watcher, fk)
                .init(handle, events, watchSubtree, buffer, countAddress,
                      overlappedAddress, completionKey);
            // map file key to watch key
            fk2key.put(fk, watchKey);
        } else {
            // directory already registered so need to:
            // 1. remove mapping from old completion key to existing watch key
            // 2. release existing key's resources (handle/buffer)
            // 3. re-initialize key with new handle/buffer
            int2key.remove(existing.completionKey());
            existing.releaseResources();
            watchKey = existing.init(handle, events, watchSubtree, buffer,
                countAddress, overlappedAddress, completionKey);
        }
        // map completion map to watch key
        int2key.put(completionKey, watchKey);

        registered = true;
        return watchKey;

    } finally {
        if (!registered) CloseHandle(handle);
    }
}
```

WindowsWatchService 对象持有一个 Poller 对象。Poller 中持有一个 port 这是一个IO完成端口。

WindowsWatchService 被创建（FileSystem.newWatchService）时创建一个Poller 对象，然后调用 Poller 对象的 start 方法，start 方法中会启动一个线程。这个线程调用 Poller（实现了Runnable接口）的 run 方法。开始进行循环。这个循环的功能就是：

1. 在 io 完成端口上等待
2. 上面的等待可能会返回，其返回的原因，可能是，被其它线程唤醒了
3. 线程因为：Path.register 方法被唤醒，则是提交新的 Watch.Event
4. 线程因为：WatchService.close 方法，Closes this watch service. 关闭 WatchService 就是关闭 Poller 的主循环。
5. 线程因为：WatchKey.cancel 方法，Cancels the registration with the watch service.
6. 上面三种方法，都调用 AbstractPoller.invoke 方法

	``` java
    // Enqueues request to poller thread and waits for result
    private Object invoke(RequestType type, Object... params) throws IOException {
        // 1. submit request
        Request req = new Request(type, params);
        synchronized (requestList) {
            if (shutdown) {
                throw new ClosedWatchServiceException();
            }
            requestList.add(req);
        }

        // 2. wakeup thread
        // 也就是唤醒 Poller 主循环
        // 处理请求 requestList 中的请求。
        wakeup();

        // 3. wait for result
        // 等待请求的结束
        Object result = req.awaitResult();

        if (result instanceof RuntimeException)
            throw (RuntimeException)result;
        if (result instanceof IOException )
            throw (IOException)result;
        return result;
    }
	```
7. 如果主循环是因为上面三种情况被循环则，在主循环中调用 processRequests用来相应请求
8. 由于IO请求完成，而 main loop 被唤醒，则处理，返回事件。
9. 调用 processEvents， 最终调用  key.signalEvent(kind, name);
10.  key.signalEvent 最终 调用 signal 方法
11.  signal 方法将 key 添加到 WatchService 的一个阻塞队列中
12.  在 WatchService 上调用 take 方法的线程将，take 将从上面的队列中返回数据
	
	``` java
	// AbstractWatchService
	// signaled keys waiting to be dequeued
    private final LinkedBlockingDeque<WatchKey> pendingKeys =
        new LinkedBlockingDeque<WatchKey>();

    @Override
    public final WatchKey take()
        throws InterruptedException
    {
        checkOpen();
        WatchKey key = pendingKeys.take();
        checkKey(key);
        return key;
    }
	```

## 查询和设置文件属性

文件属性相关的包 java.nio.file.attribute

### 获得 FileAttributeView

``` java
Files.getFileAttributeView
```
> Returns a file attribute view of a given type. 

### 获得 FileStoreAttributeView 

``` java
// 获得 FileStore 对象
Files.getFileStore
// 使用 FileStore 对象，获取FileStoreAttributeView
FileStore.getFileStoreAttributeView
```

## Files 类

> This class consists exclusively of static methods that operate on files, directories, or other types of files. 
>
> Service-provider class for file systems. The methods defined by the Files class will typically delegate to an instance of this class(FileSystemProvider). 

这个类提供了关于文件系统的操作接口。