---
title: AsynchronousFileChannel
date: 2016-10-19 17:47:50
---

## AsynchronousFileChannel

> An asynchronous channel for reading, writing, and manipulating a file.

这个类实现了 `AsynchronousChannel` 接口，但是接口 `AsynchronousChannel` 是一个 marker interface (标记型接口)，其中并没有对于异步操作抽象出具体的方法签名(method sign)，而是仅仅给出了 异步方法的通用约定形式。

> A channel that supports asynchronous I/O operations. Asynchronous I/O operations will usually take one of two forms: 
> 
> * Future<V> operation(...)
> 
> * void operation(... A attachment, CompletionHandler<V,? super A> handler)
> 
> In the first form, the methods defined by the Future interface may be used to check if the operation has completed, wait for its completion, and to retrieve the result. 
> 
> In the second form, a CompletionHandler is invoked to consume the result of the I/O operation when it completes or fails. 

这两种方式，最终实现的功能都是一样的，那么如何选择呢？如果，我们需要对返回结果做特殊处理，例如提交到某个线程池中去执行，那么可以选择 Future 类的API，这样可以自主设置线程池配置等。否则，就选择 CompletionHandler 这种回调机制，但是 CompletionHandler 回调机制，也存在一定的问题，由于回调是 AsynchronousFileChannel 内部用来执行 read/write 操作的线程池进行的，如果回调任务的实现是耗时操作，则可能造成 内部线程池 的吞吐量下降。所以如果选择回调机制处理异步任务，则实现的回调功能应该尽量简洁高效，不要出现阻塞。

**这个类提供的 read/write 方法都不会改变 file pointer.**

> An asynchronous file channel does not have a current position within the file. Instead, the file position is specified to each read and write method that initiates asynchronous operations. 

获得这个类的对象的方法：**AsynchronousFileChannel.open**

## 如何使用 AsynchronousFileChannel

AsynchronousFileChannel 类对 `read, writer, lock` 方法都提供了上面两种异步接口，使用示例如下：

### Future<V> operation(...)

对于返回 Future 对象的方法，其提供一个 get 方法可以等待到线程获得数据。

``` java

public class AIOTest {

	public static void main(String[] args) throws IOException {
		
		Path path = Paths.get("file.txt");
		
		AsynchronousFileChannel afc = AsynchronousFileChannel.open(path);
		
		// 建立一个 100KB 的缓冲区
		ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024); 

		// 异步read,并不会阻塞当前线程
		// 此时, 任务已经被 AsynchronousFileChannel 内部持有的线程池开始
		// 调度执行了
		Future<Integer> future = afc.read(buffer, 0L);
		
		IOTask<Integer> task = new IOTask<Integer>(future, buffer);
		
		// 这里可以直接使用线程，也可以考虑使用线程池
		Thread iothread = new Thread(task);
		iothread.setName("io-thread");
		iothread.start();

		System.out.println(Thread.currentThread() + ": =======main=======");
		
	}
}

class IOTask<V> implements Runnable {
	
	private Future<V> future;
	private Buffer buffer;
	
	public IOTask(Future<V> future, Buffer buffer) {
		this.future = future;
		this.buffer = buffer;
	}

	@Override
	public void run() {
		
		try {
			System.out.println(Thread.currentThread() + " before: " + buffer.remaining());

			// 等待异步任务的结束
			future.get();
			
			// 任务结束, buffer中存储着返回的数据
			// 可以使用 buffer 了
			System.out.println(Thread.currentThread() + " after: " + buffer.remaining());

		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
	}
}

```

### void operation(... A attachment, CompletionHandler<V,? super A> handler)

``` java
public class AIOTest {

	public static void main(String[] args) throws IOException {

	Path path = Paths.get("/msdia80.dll");
	
	AsynchronousFileChannel afc = AsynchronousFileChannel.open(path);
	
	// 建立一个 100KB 的缓冲区
	ByteBuffer buffer = ByteBuffer.allocateDirect(1024); 
	
	// 建立 handler
	CompletionHandler<Integer, Buffer> handler = new Handler();
	
	// handler 将共享 afc 内部持有的 线程池
	// 也就是说 afc 执行完毕 read 之后将调用 handler
	afc.read(buffer, 0, buffer, handler);
	
	System.out.println(Thread.currentThread() + ": =======main=======");
	}
}	
	
class Handler implements CompletionHandler<Integer, Buffer>{

	@Override
	public void completed(Integer result, Buffer attachment) {
		System.out.println(Thread.currentThread() + " after: " + attachment);
	}

	@Override
	public void failed(Throwable exc, Buffer attachment) {
		System.out.println(Thread.currentThread() + " after: " + exc);
	}
}
```

## 异步任务出现异常如何处理

如果异步任务出现异常，通常应该记录日志，这两种不同的异步执行机制如何实现异常处理：

### Future

Future 是通过 get 方法来获得计算结果，这个方法会阻塞直到异步任务结束。当异步任务正常结束，则get 方法返回计算结果。否则 get 方法会抛出ExecutionException 来表示异步任务执行过程出现异常。可以通过 catch 这个异常来处理，异步任务执行出现的异常。 

### CompletionHandler

这个接口提供了 failed 方法，当异步任务(read/write)出现异常,则 failed 方法会被调用。可以在这里处理异常，例如记录日志录。

## AsynchronousFileChannel 的创建

open 方法的实现：

``` java
// WindowsChannelFactory
static AsynchronousFileChannel newAsynchronousFileChannel(String pathForWindows, String pathToCheck, 
	Set<? extends OpenOption> options, long pSecurityDescriptor, ThreadPool pool)
    throws IOException
{
    Flags flags = Flags.toFlags(options);

    // Overlapped I/O required
    flags.overlapped = true;

    // default is reading
    if (!flags.read && !flags.write) {
        flags.read = true;
    }

    // validation
    if (flags.append)
        throw new UnsupportedOperationException("APPEND not allowed");

    // open file for overlapped I/O
    // 获得 FileDescriptor 对象
    FileDescriptor fdObj;
    try {
        fdObj = open(pathForWindows, pathToCheck, flags, pSecurityDescriptor);
    } catch (WindowsException x) {
        x.rethrowAsIOException(pathForWindows);
        return null;
    }

    // create the AsynchronousFileChannel
    // 创建 AsynchronousFileChannel 对象
    try {
        return WindowsAsynchronousFileChannelImpl.open(fdObj, flags.read, flags.write, pool);
    } catch (IOException x) {
        // IOException is thrown if the file handle cannot be associated
        // with the completion port. All we can do is close the file.
        long handle = fdAccess.getHandle(fdObj);
        CloseHandle(handle);
        throw x;
    }
}

// linux
// UnixChannelFactory
static AsynchronousFileChannel newAsynchronousFileChannel(UnixPath path, Set<? extends OpenOption> options, int mode, ThreadPool pool)
    throws UnixException
{
    Flags flags = Flags.toFlags(options);

    // default is reading
    if (!flags.read && !flags.write) {
        flags.read = true;
    }

    // validation
    if (flags.append)
        throw new UnsupportedOperationException("APPEND not allowed");

    // for now use simple implementation
    FileDescriptor fdObj = open(-1, path, null, flags, mode);
    return SimpleAsynchronousFileChannelImpl.open(fdObj, flags.read, flags.write, pool);
}
```

由上面的 open 方法可知 AsynchronousFileChannel 的内部实现：

* windows: WindowsAsynchronousFileChannelImpl
* linux: SimpleAsynchronousFileChannelImpl

### SimpleAsynchronousFileChannelImpl

``` java
// SimpleAsynchronousFileChannelImpl.implRead
<A> Future<Integer> implRead(final ByteBuffer dst,
                             final long position,
                             final A attachment,
                             final CompletionHandler<Integer,? super A> handler)
{
    if (position < 0)
        throw new IllegalArgumentException("Negative position");
    if (!reading)
        throw new NonReadableChannelException();
    if (dst.isReadOnly())
        throw new IllegalArgumentException("Read-only buffer");

    // complete immediately if channel closed or no space remaining
    if (!isOpen() || (dst.remaining() == 0)) {
        Throwable exc = (isOpen()) ? null : new ClosedChannelException();
        if (handler == null)
            return CompletedFuture.withResult(0, exc);
        Invoker.invokeIndirectly(handler, attachment, 0, exc, executor);
        return null;
    }

	// 创建 Future 对象
    final PendingFuture<Integer,A> result = (handler == null) ?
        new PendingFuture<Integer,A>(this) : null;
    
    // 创建 task 对象
    Runnable task = new Runnable() {
        public void run() {
            int n = 0;
            Throwable exc = null;

            int ti = threads.add();
            try {
                begin();
                do {
                    n = IOUtil.read(fdObj, dst, position, nd, null);
                } while ((n == IOStatus.INTERRUPTED) && isOpen());
                if (n < 0 && !isOpen())
                    throw new AsynchronousCloseException();
            } catch (IOException x) {
                if (!isOpen())
                    x = new AsynchronousCloseException();
                exc = x;
            } finally {
                end();
                threads.remove(ti);
            }
            
            // 为不同的API提供接口

            if (handler == null) {
            	// 如果 handler 为 null 则 API 的形式应该是
            	// Future<V> operation(...)
            	// 也就是说这个 result 将会作为 operation
            	// 的返回值 Future ，result 的类型是 PendingFuture
            	// 就是 Future 的子类
            	// result.setResult 将会使得所有在 result 这个 Future 
            	// 上进行等待的线程进行通知，
                result.setResult(n, exc);
            } else {
            	// 如果 handler 不为null, 则 上层API的形式应该是：
            	// void operation(... A attachment, CompletionHandler<V,? super A> handler)
            	// 这个方法将对 handler 进行回调
                Invoker.invokeUnchecked(handler, attachment, n, exc);
            }
        }
    };

	// 提交任务
    executor.execute(task);
    return result;
}

// AsynchronousFileChannelImpl.read
@Override
public final Future<Integer> read(ByteBuffer dst, long position) {
    return implRead(dst, position, null, null);
}
@Override
public final <A> void read(ByteBuffer dst,
                           long position,
                           A attachment,
                           CompletionHandler<Integer,? super A> handler)
{
    if (handler == null)
        throw new NullPointerException("'handler' is null");
    implRead(dst, position, attachment, handler);
}


// Invoker.invokeUnchecked
static <V,A> void invokeUnchecked(CompletionHandler<V,? super A> handler,
                                  A attachment, V value, Throwable exc)
{
    if (exc == null) {
		// 未发生异常，正常完成
        handler.completed(value, attachment);
    } else {
		// 发生异常，调用 failed
        handler.failed(exc, attachment);
    }

    // clear interrupt
    Thread.interrupted();
}
```