---
title: Netty Thrift方式通信编解码源码解读
date: 2018-03-25 11:11:12
tags:
toc: true
categories: Netty
---
# 介绍
## 零拷贝
Netty的“零拷贝”主要体现以下几个方面(<a href="http://www.infoq.com/cn/articles/netty-high-performance?utm_source=infoq&utm_medium=popular_links...">参考</a>)：
1.Netty的接收和发送ByteBuffer采用DIRECT BUFFERS，使用堆外直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中(内存拷贝)，然后才写入Socket中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
2.Netty 提供了 CompositeByteBuf 类, 它可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf, 避免了传统通过内存拷贝的方式将几个小Buffer合并成一个大的Buffer。
3.通过 FileRegion 包装的FileChannel.tranferTo方法 实现文件传输, 可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环write方式导致的内存拷贝问题。
4.通过 wrap 操作, 我们可以将 byte[] 数组、ByteBuf、ByteBuffer等包装成一个 Netty ByteBuf 对象, 进而避免了拷贝操作。

## 编解码处理器
编码与解码器原理相同, 只是做的工作相反, 这里以分析解码器ChannelInboundHandlerAdapter为例
解码处理器目前接触比较多的两种:
+ ByteToMessageDecoder
ByteToMessageDecoder解码器主要将接收的byte位按照定义的帧的结构从原始byte中解析出来, 成为一个个独立的Message(帧/数据报), 常见的比如LengthFieldBasedFrameDecoder。
+ MessageToMessageDecoder
MessageToMessageDecoder解码器主要将一个个独立的独立的Message, 根据定义的解码规则, 赋予具体的寓意, 比如将整个byte解析成string类型(StringDecoder)等。

## 代码引入
需要再次强调的是, 此时pipeline链上的处理上下文: HeadContext-> EncoderContext->DecoderContext->SelfCustemHanderContext->TailContext.
在<a href="https://kkewwei.github.io/elasticsearch_learning/2018/01/22/NioEventLoop%E7%AF%87/">NioEventLoop篇</a>说到, 关于IO SelectionKey.OP_READ类型的任务, 当接收到了数据, 会从unsafe.read()进入到如下代码中(实际调用NioByteUnsafe.read()):
```
        public final void read() {
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();//// allocHandle主要用于预估本次ByteBuf的初始大小，避免分配太多导致浪费或者分配过小放不下单次读取的数据而需要多次读取
            allocHandle.reset(config);
            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) { // 未读取到数据则直接释放该ByteBuf,如果返回-1表示读取出错，后面会关闭该连接
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        break;
                    }
                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());
                allocHandle.readComplete();//记录本次读取到的数据长度（用于计算下次分配ByteBuf时的初始化大小）
                pipeline.fireChannelReadComplete();// 本轮数据读取完毕
                if (close) {// 如果读取的时候发生错误则关闭连接
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {

            }
        }
    }
 ```
这里分配的内存是直接内存。当读取完一次数据后, 通过pipeline.fireChannelReadComplete()向下传递, HeadContext做的事仅仅是找到下一个属性为IN的Context(EncoderContext). 一般对应的handler为ByteToMessageDecoder类解码器, 本文以LengthFieldBasedFrameDecoder来分析。
# ByteToMessageDecoder
属性cumulation存放的是之前没有解析完成的数据, 作为缓存和下次接收的数据一起解析。
回到代码里, 需要关注channelRead:
```
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {    //PoolUnsafeDirectByteBuf
            CodecOutputList out = CodecOutputList.newInstance();//创建解码消息List存放集合
            try {
                ByteBuf data = (ByteBuf) msg;  //data = PoolUnsafeDirectByteBuf
                first = cumulation == null;
                if (first) {
                    cumulation = data;
                } else {
                    cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data); // cumulator = MERGE_CUMULATOR
                }
                callDecode(ctx, cumulation, out);
            } catch (DecoderException e) {
                throw e;
            } catch (Throwable t) {
                throw new DecoderException(t);
            } finally {//如果累积对象中没有数据了(因为所有发送的数据刚刚好n个msg)
                if (cumulation != null && !cumulation.isReadable()) {
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                } else if (++ numReads >= discardAfterReads) {
                    // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                    // See https://github.com/netty/netty/issues/4275
                    numReads = 0;
                    discardSomeReadBytes();
                }

                int size = out.size();
                decodeWasNull = !out.insertSinceRecycled();
                fireChannelRead(ctx, out, size); //针对解析后的out结果，逐个调用message
                out.recycle();
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }
```
主要做的事情:
1) 首先判断msg是否为ByteBuf: 若不是, 则说明此轮传递的不是数据解码, 继续向外传递。
2) 如果cumulation为空, 说明之前解析的帧与数据长度恰好吻合, 没有剩余数据需要下次拼接解析的, 否则, 需要将上次剩余的cumulation与新接收的ByteBuf合成一个新的ByteBuf继续解析。合成器默认为MERGE_CUMULATOR。
```
 @Override
        public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {  //alloc: PooledByteBufAllocator(directe:true) , cumulation = PooledUnsafeDirectByteBuf
            final ByteBuf buffer;
            if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                    || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
                // Expand cumulation (by replace it) when either there is not more room in the buffer
                // or if the refCnt is greater then 1 which may happen when the user use slice().retain() or
                // duplicate().retain() or if its read-only.
                //
                // See:
                // - https://github.com/netty/netty/issues/2327
                // - https://github.com/netty/netty/issues/1764
                buffer = expandCumulation(alloc, cumulation, in.readableBytes());
            } else {
                buffer = cumulation;
            }
            buffer.writeBytes(in); //将新的数据，写入这个cumulation
            in.release();  //释放资源
            return buffer;
        }
```
+ 首先判断目前的cumulation最大容器能否装的下即将合成的缓存, 实际上cumulation.maxCapacity()的取值非常大(2147483647), 如果装不下的, 需要申请新的缓存区域:expandCumulation
```
   ByteBuf oldCumulation = cumulation;
        cumulation = alloc.buffer(oldCumulation.readableBytes() + readable);//重新生成一个新的缓存区， 注意这里的参数是长度，而没有数据的数据
        cumulation.writeBytes(oldCumulation);  //会跑到AbstractByteBuf.writeBytes()里面，向新的cumulation写回旧的数据
        oldCumulation.release(); //释放旧的缓冲区
        return cumulation;
```
cumulation = alloc.buffer(size)可以看出是新生成的缓存与之前缓存区域毫不相关(根据size申请的), 会将新旧缓存放入同一个最新的缓存cumulation。
3) 解码callDecode
```
             while (in.isReadable()) {
                int outSize = out.size();
                if (outSize > 0) { //out为经过转码形成帧的的数据
                    fireChannelRead(ctx, out, outSize);//每当读取到帧了，就会立刻向上发送解析好的帧，看情况解析出来一个，发送一个
                    out.clear();
                    outSize = 0;
                }
                int oldInputLength = in.readableBytes();  //24
                decodeRemovalReentryProtection(ctx, in, out);  //这里会循环的调用解码decode
                if (outSize == out.size()) { //decode没有解析出东西
                    if (oldInputLength == in.readableBytes()) { //没有读取到任何东西，可能帧显示的长度大于实际的位数，没有数据了, 需要下次接受的数据补齐
                        break;
                    } else { //还是向前消费了许多东西，可能读到了坏的帧，丢弃了
                        continue;
                    }
                }
                if (oldInputLength == in.readableBytes()) {  //说明outSize < out.size(),读取到新的帧了，但是指针还没有向前进，哪里有问题
                    throw new DecoderException(
                            StringUtil.simpleClassName(getClass()) +
                                    ".decode() did not read anything but decoded a message.");
                }
              }
```
+ 首先查看是否解析出来了数据报(帧), 若解析出来了, 则通过fireChannelRead向上传递。
+ 开始这轮真正的数据解析工作, decodeRemovalReentryProtection里面需要注意decode函数, 在`LengthFieldBasedFrameDecoder`里实现。
+ 对这轮解析结果进行分析:
     若没有解析出数据, 说明缓存区域没有消费数据, 显示的帧长度大于实际拥有的数据量, 此时会将数据缓存起来放入cumulation, 等待下次接收到数据后一起解析。
     若没有解析出数据, 说明可能存在损坏的帧, 解码时候把废弃的帧给丢弃了。
     若解析出来的数据, 但是却没有消费数据, 说明出现了问题, 向外抛出异常。
4) 检查是否还有帧可以继续向上传递。
```
 static void fireChannelRead(ChannelHandlerContext ctx, List<Object> msgs, int numElements) {
        if (msgs instanceof CodecOutputList) {   //都单个单个的发送
            fireChannelRead(ctx, (CodecOutputList) msgs, numElements);
        } else {
            for (int i = 0; i < numElements; i++) {
                ctx.fireChannelRead(msgs.get(i));
            }
        }
    }
```
可以看出实际也是每个帧单独向上发送的。

# LengthFieldBasedFrameDecoder
LengthFieldBasedFrameDecoder作为ByteToMessageDecoder的父类, 它只用定义具体的规则, 如何拆分byte成为每一个个数据报(帧), 也就是只用实现protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)即可: 从原始byte in中解析出一个帧, 放入out中。
我们需要理解几个属性的函数:
+ maxFrameLength: 定义了每个帧的最大长度, 超过此长度的帧将作为废弃数据直接丢弃掉。
+ lengthFieldOffset:  帧长度位置的偏移量(起始位置), 情况:帧的第一个部分不是长度字段。
+ lengthFieldLength: 定义帧长度的字段本身的长度。
+ lengthAdjustment:  数据内容长度需需要调整的长度, 情况: 帧的长度还可能包含了部分不属于帧内容字段长度, 需要把这部分长度给去掉
+ initialBytesToStrip:  解析时候, 需要跳过的长度, 以进入到帧的数据部分
+ failFast: 当解析出的帧内容长度大于阈值, 是否立即抛出异常,默认为false, 建议不要修改。 当设置为true后, 把该帧全部内容丢弃后再抛出异常。
+ discardingTooLongFrame: 当帧解析出来的帧内容长度 > maxFrameLength时,并且剩余缓存可读字段 < 解析的帧长度, 需要discardingTooLongFrame置位true,  意味着下次接收的帧需要继续丢弃,当前帧处于丢弃模式。
+ tooLongFrameLength: 超过长度限制maxFrameLength的帧长度
+ bytesToDiscard: 对于下次接收的数据中需要继续丢弃的byte长度。 当接收的帧长度大于阈值, 会丢弃该帧及内容
关系如下:<img src="https://kkewwei.github.io/elasticsearch_learning/img/Thrift%E5%B8%A7.png"/>
也可<a href="https://blog.csdn.net/u010853261/article/details/55803933"> 参考/a>
其中:
+ head1和head2可以由用户自定义语义。
+ 有的人会想, initialBytesToStrip、lengthFieldOffset、lengthFieldLength这三个属性有一定的关系, 还为啥会当把三个参数都传递进来, 我想设计者是为了给使用者更大的灵活性,比如 lengthFieldLength后面专门空几个byte啥都不干放着也是行的, 一般initialBytesToStrip = lengthFieldOffset+lengthFieldLength
解码函数如下:
```
       if (discardingTooLongFrame) {//如果当前的编码器处于丢弃超长帧的状态，上一个包最后一个帧还有东西要丢弃，要对当前包接着丢
            long bytesToDiscard = this.bytesToDiscard; //获取需要丢弃的长度
            int localBytesToDiscard = (int) Math.min(bytesToDiscard, in.readableBytes());//丢弃的长度不能超过当前缓冲区可读的字节数
            in.skipBytes(localBytesToDiscard);//跳过需要忽略的字节长度
            bytesToDiscard -= localBytesToDiscard;////bytesToDiscard减去已经忽略的字节长度
            this.bytesToDiscard = bytesToDiscard; //下轮还需要忽略的长度
            failIfNecessary(false);
        }
        //对当前缓冲区中可读字节数和长度偏移量进行对比，如果小于偏移量，谁明缓冲区数据报内容没有，直接返回
        if (in.readableBytes() < lengthFieldEndOffset) {//数据报内数据不够，返回null，由IO线程继续读取数据，此轮不解码
            return null; //当前帧没有value
        }
       // 拿到长度字段的起始偏移量index
        int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;  //长度域终点位置
        long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);/// 拿到实际的未调整过的内容长度
        if (frameLength < 0) {
            in.skipBytes(lengthFieldEndOffset);
            throw new CorruptedFrameException(
                    "negative pre-adjustment length field: " + frameLength);
        }
        // frameLength = (head1_length+length_length)(lengthFieldEndOffset)+head2_length(lengthAdjustment)+content_length(frameLength)
        frameLength += lengthAdjustment + lengthFieldEndOffset;
        if (frameLength < lengthFieldEndOffset) {
            in.skipBytes(lengthFieldEndOffset);//当前帧忽略过
            throw new CorruptedFrameException(
                    "Adjusted frame length (" + frameLength + ") is less " +
                    "than lengthFieldEndOffset: " + lengthFieldEndOffset);
        }
        // 数据帧长长度超出最大帧长度，说明这个帧当前帧不合法， 需要丢弃当前帧，跳到包里下一个帧里面。
        if (frameLength > maxFrameLength) {
            long discard = frameLength - in.readableBytes();//前面
            tooLongFrameLength = frameLength;
            // 当前可读字节已达到frameLength，直接跳过frameLength个字节，丢弃之后，后面有可能就是一个合法的数据包
            if (discard < 0) {// // 当前可读字节已达到frameLength，直接跳过frameLength个字节，丢弃之后，后面有可能就是一个合法的数据包
                // buffer contains more bytes then the frameLength so we can discard all now
                in.skipBytes((int) frameLength);//丢弃当前不合法帧，直接跳到包里下一个帧里面
            } else {
                // Enter the discard mode and discard everything received so far.
                discardingTooLongFrame = true;//下个报接着丢上一个报最后一个帧
                bytesToDiscard = discard;
                in.skipBytes(in.readableBytes());//丢弃整个帧
            }
            failIfNecessary(true);
            return null;
        }
        // never overflows because it's less than maxFrameLength
        int frameLengthInt = (int) frameLength;
        if (in.readableBytes() < frameLengthInt) {  //什么都没有读取到，而且in指针也没有向前去，后面将退出，不会在继续循环
            return null;
        }
        if (initialBytesToStrip > frameLengthInt) {
            in.skipBytes(frameLengthInt);
            throw new CorruptedFrameException(
                    "Adjusted frame length (" + frameLength + ") is less " +
                    "than initialBytesToStrip: " + initialBytesToStrip);
        }
        in.skipBytes(initialBytesToStrip); //这段值已经读取出来了（长度），后续都是head2+content
        // extract frame
        int readerIndex = in.readerIndex();
        int actualFrameLength = frameLengthInt - initialBytesToStrip;
        ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
        in.readerIndex(readerIndex + actualFrameLength);  //设置可读位置
        return frame;
```
主要操作如下:
1) 如果当前处于丢弃模式(discardingTooLongFrame), 若是,那么继续丢弃还需要丢弃的byte, 并且检查是否该抛出异常:
```
private void failIfNecessary(boolean firstDetectionOfTooLongFrame) {
        if (bytesToDiscard == 0) { //不是第一次遇到，就是说下次编码的时候不需要丢弃了，说明这个超长帧读取完毕，那么将这个编码器的状态设置为非丢弃超长帧状态
            // Reset to the initial state and tell the handlers that
            // the frame was too large.
            long tooLongFrameLength = this.tooLongFrameLength; //超过限制的帧长度
            this.tooLongFrameLength = 0; //主要是复位
            discardingTooLongFrame = false;
            if (!failFast || firstDetectionOfTooLongFrame) {// 如果没有设置快速失败，或者设置了快速失败并且是第一次检测到大包错误，抛出异常，让handler去处理
                fail(tooLongFrameLength); //bytesToDiscard，最起码不是第一次，bytesToDiscard为0.直接丢弃
            }
        } else {  //第一次遇到，发现当前帧长度太长了
            // Keep discarding and notify handlers if necessary.
            if (failFast && firstDetectionOfTooLongFrame) {
                fail(tooLongFrameLength); //是第一次遇到，直接丢失
            }
        }
    }
```
+ 当上一个帧需要丢弃content全部丢弃完了, 那么就直接抛出异常。failFast肯定为false,因为bytesToDiscard, 就说明此次不是最开始遇见超过阈值长度的帧。
+ 反之, 说明是首次发现帧太长了, 需要丢弃。failFast肯定为true。
2) 若当前缓存可读byte < 长度偏移量, 直接退出继续, 数据仍然放在了缓存。
3) 计算出帧整体的长度,包括了length + head2 + content:
```
frameLength += lengthAdjustment + lengthFieldEndOffset;
```
4) 检查帧frameLength是否超过的阈值,若超过了:
+ 检查当前缓存可读数据是否够length长度丢弃, 若够的话, 缓存可读区域向前移动frameLength长度
+ 否则, 进入丢弃模式: discardingTooLongFrame设置为true、记录下次需要丢弃的长度。
并运行failIfNecessary, 检查是现在立刻抛出异常, 还是等下轮丢弃完再抛。
5) 检查当前缓存可读长度是否超过frameLength, 若没有的话, 说明当前帧长度超过了发送的长度限制(默认1024bit), 当前帧被多次发送了, 这里解析函数就直接退出。下次接收的数据会自动累加到当前可读数据上,等待下次再解析出这个帧。
6) 跳过initialBytesToStrip, 并开始读取相应的帧内容, 并向上传递该帧内容。
