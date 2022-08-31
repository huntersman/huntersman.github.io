---
title: "Netty之ChannelHandler"
toc: true
toc_label: "目录"
toc_icon: "cog"
toc_sticky: true
categories:
  - Netty
tags:
  - Netty
---

转自[华为开发者论坛](https://developer.huawei.com/consumer/cn/forum/topicview?tid=0201121578210430057&fid=23)

ChannelHandler（管道处理器）其工作模式类似于Java
Servlet过滤器，负责对I/O事件或者I/O操作进行拦截处理。采用事件的好处是，ChannelHandler可以选择自己感兴趣的事件进行处理，也可以对不感兴趣的事件进行透传或者终止。

# ChannelHandler

基于ChannelHandler接口，用户可以方便实现自己的业务，比如记录日志、编解码、数据过滤等，源码如下：

```java
public interface ChannelHandler {

    void handlerAdded(ChannelHandlerContext ctx) throws Exception;

    void handlerRemoved(ChannelHandlerContext ctx) throws Exception;

    @Deprecated
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;

    @Inherited
    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @interface Sharable {
        // no value
    }
}
```

- handlerAdded方法在ChannelHandler被添加到实际上下文中并准备好处理事件后调用。
- handlerRemoved方法在ChannelHandler从实际上下文中移除后调用，表明它不再处理事件。

还有一个Sharable注解，该注解用于表示多个ChannelPipeline可以共享同一个ChannelHandler。

因为ChannelHandler接口过于简单，我们在实际开发中，不会直接实现ChannelHandler接口，因此，Netty提供了ChannelHandlerAdapter抽象类。

# ChannelHandlerAdapter

```java
public abstract class ChannelHandlerAdapter implements ChannelHandler {

    // Not using volatile because it's used only for a sanity check.
    boolean added;

    protected void ensureNotSharable() {
        if (isSharable()) {
            throw new IllegalStateException("ChannelHandler " + getClass().getName() + " is not allowed to be shared");
        }
    }

    public boolean isSharable() {
        Class<?> clazz = getClass();
        Map<Class<?>, Boolean> cache = InternalThreadLocalMap.get().handlerSharableCache();
        Boolean sharable = cache.get(clazz);
        if (sharable == null) {
            sharable = clazz.isAnnotationPresent(Sharable.class);
            cache.put(clazz, sharable);
        }
        return sharable;
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        // NOOP
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        // NOOP
    }

    @Skip
    @Override
    @Deprecated
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }
}
```

ChannelHandlerAdapter对exceptionCaught方法做了实现，并提供了isSharable方法。需要注意的是，ChannelHandlerAdapter是抽象类，用户可以自由的选择是否要覆盖ChannelHandlerAdapter类的实现。如果对某个方法感兴趣，直接覆盖掉这个方法即可，这样代码就变得简单清晰。

ChannelHandlerAdapter抽象类提供了两个子类ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter用于针对入站事件
、出站事件的处理。其中ChannelInboundHandlerAdapter实现了ChannelInboundHandler接口，而ChannelOutboundHandlerAdapter实现了ChannelOutboundHandler接口。

在实际开发过程中，我们的自定义的ChannelHandler多数是继承自ChannelInboundHandlerAdapter和ChannelOutboundHandlerAdapter类或者是这两个类的子类。比如ByteToMessageDecoder、MessageToMessageDecoder、MessageToByteEncoder、MessageToMessageEncoder等，就是这两个类的子类。

# ChannelOutboundBuffer

flush操作负责将ByteBuffer消息写入到SocketChannel中发送给对方

```java
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println(ctx.channel().remoteAddress() + "->Server :" + msg.toString());
        ctx.write(msg);
        ctx.flush();
    }
}
```

在上述示例中，先是将数据通过write方法写入到ChannelHandlerContext，而后在调用flush执行发送。当然，Netty也提供了writeAndFlush方法，用于将这两个方法合二为一。那么为什么发送数据需要经过两个步骤呢？

write和flush两者作用概括如下：

- write：将需要写的ByteBuff存储到ChannelOutboundBuffer中。
- flush：从ChannelOutboundBuffer中将需要发送的数据读出来通过Channel发送出去。

ChannelOutboundBuffer类主要用于存储其待处理的出站写请求的内部数据。当Netty调用write时数据不会真正的去发送而是写入到ChannelOutboundBuffer缓存队列，直到调用flush方法Netty才会从ChannelOutboundBuffer取数据发送。每个Unsafe都会绑定一个ChannelOutboundBuffer，也就是说每个客户端连接上服务端都会创建一个ChannelOutboundBuffer绑定客户端Channel。Netty设计ChannelOutboundBuffer是为了减少TCP缓存的压力提高系统的吞吐率。

部分源码如下：

```java
public final class ChannelOutboundBuffer {
    // The Entry that is the first in the linked-list structure that was flushed
    private Entry flushedEntry;
    // The Entry which is the first unflushed in the linked-list structure
    private Entry unflushedEntry;
    // The Entry which represents the tail of the buffer
    private Entry tailEntry;
    // The number of flushed entries that are not written yet
    private int flushed;

    //...
}
```

- flushedEntry：表示待发送数据起始节点
- unflushedEntry：表示暂存数据起始节点
- tailEntry：表示尾节点
- flushed：表示待发送数据个数

三个Entry 类型flushedEntry、unflushedEntry、tailEntry的转换过程如下：
`Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)`

flushedEntry（包括）到unflushedEntry之间的就是待发送数据，unflushedEntry（包括）到tailEntry就是暂存数据，flushed就是待发送数据个数。正常情况下待发送数据发送完成后会flushedEntry指向unflushedEntry位置，并将unflushedEntry置空。

ChannelOutboundBuffer主要提供了以下方法：

- addMessage方法，功能是添加数据到队列的队尾。
- addFlush方法，准备待发送的数据，在flush前需要调用。
- nioBuffers方法，获取待发送数据，发送数据的时候需要调用拿数据。
- removeBytes方法，发送完成后需要调用删除已经写入TCP缓存成功的数据。
## addMessage
```java
    public void addMessage(Object msg, int size, ChannelPromise promise) {
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        if (tailEntry == null) {
            flushedEntry = null;
        } else {
            Entry tail = tailEntry;
            tail.next = entry;
        }
        tailEntry = entry;
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }

        incrementPendingOutboundBytes(entry.pendingSize, false);
    }

    static Entry newInstance(Object msg, int size, long total, ChannelPromise promise) {
        Entry entry = RECYCLER.get();
        entry.msg = msg;
        entry.pendingSize = size + CHANNEL_OUTBOUND_BUFFER_ENTRY_OVERHEAD;
        entry.total = total;
        entry.promise = promise;
        return entry;
    }

    private static final ObjectPool<Entry> RECYCLER = ObjectPool.newPool(new ObjectCreator<Entry>() {
        @Override
        public Entry newObject(Handle<Entry> handle) {
            return new Entry(handle);
        }
    });
```
上述源码流程如下:
1. 将消息数据包装成Entry对象。
2. 如果队列为空，直接设置尾节点为当前节点，否则将新节点放尾部。
3. unflushedEntry为空说明不存在暂时不需要发送的节点，当前节点就是第一个暂时不需要发送的节点。
4. 将消息添加到未刷新的数组后，增加挂起的字节。

其中Recycler类型的基于线程本地堆栈的轻量级对象池。这意味着，调用newInstance方法时，并不是直接创建了一个Entry实例，而是通过从对象池获取的。

## addFlush
addFlush方法是在系统调用flush方法的时候调用的
```java
    public void addFlush() {
        Entry entry = unflushedEntry;
        if (entry != null) {
            if (flushedEntry == null) {
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    int pending = entry.cancel();
                    decrementPendingOutboundBytes(pending, false, true);
                }
                entry = entry.next;
            } while (entry != null);

            unflushedEntry = null;
        }
    }
```
以上方法的主要功能就是将暂存数据节点变成待发送节点，即flushedEntry指向的节点到unflushedEntry指向的节点（不包含unflushedEntry）之间的数据。

上述源码流程如下：
1. 先获取unflushedEntry指向的暂存数据的起始节点
2. 将待发送数据起始指针flushedEntry指向暂存起始节点
3. 通过promise.setUncancellable()锁定待发送数据，反正发送过程中取消，如果锁定过程中发现其节点已经取消，则调用entry.cancel()取消节点发送，并减少待发送的总字节数。\

## nioBuffers
nioBuffers方法是在系统调用addFlush方法完成后调用
```java
    public ByteBuffer[] nioBuffers(int maxCount, long maxBytes) {
        assert maxCount > 0;
        assert maxBytes > 0;
        long nioBufferSize = 0;
        int nioBufferCount = 0;
        final InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        ByteBuffer[] nioBuffers = NIO_BUFFERS.get(threadLocalMap);
        Entry entry = flushedEntry;
        while (isFlushedEntry(entry) && entry.msg instanceof ByteBuf) {
            if (!entry.cancelled) {
                ByteBuf buf = (ByteBuf) entry.msg;
                final int readerIndex = buf.readerIndex();
                final int readableBytes = buf.writerIndex() - readerIndex;

                if (readableBytes > 0) {
                    if (maxBytes - readableBytes < nioBufferSize && nioBufferCount != 0) {
                        break;
                    }
                    nioBufferSize += readableBytes;
                    int count = entry.count;
                    if (count == -1) {
                        entry.count = count = buf.nioBufferCount();
                    }
                    int neededSpace = min(maxCount, nioBufferCount + count);
                    if (neededSpace > nioBuffers.length) {
                        nioBuffers = expandNioBufferArray(nioBuffers, neededSpace, nioBufferCount);
                        NIO_BUFFERS.set(threadLocalMap, nioBuffers);
                    }
                    if (count == 1) {
                        ByteBuffer nioBuf = entry.buf;
                        if (nioBuf == null) {
                            entry.buf = nioBuf = buf.internalNioBuffer(readerIndex, readableBytes);
                        }
                        nioBuffers[nioBufferCount++] = nioBuf;
                    } else {
                        nioBufferCount = nioBuffers(entry, buf, nioBuffers, nioBufferCount, maxCount);
                    }
                    if (nioBufferCount >= maxCount) {
                        break;
                    }
                }
            }
            entry = entry.next;
        }
        this.nioBufferCount = nioBufferCount;
        this.nioBufferSize = nioBufferSize;
        
        return nioBuffers;
    }
```
以上方法的主要功能就是将要发送数据并转成Java原生的ByteBuffer数组类型。ByteBuffer数组这里是相同线程共享的，也就是说一个客户端跟服务端通讯会使用相同的ByteBuffer数组来发送数据，这样减少了空间创建和销毁时间消耗

上述源码流程如下：
1. 调用NIO_BUFFERS.get获取原生ByteBuffer数组，这里的ByteBuffer是相同线程共享的。
2. 从待发送数据起始节点开始循环处理数据，直至处理到unflushedEntry指向的Entry，或者到最后或者累计的发送字节数大于Integer.MAX_VALUE。
3. 处理跳过被关闭的节点。
4. 如果ByteBuffer数组过小则进行扩容。
5. 将ByteBuf转成ByteBuffer类型存入ByteBuffer数组。
6. 处理下个节点。

## removeBytes
removeBytes方法是在系统调用nioBuffers方法并完成发送后调用
```java
    public void removeBytes(long writtenBytes) {
        for (;;) {
            Object msg = current();
            if (!(msg instanceof ByteBuf)) {
                assert writtenBytes == 0;
                break;
            }

            final ByteBuf buf = (ByteBuf) msg;
            final int readerIndex = buf.readerIndex();
            final int readableBytes = buf.writerIndex() - readerIndex;

            if (readableBytes <= writtenBytes) {
                if (writtenBytes != 0) {
                    progress(readableBytes);
                    writtenBytes -= readableBytes;
                }
                remove();
            } else {
                if (writtenBytes != 0) {
                    buf.readerIndex(readerIndex + (int) writtenBytes);
                    progress(writtenBytes);
                }
                break;
            }
        }
        clearNioBuffers();
    }
```

以上方法的主要功能就是移除已经发送成功的数据，移除的数据是从flushedEntry指向的节点开始遍历链表移除，移除数据分2种情况：
1. 第一种就是当前整个节点的数据已经发送成功，这种情况的做法就是将整个节点移除即可。
2. 第二种就是当前节点部分发送成功，这种情况的做法就是将当前节点的可发送字节数缩短，比如说当前节点有100kb，只发送了30kb，那就将此节点缩短至70kb。

上述源码流程如下：

1. 获取flushedEntry指向的节点数据。
2. 计算整个节点的数据字节长度。
3. 如果当前整个节点的数据已经发送成功将整个节点移除，否则将当前节点的可发送字节数缩短。
4. 清理ByteBuffer数组。
5. 处理下个节点。