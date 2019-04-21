---
title: kafka log文件格式
date: 2016-6-8 11:07:19
tags: [kafka,log,index]
---

## topic存储位置
broker在启动的时候会，配置这个broker产生的topic的存储位置，kafka默认的配置在 server.properties 文件中：
``` bash
# A comma seperated list of directories under which to store log files
log.dirs=/tmp/kafka-logs
```
在这里可以查看到各个topic的物理文件。

## 查看 topic 的 index 文件和 log 文件
kafka自身提供了一些工具来查看这些文件，关于这些工具的介绍，参考文档：[system tools](https://cwiki.apache.org/confluence/display/KAFKA/System+Tools)

其中有一个工具：Dump Log Segment

This can print the messages directly from the log files or just verify the indexes correct for the logs

这个工具可以打印出 index 和 log 文件的内容。执行以下命令，查看这个工具的使用帮助：
``` bash
bin/kafka-run-class.sh kafka.tools.DumpLogSegments
```
其中比较重要的两个选项：
``` bash
# 用来指定要查看的 log 或 index 文件
--files <file1, file2, ...> 

# 这是个标记选项，如果带了这个参数，则当解析 log 文件时
# 会将存储的message 和 这个message对应的key都打印出来
--print-data-log
```
## 与文件格式相关的类
查看 kafka.tools.DumpLogSegments 这个类的源代码可知
最终，index 和 log 文件的解析都由kafka.log.FileMessageSet 这个类的 iterator 方法来完成，这个方法的返回值是： Iterator[MessageAndOffset]
这个类的核心代码：
``` java
 new MessageAndOffset(new Message(buffer), offset)
```
最终从文件中读入的字节流buffer都交由 Messsage 类完成解析。
kafka.message.Message 这个类的：
``` java
/**
 * A message. The format of an N byte message is the following:
 *
 * 1. 4 byte CRC32 of the message
 * 2. 1 byte "magic" identifier to allow format changes, value is 2 currently
 * 3. 1 byte "attributes" identifier to allow annotations on the message independent of the version (e.g. compression enabled, type of codec used)
 * 4. 4 byte key length, containing length K
 * 5. K byte key
 * 6. 4 byte payload length, containing length V
 * 7. V byte payload
 *
 * Default constructor wraps an existing ByteBuffer with the Message object with no change to the contents.
 */
class Message(val buffer: ByteBuffer) {
```
这个类描述了Message在文件中存储格式。

log文件中的一个例子： 
``` bash
offset: 291178 position: 180 isvalid: true payloadsize: 10 magic: 0 compresscodec: NoCompressionCodec crc: 576249152 keysize: 2 key: 11 payload: Message_11
```
这是一条 message 记录，共 10 个字段，其中后 7 个字段和上面 message 中定义的信息相对应。还有 3 个字段，没有对应，分别是：

- isvalid：Returns true if the crc stored with the message matches the crc computed off the message contents，这个值是 message 被读取出来之后，进行crc校验之后的结果。
- position: 这个标志的含义是：当前这条 message 在这个文件内的绝对偏移量。也就是 message 的在这个文件中的 起始的字节个数。例如，上面的那条 message，其 position 是 180，表示这个 message 在这个文件的第 180 个字节处开始。这数字详细算法见： kafka.message.MessageSet 类的 entrySize 方法。其实就是头部的 12 字节 + message的size.
- offset: 这个变量的值是： message 头部的前 8 个字节。

##　index 和 log 文件的解析
由上面的分析可知，这两个文件由
kafka.log.FileMessageSet 的 iterator 方法来完成。这个方法的实现代码如下：
``` java
  def iterator(maxMessageSize: Int): Iterator[MessageAndOffset] = {
    new IteratorTemplate[MessageAndOffset] {
      var location = start
      // 这个 buufer 用来存储，channel 中的 message 的头部。
	 // 这个头部包含两个信息。这两个信息共占，12个字节
	 // 所以只分配 12 个字节	 
      val sizeOffsetBuffer = ByteBuffer.allocate(12)
      
      override def makeNext(): MessageAndOffset = {
        if(location >= end)
          return allDone()
          
        // read the size of the item
        sizeOffsetBuffer.rewind()
        // 从 channel 中读取数据
        channel.read(sizeOffsetBuffer, location)
        if(sizeOffsetBuffer.hasRemaining)
          return allDone()
        
        sizeOffsetBuffer.rewind()
        // 从 buffer 中获取，这个 message 的 offset
        // 占 8 个字节（long 类型）
        val offset = sizeOffsetBuffer.getLong()
        // 从 buffer 中获取，这个 message 的 size
        // 占 4 个字节（int 类型）
        val size = sizeOffsetBuffer.getInt()
        if(size < Message.MinHeaderSize)
          return allDone()
        if(size > maxMessageSize)
          throw new InvalidMessageException("Message size exceeds the largest allowable message size (%d).".format(maxMessageSize))
        
        // read the item itself
        val buffer = ByteBuffer.allocate(size)
        // 读取 message 本身，其大小就是上面的size
        channel.read(buffer, location + 12)
        if(buffer.hasRemaining)
          return allDone()
        buffer.rewind()
        
        // increment the location and return the item
        // 调整 location 的位置，用于读取下一条message
        location += size + 12
        new MessageAndOffset(new Message(buffer), offset)
      }
    }
  }
```
## offset 
message 在一个分区内会维护其顺序，这个 offset 是指这条 message 在这个分区内的偏移量。同一个分区中的数据可以存储到不同的文件中，文件大小的指定是在：
server.properties 中
``` bash
log.segment.bytes=10485760
```
不分区中的数据达到这个字节数时，server 就会另创建一个文件，这个文件的命名就是以这个文件中将要存储的第一条数据的offset来命名的。
**由于 offset 存储为 8 字节，而8字节能够表示的最大值是：18446744073709551615，其共有 20 位，所以 server 生成的数据的命名是 20 位，其规则是: 当前这个文件中的第一个 message 中的 offset ,不足20位，前面补0 **

## index

对于 index 文件也可以由 DumpLogSegments 类来查看，解析。由这个类的 dumpIndex 方法来完成。查看这个方法，可知，最终的解析由 kafka.log.OffsetIndex 类来完成。查看这个类的文档：

An index that maps offsets to physical file locations for a particular log segment. This index may be sparse: that is it may not hold an entry for all messages in the log.

The index is stored in a file that is pre-allocated to hold a fixed maximum number of 8-byte entries.

The index supports lookups against a memory-map of this file. These lookups are done using a simple binary search variant to locate the offset/location pair for the greatest offset less than or equal to the target offset.

Index files can be opened in two ways: either as an empty, mutable index that allows appends or an immutable read-only index file that has previously been populated. The makeReadOnly method will turn a mutable file into an immutable one and truncate off any extra bytes. This is done when the index file is rolled over.

No attempt is made to checksum the contents of this file, in the event of a crash it is rebuilt.

The file format is a series of entries. The physical format is a 4 byte "relative" offset and a 4 byte file location for the message with that offset. The offset stored is relative to the base offset of the index file. So, for example, if the base offset was 50, then the offset 55 would be stored as 5. Using relative offsets in this way let's us use only 4 bytes for the offset.

index 文件： 存储 message 的 offset 和 position 映射 entity, 每个 entity 占8个字节： The physical format is a 4 byte "relative" offset and a 4 byte file location for the message with that offset

这里的 offset 和 log 文件中的 offset 有区别，log 文件中的 offset 是这个分区中的 绝对offset, 而在 index 文件中存储的是相对于当前的这个 index 文件的 相对（relative）offset 

kafka实现参考：[kafka实现](http://blog.csdn.net/kisimple/article/details/42342459)