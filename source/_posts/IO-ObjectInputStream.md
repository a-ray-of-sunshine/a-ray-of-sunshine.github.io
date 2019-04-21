---
title: IO-ObjectInputStream
date: 2016-9-11 9:38:02
---

ObjectInputStream 和 ObjectOutputStream 都是和对象 Serializable 有关。

## writeObject 的过程

### 原生数据类型的序列化

创建 ObjectOutputStream 的过程。

``` java
public ObjectOutputStream(OutputStream out) throws IOException {
    verifySubclass();
	// 持有底层的 OutputStream 和 DataOutputStream
	// 用来完成写出数据的作用。
    bout = new BlockDataOutputStream(out);
    handles = new HandleTable(10, (float) 3.00);
    subs = new ReplaceTable(10, (float) 3.00);
    enableOverride = false;

	// 向流中写入 头部信息。
    writeStreamHeader();
	// 调整 bout 写入数据的模式，
	// setBlockDataMode这个方法，如果原先的模式和
	// 现在设置的模式不同，则会把 buffer 中的数据先写入到
	// out 流，也就是清空 buffer.
    bout.setBlockDataMode(true);
    if (extendedDebugInfo) {
        debugInfoStack = new DebugTraceInfoStack();
    } else {
        debugInfoStack = null;
    }
}

// 将buffer的数据写入到底层的流中。
// 可以看到这个方法中使用到了 blkmode， 此时是 true
// 则会调用 writeBlockHeader 方法。
// 这个方法会向流中写入两个数据，
// tag byte: TC_BLOCKDATA 表示下一个字节是block的长度。 
// len byte: 这个块的长度，占一个字节。
// 注意，如果块的长度 <= 255 则使用上面的 TC_BLOCKDATA 标记，
// 否则，使用 TC_BLOCKDATALONG，来标识。表示 后续 4 个字节，是数据的len.
// 所以 writeBlockHeader 方法写入的数据是变长的： 2 或者 5.
void drain() throws IOException {
    if (pos == 0) {
        return;
    }
    if (blkmode) {
        writeBlockHeader(pos);
    }
    out.write(buf, 0, pos);
    pos = 0;
}
```

BlockDataOutputStream 在什么时候会形成一个 data block.

* BlockDataOutputStream.MAX_BLOCK_SIZE = 1024 [0x400]

	``` java
   if (pos >= MAX_BLOCK_SIZE) {
        drain();
    }
	```

* 调用 flush 或者 close 方法时

	``` java
    public void flush() throws IOException {
        drain();
        out.flush();
    }

    public void close() throws IOException {
        flush();
        out.close();
    }
	```

在 drain 方法中，如果 当前是 块模式，则调用 writeBlockHeader 生成一个数据块头。

``` java
void drain() throws IOException {
    if (pos == 0) {
        return;
    }
    if (blkmode) {
		// 生成数据块头， 2 或者 5 个字节。
        writeBlockHeader(pos);
    }
    out.write(buf, 0, pos);
    pos = 0;
}
```

BlockDataOutputStream 的 blkmode 状态默认 false, 表示这个流是 关闭块模式。块模式表示的是 可以向这个流中写入数据，写入的数据默认存储在 buffer 中，当数据最终写入到 buffer 之前，会在这个前面加一个 BlockHeader。

ObjectOutputStream 内部有一个 BlockDataOutputStream 这个对象中拥有缓冲区，流的所有写入操作都先写入到这个对象的缓冲区中，

此时是写到，缓冲区的。
writeStreamHeader
写入流的 STREAM_MAGIC 和 STREAM_VERSION 共 4 个字节。

``` java
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("data"));

oos.writeInt(9999);

oos.close();

// 上面的代码将生成 data 文件的结构如下：

         STREAM_VERSION block_size
               ↓            ↓
          ↓_________↓    ↓____↓
+----+----+----+----+----+----+----+----+----+----+
| AC | ED | 00 | 05 | 77 | 04 | 00 | 00 | 27 | 0f |
+----+----+----+----+----+----+----+----+----+----+
↑_________↑         ↑____↑    ↑___________________↑
     ↑                 ↑                ↑
STREAM_MAGIC     TC_BLOCKDATA          data

其中 data 存储的字节序是：大端。
并且 data 区域，并不维护，数据类型，数据数据的维护，由代码来维护，代码当然知道，写入的数据类型是什么。
```

### 对象类型的序列化

ObjectStreamClass

ObjectOutputStream.writeObject0

写入一个对象：

writeOrdinaryObject

``` java


// 在写入对象之前会创建对象的 ObjectStreamClass 对象 desc.
private void writeObject0(Object obj, boolean unshared) throws IOException{

	// ...

	Class cl = obj.getClass();
    ObjectStreamClass desc;
	// 这个 lookup 方法将创建 cl 的 ObjectStreamClass
	// 同时也会创建其 supclass 的 ObjectStreamClass
    desc = ObjectStreamClass.lookup(cl, true);

	// ...

	writeOrdinaryObject(obj, desc, unshared);
}

private void writeOrdinaryObject(Object obj,
                                 ObjectStreamClass desc,
                                 boolean unshared)
    throws IOException
{
    if (extendedDebugInfo) {
        debugInfoStack.push(
            (depth == 1 ? "root " : "") + "object (class \"" +
            obj.getClass().getName() + "\", " + obj.toString() + ")");
    }
    try {
        desc.checkSerialize();

	    // 写入对象头标记
        bout.writeByte(TC_OBJECT);

		// 写入类描述，最后调用：writeNonProxyDesc
        writeClassDesc(desc, false);
        handles.assign(unshared ? null : obj);
        if (desc.isExternalizable() && !desc.isProxy()) {
            writeExternalData((Externalizable) obj);
        } else {
			// 写入对象的数据
            writeSerialData(obj, desc);
        }
    } finally {
        if (extendedDebugInfo) {
            debugInfoStack.pop();
        }
    }
}

// 写入类描述: 1. 由 writeClassDesc 调用
private void writeNonProxyDesc(ObjectStreamClass desc, boolean unshared)
    throws IOException
{
    bout.writeByte(TC_CLASSDESC);
    handles.assign(unshared ? null : desc);

    if (protocol == PROTOCOL_VERSION_1) {
        // do not invoke class descriptor write hook with old protocol
        desc.writeNonProxy(this);
    } else {
		// 调用 ObjectStreamClass.writeNonProxy
        writeClassDescriptor(desc);
    }

    Class cl = desc.forClass();
    bout.setBlockDataMode(true);
    if (isCustomSubclass()) {
        ReflectUtil.checkPackageAccess(cl);
    }
    annotateClass(cl);
    bout.setBlockDataMode(false);
	// 表示数据块结束。
    bout.writeByte(TC_ENDBLOCKDATA);
	
	// 开始递归调用 写入 desc 的父类的 描述信息。
    writeClassDesc(desc.getSuperDesc(), false);
}

// 写入类描述
void writeNonProxy(ObjectOutputStream out) throws IOException {
	
	// 写入类名
    out.writeUTF(name);
	// 写入类的 SerialVersionUID
    out.writeLong(getSerialVersionUID());

    byte flags = 0;
    if (externalizable) {
        flags |= ObjectStreamConstants.SC_EXTERNALIZABLE;
        int protocol = out.getProtocolVersion();
        if (protocol != ObjectStreamConstants.PROTOCOL_VERSION_1) {
            flags |= ObjectStreamConstants.SC_BLOCK_DATA;
        }
    } else if (serializable) {
        flags |= ObjectStreamConstants.SC_SERIALIZABLE;
    }
    if (hasWriteObjectData) {
        flags |= ObjectStreamConstants.SC_WRITE_METHOD;
    }
    if (isEnum) {
        flags |= ObjectStreamConstants.SC_ENUM;
    }

	// 写入 flags = ObjectStreamConstants.SC_SERIALIZABLE
    out.writeByte(flags);

	// 写入字段信息：
	// 字段个数
    out.writeShort(fields.length);
    for (int i = 0; i < fields.length; i++) {
        ObjectStreamField f = fields[i];
		// 字段类型代码
        out.writeByte(f.getTypeCode());
        // 字段名称
		out.writeUTF(f.getName());
        if (!f.isPrimitive()) {
			// 如果字段不是原生数据类型，则将字段的类型信息记录下来
            out.writeTypeString(f.getTypeString());
        }
    }
}

// 写入对象数据
private void writeSerialData(Object obj, ObjectStreamClass desc)
    throws IOException
{
    ObjectStreamClass.ClassDataSlot[] slots = desc.getClassDataLayout();
    for (int i = 0; i < slots.length; i++) {
        ObjectStreamClass slotDesc = slots[i].desc;
        if (slotDesc.hasWriteObjectMethod()) {
			// 对象有 writeObject 方法，则调用对象的 writeObject 方法
			// 来完成对象的序列化。
            PutFieldImpl oldPut = curPut;
            curPut = null;
            SerialCallbackContext oldContext = curContext;

            if (extendedDebugInfo) {
                debugInfoStack.push(
                    "custom writeObject data (class \"" +
                    slotDesc.getName() + "\")");
            }
            try {
                curContext = new SerialCallbackContext(obj, slotDesc);
                bout.setBlockDataMode(true);
				// slotDesc.invokeWriteObject 调用 obj.writeObject
				// writeObjectMethod.invoke(obj, new Object[]{ out });
                slotDesc.invokeWriteObject(obj, this);
                bout.setBlockDataMode(false);
                bout.writeByte(TC_ENDBLOCKDATA);
            } finally {
                curContext.setUsed();
                curContext = oldContext;
                if (extendedDebugInfo) {
                    debugInfoStack.pop();
                }
            }

            curPut = oldPut;
        } else {
			// 对象没有 writeObject 方法，则调用
			// defaultWriteFields 默认的序列化方法
			// 进行对象序列化
			// defaultWriteFields 会将所有
			// non-transient and non-static fields
			// 进行序列化
            defaultWriteFields(obj, slotDesc);
        }
    }
}

```

同理，对于 readObject 方法的调用机制，也是如此，并且，如果 一个 Serializable 提供了 writeObject 方法，则其就应该提供 readObject 方法。这两个方法的签名是：

``` java
private void readObject(java.io.ObjectInputStream stream)
     throws IOException, ClassNotFoundException;
private void writeObject(java.io.ObjectOutputStream stream)
     throws IOException
```

ObjectStreamClass 的构造

``` java

private ObjectStreamClass(final Class<?> cl) {

    serializable = Serializable.class.isAssignableFrom(cl);
    externalizable = Externalizable.class.isAssignableFrom(cl);

    Class<?> superCl = cl.getSuperclass();
	// 初始化 superDesc, 从而进入递归调用，将 cl 的继承链
	// 上的所有 superClass 初始化好
    superDesc = (superCl != null) ? lookup(superCl, false) : null;

    if (serializable) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                // ....

				// 初始化 fields
				fields = getSerialFields(cl);

				// 获得对象的 writeObject 和 readObject 方法
				// 如果对象有 writeObject 方法，
				// 则 hasWriteObjectData == true
				// 如果对象有 writeObjectMethod 方法，则在
                if (externalizable) {
                    cons = getExternalizableConstructor(cl);
                } else {
                    cons = getSerializableConstructor(cl);
                    writeObjectMethod = getPrivateMethod(cl, "writeObject",
                        new Class<?>[] { ObjectOutputStream.class },
                        Void.TYPE);
                    readObjectMethod = getPrivateMethod(cl, "readObject",
                        new Class<?>[] { ObjectInputStream.class },
                        Void.TYPE);
                    readObjectNoDataMethod = getPrivateMethod(
                        cl, "readObjectNoData", null, Void.TYPE);
                    hasWriteObjectData = (writeObjectMethod != null);
                }
                writeReplaceMethod = getInheritableMethod(
                    cl, "writeReplace", null, Object.class);
                readResolveMethod = getInheritableMethod(
                    cl, "readResolve", null, Object.class);
                return null;
            }
        });
    }

	// ....
}

```

所以写入对象数据的过程，如下：

1. TC_OBJECT 表示一个对象
2. 写入对象的类描述：例如类名，类中的字段名称，字段类型
3. 写入对象数据，如果对象有 writeObject ，调用 writeObject 写入数据
4. 如果对象没有 writeObject 方法，则调用 defaultWriteFields 将所有 non-static,non-transient 字段写入

	其写入过程： 如果当前字段是原生数据类型，则直接写入，否则，进入递归调用，转到步骤 1 开始写入这个对象字段。直到当前对象的所有字段写完，返回

5. 写 superClass 的数据，重复上面的步骤
6. 直到 superClass 到根类 Object 类。