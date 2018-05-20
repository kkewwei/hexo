---
title: Netty Http通信源码二(编码)阅读
date: 2018-05-04 00:02:39
tags:
---
解码过程仍以<a href="https://kkewwei.github.io/elasticsearch_learning/2018/04/16/Netty-Http%E9%80%9A%E4%BF%A1%E8%A7%A3%E7%A0%81%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/">Netty Http通信源码一(解码)阅读</a>提供的示例为例, 编码发送的主体DefaultFullHttpResponse如下:
<img src="http://owsl7963b.bkt.clouddn.com/DefaultFullHttpResponse.png" />
涉及到的ChannelOutboundHandler类有:HttpContentCompressor、HttpObjectEncoder, 及其父类。 本wiki仍然以数据的流向作为引导线。
# HttpContentCompressor及父类HttpContentEncoder、MessageToMessageCodec
我们需要知道: MessageToMessageCodec该类是一个ChannelDuplexHandler类型的, 可以同时在IN, OUT场景下使用。
首先进入的是MessageToMessageCodec的write()函数, 通过该函数的encoder.write(ctx, msg, promise)跳转到MessageToMessageEncoder的write()函数中, 实现如下:
```
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        CodecOutputList out = null;
        try {
            if (acceptOutboundMessage(msg)) {
                out = CodecOutputList.newInstance();
                I cast = (I) msg;
                try {
                    encode(ctx, cast, out);
                } finally {
                    ReferenceCountUtil.release(cast);
                }
                if (out.isEmpty()) {
                    out.recycle();
                    out = null;
                    throw new EncoderException(
                            StringUtil.simpleClassName(this) + " must produce at least one message.");
                }
            } else {
                ctx.write(msg, promise);
            }
        }
        } finally {
            if (out != null) {
                final int sizeMinusOne = out.size() - 1;
                if (sizeMinusOne == 0) {
                    ctx.write(out.get(0), promise);
                } else if (sizeMinusOne > 0) {
                    // Check if we can use a voidPromise for our extra writes to reduce GC-Pressure
                    // See https://github.com/netty/netty/issues/2525
                    ChannelPromise voidPromise = ctx.voidPromise();
                    boolean isVoidPromise = promise == voidPromise;
                    for (int i = 0; i < sizeMinusOne; i ++) {//分开向下发送
                        ChannelPromise p;
                        if (isVoidPromise) {
                            p = voidPromise;
                        } else {
                            p = ctx.newPromise();
                        }
                        ctx.write(out.getUnsafe(i), p);
                    }
                    ctx.write(out.getUnsafe(sizeMinusOne), promise);
                }
                out.recycle();
            }
        }
    }
```
实现也很简单,主要做了如下两件事:
1. 首先通过encode()进行编码, encode()是在HttpContentEncoder中实现的。; 若out没有编码输出, 则直接抛出异常;最终通过msg.release()释放response.content占用的空间。
2. 针对编码输出out, 循环遍历每一个compoment, 通过DefalueChannalHadlerContext.write()向外写出数据。

## HttpContentEncoder的encode()函数
首先需要了解decode(), 在写入的时候, 将header里面的accept-encoding属性取值赋给acceptEncodingQueue, 这样编码的时候就知道需要使用什么编码器了, 本wiki在使用时, 客户端发送的编码器: "gzip,deflat,br"

endoce函数如下, 其中msg为DefaultFullHttpResponse, 包含了header和content部分
```
@Override    //msg: DefaultFullHttpResponse
    protected void encode(ChannelHandlerContext ctx, HttpObject msg, List<Object> out) throws Exception {
        final boolean isFull = msg instanceof HttpResponse && msg instanceof LastHttpContent;
        switch (state) {
            case AWAIT_HEADERS: {  //初始取值
                ensureHeaders(msg);
                assert encoder == null;

                final HttpResponse res = (HttpResponse) msg;
                 //根据返回结果确定是否需要编码
                final int code = res.status().code();
                final CharSequence acceptEncoding;
                if (code == CONTINUE_CODE) { //continue_code
                    // We need to not poll the encoding when response with CONTINUE as another response will follow
                    // for the issued request. See https://github.com/netty/netty/issues/4079
                    acceptEncoding = null;
                } else {
                    // Get the list of encodings accepted by the peer.
                    acceptEncoding = acceptEncodingQueue.poll(); //"gzip.default.br"
                    if (acceptEncoding == null) {
                        throw new IllegalStateException("cannot send more responses than requests");
                    }
                }
                /*
                 * per rfc2616 4.3 Message Body
                 * All 1xx (informational), 204 (no content), and 304 (not modified) responses MUST NOT include a
                 * message-body. All other responses do include a message-body, although it MAY be of zero length.
                 *
                 * 9.4 HEAD
                 * The HEAD method is identical to GET except that the server MUST NOT return a message-body
                 * in the response.
                 *
                 * Also we should pass through HTTP/1.0 as transfer-encoding: chunked is not supported.
                 *
                 * See https://github.com/netty/netty/issues/5382
                 */
                if (isPassthru(res.protocolVersion(), code, acceptEncoding)) { //是否接下来是没有body的
                    if (isFull) {
                        out.add(ReferenceCountUtil.retain(res));
                    } else {
                        out.add(res);
                        // Pass through all following contents.
                        state = State.PASS_THROUGH;
                    }
                    break;
                }
                if (isFull) {
                    // Pass through the full response with empty content and continue waiting for the the next resp.
                    if (!((ByteBufHolder) res).content().isReadable()) {
                        out.add(ReferenceCountUtil.retain(res));
                        break;
                    }
                }

                // Prepare to encode the content.   通过curl 发送的请求中是没有压缩的，为identity
                final Result result = beginEncode(res, acceptEncoding.toString());

                // If unable to encode, pass through.
                if (result == null) {
                    if (isFull) {
                        out.add(ReferenceCountUtil.retain(res));
                    } else {
                        out.add(res);
                        // Pass through all following contents.
                        state = State.PASS_THROUGH;
                    }
                    break;
                }

                encoder = result.contentEncoder(); //encoder = EmbeddedChannel

                // Encode the content and remove or replace the existing headers
                // so that the message looks like a decoded message.
                res.headers().set(HttpHeaderNames.CONTENT_ENCODING, result.targetContentEncoding()); //gzip

                // Output the rewritten response.
                if (isFull) {
                    // Convert full message into unfull one.
                    HttpResponse newRes = new DefaultHttpResponse(res.protocolVersion(), res.status());
                    newRes.headers().set(res.headers());
                    out.add(newRes);  //newRes里面还没有放数据

                    ensureContent(res);
                    encodeFullResponse(newRes, (HttpContent) res, out);
                    break;
                } else {
                    // Make the response chunked to simplify content transformation.
                    res.headers().remove(HttpHeaderNames.CONTENT_LENGTH);
                    res.headers().set(HttpHeaderNames.TRANSFER_ENCODING, HttpHeaderValues.CHUNKED);

                    out.add(res);
                    state = State.AWAIT_CONTENT;
                    if (!(msg instanceof HttpContent)) {
                        // only break out the switch statement if we have not content to process
                        // See https://github.com/netty/netty/issues/2006
                        break;
                    }
                    // Fall through to encode the content
                }
            }
            case AWAIT_CONTENT: {
                ensureContent(msg);
                if (encodeContent((HttpContent) msg, out)) {
                    state = State.AWAIT_HEADERS;
                }
                break;
            }
            case PASS_THROUGH: {
                ensureContent(msg);
                out.add(ReferenceCountUtil.retain(msg));
                // Passed through all following contents of the current response.
                if (msg instanceof LastHttpContent) {
                    state = State.AWAIT_HEADERS;
                }
                break;
            }
        }
    }
```
该编码器encode主要做的事情:
1. 根据state初始值AWAIT_HEADERS(默认)首先AWAIT_HEADERS分支, 获取result_code:
+ 若为100, 说明之时一个continue信号, acceptEncoding赋值为空, 告诉后面不用编码该返回。
+ 否则, 根据encode()获取可接受编码accept-encoding: gzip,deflat,br
2. 根据规范code返回值若为All 1xx (informational), 204 (no content), and 304 (not modified), 一定不能包含message-body部分。继续检查result_code, 若是该类code, 直接将将out.add(res)退出, 而不用考虑对content部分进行压缩。
3. 检查contet部分是否还有未读数据, 没有的话直接放入out.add(res), 也是不用继续压缩。
4. 在beginEncode中建立相应压缩管道EmbeddedChannel:
```
 protected Result beginEncode(HttpResponse headers, String acceptEncoding) throws Exception {
        ZlibWrapper wrapper = determineWrapper(acceptEncoding);//GZIP
        if (wrapper == null) {
            return null;
        }
        String targetContentEncoding;
        switch (wrapper) {
        case GZIP:
            targetContentEncoding = "gzip";
            break;
        case ZLIB:
            targetContentEncoding = "deflate";
            break;
        default:
            throw new Error();
        }

        return new Result(
                targetContentEncoding,
                new EmbeddedChannel(ctx.channel().id(), ctx.channel().metadata().hasDisconnect(),
                        ctx.channel().config(), ZlibCodecFactory.newZlibEncoder(
                        wrapper, compressionLevel, windowBits, memLevel)));
    }
```
主要做了如下事情,
+ 首先在determineWrapper判断使用哪种压缩编码, 使用优先级gzip>deflate
+ 返回EmbeddedChannel, 我们需要注意该channel里面通过ZlibCodecFactory.newZlibEncoder()方式添加了一个handler, 该返回EmbeddedChannel的pipeline结构如下:<img src="http://owsl7963b.bkt.clouddn.com/GzipPipline.png" />
对gzip编码感兴趣的话, 可以看下JdkZlibEncoder.encode关于编码的细节。
5. 向返回值headlerz中添加 content-encoding:gzip
6. 封装header, result_code, http_version, 产生一个DefaultHttpResponse, 放入out.
7. 在encodeFullResponse中调用编码函数encodeContent()
```
private boolean encodeContent(HttpContent c, List<Object> out) {
        ByteBuf content = c.content();
        encode(content, out);
        if (c instanceof LastHttpContent) {
            finishEncode(out);
            LastHttpContent last = (LastHttpContent) c;
            // Generate an additional chunk if the decoder produced
            // the last product on closure,
            HttpHeaders headers = last.trailingHeaders();
            if (headers.isEmpty()) {
                out.add(LastHttpContent.EMPTY_LAST_CONTENT);
            } else {
                out.add(new ComposedLastHttpContent(headers));
            }
            return true;
        }
        return false;
    }
```
7.1 注意这里的encode部分, 调用的是 encoder.writeOutbound(in.retain()), 而encoder就是前面描述的EmbeddedChannel, 进去后, 发现调用的是EmbeddedChannel.write(m),  依次处理的handler见上图EmbeddedChannel的pipeline。
+ 调用JdkZlibEncoder.write()进行压缩。
+ 将数据写入ChannelOutboundBuffer对象并刷新, 写入的时候也会受限制于高水位,但是实际并不起什么作用, 后面在真正发送数据的时候会详细讲解这部分。
+ 在finishEncode()中会产生DefaultHttpContent, 里面存放的是gzip压缩的footer, 具体byte见JdkZlibEncoder.finishEncode里面描述。
7.2 向out中写入LastHttpContent.EMPTY_LAST_CONTENT, 代表这个帧内容结束。
这样整个输出帧的内容存放在out中, 拥有的对象如下:
<img src="http://owsl7963b.bkt.clouddn.com/HttpOutPutResponse.png" />
其中:
+ DefaultHttpResponse: 存放的是Http/1.1 status, Header等
+ 第一个DefaultHttpContent存放的是压缩的内容。
+ 第二个DefaultHttpContent存放的是压缩器gzip的尾部标识部分。
+ LastHttpContent代表整个帧的结束, content部分为空。
8. 在encodeFullResponse中, 向header部分添加整个帧的长度content-length属性。
## JdkZlibEncoder编码
我们可以了解下JdkZlibEncoder.write()是怎么编码的
```
@Override
    protected void encode(ChannelHandlerContext ctx, ByteBuf uncompressed, ByteBuf out) throws Exception {
        int len = uncompressed.readableBytes(); //总共刻度数据
        int offset;
        byte[] inAry;
        if (uncompressed.hasArray()) {  //若有数组,直接获得数组
            // if it is backed by an array we not need to to do a copy at all
            inAry = uncompressed.array();
            offset = uncompressed.arrayOffset() + uncompressed.readerIndex();
            // skip all bytes as we will consume all of them
            uncompressed.skipBytes(len); //读取的数据， 直接跳过数组的长度
        } else {
            inAry = new byte[len];
            uncompressed.readBytes(inAry);//将数据读取到这个byte数组中
            offset = 0;
        }
        if (writeHeader) { //将数组写进去， 最开始编码，需要写
            writeHeader = false;
            if (wrapper == ZlibWrapper.GZIP) {
                out.writeBytes(gzipHeader);//首先写进去头
            }
        }
        if (wrapper == ZlibWrapper.GZIP) {
            crc.update(inAry, offset, len);
        }
        //向压缩器中传递带压缩的数组
        deflater.setInput(inAry, offset, len);
        while (!deflater.needsInput()) {
            deflate(out); //进行真正的压缩
        }
    }
```
可以看到:
+ 首先获得bytebuf的byte数组
+ 向最终存放压缩数据的out buf(PooledUnsafeDirectByteBuf)中写入gzip压缩标志的头部gzipHeader: {0x1f, (byte) 0x8b, Deflater.DEFLATED, 0, 0, 0, 0, 0, 0, 0};
我们需要了解的是这个buf长度 =  (int) Math.ceil(msg.readableBytes() * 1.001) + 12 + gzipHeader.len(), 看来极端情况下压缩后可能和压缩前长度差不多;
+ 直接调用gzip的压缩算法, 将byte压缩后写入out中. 至于具体的压缩算法, 感兴趣的同学可以自行查看源代码。


## DefalueChannalHadlerContext.write()
DefalueChannalHadlerContext.write()函数之前的工作主要是编码部分、组成帧。 这里开始将压缩后最终的帧继续向外传递write。
接下来OutHanlder为HttpResponseEncoder, 实际调用的是其父类MessageToMessageEncoder.write(), 该函数已经在最开始介绍了, 调用的是HttpObjectEncoder.encode(), 函数如下:
```
         ByteBuf buf = null;
        if (msg instanceof HttpMessage) {  //如果是头部，则先编码头部
            if (state != ST_INIT) {
                throw new IllegalStateException("unexpected message type: " + StringUtil.simpleClassName(msg));
            }
            H m = (H) msg;
            buf = ctx.alloc().buffer();//直接内存分配的地址
            // Encode the message.
            encodeInitialLine(buf, m); //先是编码initial部分
            encodeHeaders(m.headers(), buf);//再编码header部分
            buf.writeBytes(CRLF);
            state = isContentAlwaysEmpty(m) ? ST_CONTENT_ALWAYS_EMPTY ://一般都是ST_CONTENT_NON_CHUNK
                    HttpUtil.isTransferEncodingChunked(m) ? ST_CONTENT_CHUNK : ST_CONTENT_NON_CHUNK;
        }
        if (msg instanceof ByteBuf && !((ByteBuf) msg).isReadable()) {
            out.add(EMPTY_BUFFER);
            return;
        }
        //如果是数据部分，则编码数据部分， 若是DefaultFullHttpResponse
        if (msg instanceof HttpContent || msg instanceof ByteBuf || msg instanceof FileRegion) {
            switch (state) {
                case ST_INIT:
                    throw new IllegalStateException("unexpected message type: " + StringUtil.simpleClassName(msg));
                case ST_CONTENT_NON_CHUNK: //st_content_non_chunk
                    final long contentLength = contentLength(msg);
                    if (contentLength > 0) {//可写的空间够，直接放到直接内存buf中
                        if (buf != null && buf.writableBytes() >= contentLength && msg instanceof HttpContent) {//必须是content类型的
                            // merge into other buffer for performance reasons
                            buf.writeBytes(((HttpContent) msg).content());
                            out.add(buf);
                        } else {
                            if (buf != null) {
                                out.add(buf); //先把直接内存放进去
                            }
                            out.add(encodeAndRetain(msg));//放进去的是CompositeByteBuf, 可以看出分了两部分放进去
                        }

                        if (msg instanceof LastHttpContent) {
                            state = ST_INIT; //编码完成后，直接复位
                        }
                        break;
                    }
                    // fall-through!
                case ST_CONTENT_ALWAYS_EMPTY:

                    if (buf != null) {
                        // We allocated a buffer so add it now.
                        out.add(buf);
                    } else {
                        // Need to produce some output otherwise an
                        // IllegalStateException will be thrown
                        out.add(EMPTY_BUFFER);
                    }

                    break;
                case ST_CONTENT_CHUNK:
                    if (buf != null) {
                        // We allocated a buffer so add it now.
                        out.add(buf);
                    }
                    encodeChunkedContent(ctx, msg, contentLength(msg), out);
                    break;
                default:
                    throw new Error();
            }
            if (msg instanceof LastHttpContent) { //解码完成，再置位
                state = ST_INIT;
            }
        } else if (buf != null) {
            out.add(buf);
        }
```
state初始值为ST_INIT, 该函数主要做了如下操作:
1. 首先检查是否是HttpMessage, Http Response 结构如上所示, 最开始是DefaultHttpResponse。
+ 通过encodeInitialLine编码initial部分(HHttpResponseEncoder中定义)
```
         response.protocolVersion().encode(buf); //首先存放version编码
        buf.writeByte(SP); //存放byte:32水平空格
        response.status().encode(buf); //存放status, 比如[50 48 48 32 79 79]="200 ok"
        buf.writeBytes(CRLF); //  { CR, LF }回车换行
```
+ 通过encodeHeaders编码header部分, 每个header属性编码如下:
```
         final int nameLen = name.length();
        final int valueLen = value.length();
        final int entryLen = nameLen + valueLen + 4;
        buf.ensureWritable(entryLen);  //检查buf的最小长度
        int offset = buf.writerIndex();
        writeAscii(buf, offset, name); // buf.setCharSequence(offset, value, CharsetUtil.US_ASCII);使用US_ASCII编码
        offset += nameLen;
        buf.setByte(offset ++, ':');//:
        buf.setByte(offset ++, ' ');//空格
        writeAscii(buf, offset, value);
        offset += valueLen;
        buf.setByte(offset ++, '\r');//
        buf.setByte(offset ++, '\n');
        buf.writerIndex(offset);
```
1) 可以看出实际编码后存放的是 key: value\r\n; 注意:后面是空格
2) 通过CharsetUtil.US_ASCII编码key和value
+ 再接着写入[CRLF], 可以看出,每部分内容都是以[CRLF]作为分隔符。
然后根据header部分还改变state状态, 一般state会被置为ST_CONTENT_NON_CHUNK。根据MessageToMessageEncoder.write()可知, 编码完DefaultHttpResponse, 就调用DefalueChannalHadlerContext.write继续向外写, 后面会详细讲些该部分。
2.第二、三次、四次传递过来的是DefaltHttpContent, 将进入ST_CONTENT_NON_CHUNK部分。
+ 该部分, 直接将整个DefaltHttpContent放入out向外写
+ 当发现传递过来的Contnt为末尾标识符LastHttpContent时, 置state=ST_INIT, 等待下一个帧传递过来。

# Netty水位
写数据分为两个过程:write()和flush():
+ write只是将数据放在了缓存ChannelOutboundBuffer中
+ 通过调用channal.write()向网络发送数据。

向外写的最外层为HeadContext, 其write直接调用unsafe.write(msg, promise), 实际调用的是AbstractChannel$AbstractSafeUnSafe.write(), 如下:
```
   @Override
        public final void write(Object msg, ChannelPromise promise) {
            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;//每个管道都有一个高水位和低水位
            int size;
            try {
                msg = filterOutboundMessage(msg); //自定义, 在真正写出的时候, msg必须转变为直接内存heap
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }
            outboundBuffer.addMessage(msg, size, promise);//ChannelOutboundBuffer
        }
```
+ 在这个函数中, 我们需要了解的是: 若直接是最外层发送, 那么filterOutboundMessage将会把msg转变为直接内存buf。
+ 通过ChannelOutboundBuffer.addMessage(msg, size, promise), 将输出结果暂时缓存起来, 形成一个链再批量发送。
我们需要了解下ChannelOutboundBuffer这个类, 它作为输出byte暂时缓存的地方, 维护着该数据byte组成的链, 结构如下:
<img src="http://owsl7963b.bkt.clouddn.com/ChannelOutboundBuffer.png" />
flushEntry 表示即将刷新的其实位置
unflushEntry: 每次调用addFlush()将unflushEntry赋值给flushEntry, 才算真正开始flush数据了。
tailEntry: 当前缓存message链尾部, 新增message都是尾部追加。

添加完message之后, 同时调用incrementPendingOutboundBytes(), 记录当前缓存的数据量:
```
        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);////原子更新一下当前的水位，并获取最新的水位信息
        if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {//如果当前的水位高于配置的高水位，那么就要调用setUnwriteable方法
            setUnwritable(invokeLater);
        }
```
当然向ChannelOutboundBuffer添加content不能太快了, 否则若来不及发送的话, 都是堆积在直接内存中, 容易造成内存OOM, 这里是如何限处理存数据大小的呢?
在netty启动时, 只需要添加如下参数即可:
```
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 64 * 1024);
bootstrap.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 32 * 1024);
```
代表:
+ 当超过高水位64kb时候, 就会调用fireChannelWritabilityChanged函数, 让上游感知, 同时Channel.isWritable()返回false。
+ 当超过高水位之后, 又通过发送后回落到低水位时, Channel.isWritable() 将会返回true.
## setUnwritable设置不可写
```
        for (;;) {
            final int oldValue = unwritable;
            final int newValue = oldValue | 1;
            if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {//高水位的时候就会可以通知到业务handler中的WritabilityChanged方法，并且修改buffer的状态
                if (oldValue == 0 && newValue != 0) {//如果之前的状态是可写，现在的状态是不可写，就要调用pipeline上handerd的lWritabilityChanged方法
                    fireChannelWritabilityChanged(invokeLater);//
                }//事实上，达到高水位之后，Netty仅仅会发送一个Channle状态位变更事件通知，并不会阻止用户继续发送消息.发现的确如此。
                break;
            }
        }
```
这里可以看出使用for循环, 直到将unwritable属性有0变为1(可写->不可写), 然后调用fireChannelWritabilityChanged向上层handler发送信号, 上层可以覆盖该函数, 并通过channelWritable()判断是达到水位上线还是恢复可写了。
我们需要知道, netty本省并不会为我们做限制写入, 它只是负责通知我们达到内存使用水位上限了。 我们需要自己在函数中控制写入数据, 比如在发送数据时, 当且仅当channel.isWritable()为true才继续发送数据。


