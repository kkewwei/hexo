---
title: Netty Http通信源码一(解码)阅读
date: 2018-04-16 00:06:17
tags:
---
首先给出一个http server pipiLine里面的处理器的组成结构的示例:
```
        @Override
        protected void initChannel(Channel ch) throws Exception {
            final HttpRequestDecoder decoder = new HttpRequestDecoder(4096, 8192, 8192);
            decoder.setCumulator(ByteToMessageDecoder.COMPOSITE_CUMULATOR);  //默认是另一个
            ch.pipeline().addLast("decoder", decoder);
            ch.pipeline().addLast("encoder", new HttpResponseEncoder());
            final HttpObjectAggregator aggregator = new HttpObjectAggregator(Math.toIntExact(transport.maxContentLength.getBytes()));
            ch.pipeline().addLast("aggregator", aggregator);  //包的聚合
            ch.pipeline().addLast("encoder_compress", new HttpContentCompressor(transport.compressionLevel));
            ch.pipeline().addLast("pipelining", new HttpPipeliningHandler(transport.logger, transport.pipeliningMaxEvents));
            ch.pipeline().addLast("handler", requestHandler);
        }
```
其中只有HttpRequestDecoder属于ByteToMessageDecoder类型, 主要作用是从byte中拼接处每一个帧, 其余处理器大部分是根据自定义的语义对这个帧转化, 本文将以示例中的重要的handler为处理器, 以POST请求解析过程为串分析下去。
http处理方式是每次将缓冲池放满(默认65536个), 然后将65536个字符按照虚拟的chunk分片(默认一个HttpChunk 8192个字符),通过handler, 最后在HttpObjectAggregator聚合, 然后发向后面。
这里有一个问题:
`为什么不将65536个字符一下发送到最终handler, 而需要先分解成虚拟的chunked, 一个一个发送到后面再聚合起来?`
# HttpObjectDecoder和HttpRequestDecoder
首先需要知道, Rquest请求由FullHttpRequest构成, 主要分为两部分:
+ HttpRequest: 主要存放inital, head等。
+ HttpContent: 传输的数据部分
HttpRequestDecoder类继承自HttpObjectDecoder, 主要实现了decode函数, 主要负责把数据流解析成`HttpRequest`,实际就是将`ChannelBuffer`转变为多个`HttpChunk`对象。
HttpObjectDecoder继承自ByteToMessageDecoder类, 这个类是不是很熟悉, 详见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/03/25/Netty%E9%80%9A%E4%BF%A1%E7%BC%96%E8%A7%A3%E7%A0%81%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">Netty通信编解码源码解读</a>。
HttpObjectDecoder主要根据当前保存的状态位currentState(不要被定义的状态位吓到, 根据名称就能大致了解其作用)来决定即将完成的动作, 默认初始值为:State.SKIP_CONTROL_CHARS。
decode函数实现如下:
```
protected void decode(ChannelHandlerContext ctx, ByteBuf buffer, List<Object> out) throws Exception {
        if (resetRequested) {
            resetNow();
        }

        switch (currentState) { //没有break
        case SKIP_CONTROL_CHARS: { //skip_control_chars
            if (!skipControlCharacters(buffer)) {
                return;
            }
            currentState = State.READ_INITIAL;   //read_initail
        }
        case READ_INITIAL: try {  //read_initial   请求换行符(line)  比如解析出来GET /_cat/indices HTTP/1.1
            AppendableCharSequence line = lineParser.parse(buffer); //lineParser: LineParser继承自HeaderParser，调用的还是HeaderParser.parse
            if (line == null) {
                return;
            }
            String[] initialLine = splitInitialLine(line); //{Method, URL, HTTPVersion
            if (initialLine.length < 3) { //无效的请求， 忽略。
                // Invalid initial line - ignore.
                currentState = State.SKIP_CONTROL_CHARS;  //skip_control_chars
                return;
            }
            message = createMessage(initialLine);  //DefaultHttpRequest
            currentState = State.READ_HEADER;  //read_header
            // fall-through
        } catch (Exception e) {
            out.add(invalidMessage(buffer, e));
            return;
        }
        case READ_HEADER: try {    //read_header
            State nextState = readHeaders(buffer); //读取完header部分，同时根据header部分修改了nextState的值，告诉了读取content的方式
            if (nextState == null) {
                return;
            }
            currentState = nextState;
            switch (nextState) {
            case SKIP_CONTROL_CHARS:   //skip_control_char
                out.add(message);
                out.add(LastHttpContent.EMPTY_LAST_CONTENT);  //empty_last_content
                resetNow();
                return;
            case READ_CHUNK_SIZE: //read_chunk_size
                if (!chunkedSupported) {
                    throw new IllegalArgumentException("Chunked messages not supported");
                }
                // Chunked encoding - generate HttpMessage first.  HttpChunks will follow.
                out.add(message);
                return;
            default:  //或者读取变量类型长度或者定长
                long contentLength = contentLength();//没有长度相关变量就是-1
                if (contentLength == 0 || contentLength == -1 && isDecodingRequest()) {
                    out.add(message);
                    out.add(LastHttpContent.EMPTY_LAST_CONTENT);  //enpty_last_content
                    resetNow();
                    return;
                }
                assert nextState == State.READ_FIXED_LENGTH_CONTENT ||  //read_fixed_length_content
                        nextState == State.READ_VARIABLE_LENGTH_CONTENT; //read_variable_length_content
                out.add(message); //目前message=DefaultHttpRequest, 放进去了line和header部分
                if (nextState == State.READ_FIXED_LENGTH_CONTENT) {  //read_fixed_lengt_content
                    // chunkSize will be decreased as the READ_FIXED_LENGTH_CONTENT state reads data chunk by chunk.
                    chunkSize = contentLength; // 注意这两个直接赋值一样
                }
                // We return here, this forces decode to be called again where we will decode the content
                return;
            }
        } catch (Exception e) {
            out.add(invalidMessage(buffer, e));
            return;
        }
        case READ_VARIABLE_LENGTH_CONTENT: {  //read_variable_length_content
            // Keep reading data as a chunk until the end of connection is reached.
            int toRead = Math.min(buffer.readableBytes(), maxChunkSize);
            if (toRead > 0) {
                ByteBuf content = buffer.readRetainedSlice(toRead);
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        case READ_FIXED_LENGTH_CONTENT: {  //read_fixed_length_content
            int readLimit = buffer.readableBytes();
            // Check if the buffer is readable first as we use the readable byte count
            // to create the HttpChunk. This is needed as otherwise we may end up with
            // create a HttpChunk instance that contains an empty buffer and so is
            // handled like it is the last HttpChunk.
            //
            // See https://github.com/netty/netty/issues/433
            if (readLimit == 0) {
                return;
            }
            int toRead = Math.min(readLimit, maxChunkSize);
            if (toRead > chunkSize) {
                toRead = (int) chunkSize;
            }
            ByteBuf content = buffer.readRetainedSlice(toRead);  //buffer = PooledUnsafeDirectByteBuf, 实际会跑到AbstractByteBuf.readRetainedSlice()里面
            chunkSize -= toRead; //content = PooledSlicedByteBuf
            if (chunkSize == 0) {  //要是定长的话，就直接content就是DefaultLastHttpContent，
                // Read all content.
                out.add(new DefaultLastHttpContent(content, validateHeaders));
                resetNow(); //解析完了就该返回了
            } else {
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        /**
         * everything else after this point takes care of reading chunked content. basically, read chunk size,
         * read chunk, read and ignore the CRLF and repeat until 0
         */
        case READ_CHUNK_SIZE: try {//read_chunk_size
            AppendableCharSequence line = lineParser.parse(buffer);
            if (line == null) {
                return;
            }
            int chunkSize = getChunkSize(line.toString());
            this.chunkSize = chunkSize;
            if (chunkSize == 0) {
                currentState = State.READ_CHUNK_FOOTER;//read_chunk_footer
                return;
            }
            currentState = State.READ_CHUNKED_CONTENT;//read_chunked_content
            // fall-through
        } catch (Exception e) {
            out.add(invalidChunk(buffer, e));
            return;
        }
        case READ_CHUNKED_CONTENT: { //read_chunked_content
            assert chunkSize <= Integer.MAX_VALUE;
            int toRead = Math.min((int) chunkSize, maxChunkSize);
            toRead = Math.min(toRead, buffer.readableBytes());
            if (toRead == 0) {
                return;
            }
            HttpContent chunk = new DefaultHttpContent(buffer.readRetainedSlice(toRead));
            chunkSize -= toRead;
            out.add(chunk);
            if (chunkSize != 0) {
                return;
            }
            currentState = State.READ_CHUNK_DELIMITER;//read_chunked_delimiter
            // fall-through
        }
        case READ_CHUNK_DELIMITER: {//read_chunked_delimiter
            final int wIdx = buffer.writerIndex();
            int rIdx = buffer.readerIndex();
            while (wIdx > rIdx) {
                byte next = buffer.getByte(rIdx++);
                if (next == HttpConstants.LF) {
                    currentState = State.READ_CHUNK_SIZE;//read_chunked_size
                    break;
                }
            }
            buffer.readerIndex(rIdx);
            return;
        }
        case READ_CHUNK_FOOTER: try {//read_chunked_foooter
            LastHttpContent trailer = readTrailingHeaders(buffer);
            if (trailer == null) {
                return;
            }
            out.add(trailer);
            resetNow();
            return;
        } catch (Exception e) {
            out.add(invalidChunk(buffer, e));
            return;
        }
        case BAD_MESSAGE: {  //bad_message
            // Keep discarding until disconnection.
            buffer.skipBytes(buffer.readableBytes());
            break;
        }
        case UPGRADED: {//upgraded
            int readableBytes = buffer.readableBytes();
            if (readableBytes > 0) {
                // Keep on consuming as otherwise we may trigger an DecoderException,
                // other handler will replace this codec with the upgraded protocol codec to
                // take the traffic over at some point then.
                // See https://github.com/netty/netty/issues/2173
                out.add(buffer.readBytes(readableBytes));
            }
            break;
        }
        }
    }
```
注意这里的case并没有break, decode主要做了如下逻辑:
1)  首先检查byte, 要跳过最开始的控制符或者空格, 部分控制符就是ascii编码为31之前的字符。
```
private static boolean skipControlCharacters(ByteBuf buffer) {
        boolean skiped = false;
        final int wIdx = buffer.writerIndex();
        int rIdx = buffer.readerIndex();
        while (wIdx > rIdx) {
            int c = buffer.getUnsignedByte(rIdx++);
            if (!Character.isISOControl(c) && !Character.isWhitespace(c)) {//0～31及127(共33个)是控制字符或通信专用字符（其余为可显示字符），如控制符：LF（换行）、CR（回车）、FF（换页）、DEL（删除）、BS（退格)、BEL（响铃）等
                rIdx--;
                skiped = true;
                break;
            }
        }
        buffer.readerIndex(rIdx);
        return skiped;
    }
```
首先读取当前字母, 若发现符合要求, 再复位当前读指针。 并将动作设置为READ_INITIAL, 表示接下来将要读取initial部分。
2) 读取INITIAL部分
从当前节点开始读取字符,直到读取分割符号为HttpConstants.LF(换行符), 该部分将解析出如下信息:`GET /_cat/indices HTTP/1.1`, 创建对象:DefaultHttpRequest, 其中
```
httpVersion: HTTP/1.1
method: GET
uri: /_cat/indices
```
这个DefaultHttpRequest在HttpObjectDecoder中生成, 作为最终的这个请求的头部分。然后将状态位置为READ_HEADER, 表示即将读取header部分。
3) 读取Headers部分
```
private State readHeaders(ByteBuf buffer) {
        final HttpMessage message = this.message;  //DefaultHttpRequest
        final HttpHeaders headers = message.headers();  //headers = DefaultHttpHeaders
        AppendableCharSequence line = headerParser.parse(buffer);//不停地解析header， 下面是个do()while{}为循环
        if (line == null) {
            return null;
        }
        if (line.length() > 0) {
            do { //这是个while循环，以换行符来进行分割
                char firstChar = line.charAt(0);
                if (name != null && (firstChar == ' ' || firstChar == '\t')) {
                    String trimmedLine = line.toString().trim();
                    String valueStr = String.valueOf(value);
                    value = valueStr + ' ' + trimmedLine;
                } else {
                    if (name != null) {
                        headers.add(name, value);
                    }
                    splitHeader(line);
                }
                line = headerParser.parse(buffer);   //
                if (line == null) {
                    return null;
                }
            } while (line.length() > 0);
        }

        // Add the last header.
        if (name != null) {   //解析出最后一个header
            headers.add(name, value);
        }
        // reset name and value fields
        name = null;
        value = null;

        State nextState;

        if (isContentAlwaysEmpty(message)) {  //header是否为空
            HttpUtil.setTransferEncodingChunked(message, false);
            nextState = State.SKIP_CONTROL_CHARS;  // 哪里有问题，又是重头开始
        } else if (HttpUtil.isTransferEncodingChunked(message)) { // 是否包含 transfer-encoding: chunked
            nextState = State.READ_CHUNK_SIZE;
        } else if (contentLength() >= 0) {   //Content-Length: 80
            nextState = State.READ_FIXED_LENGTH_CONTENT;  //下一个读取Content值
        } else { //没有Content-Length和chunked相关的，就是读取变量类型长度
            nextState = State.READ_VARIABLE_LENGTH_CONTENT;
        }
        return nextState;
        }
```
主要做的事:
+ 读取header部分和读取inital部分一样, 也是根据HttpConstants.LF(换行符)循环读取每一行, 并且解读出key-value出来, 获取到所有的header内容, 同时也放入DefaultHttpRequest中, header内容示例如下:
```
"Accept" -> "*/*"
"User-Agent" -> "curl/7/43/0"
"Host" -> "127.0.0.1:9200"
"Content-Length" -> "66735"
"Content-Encoding" -> "gzip"
"Content-Type" -> "application/x-www-form-urlencoded"
"Expect" -> "100-continue"
"null" -> "null"
```
我们需要了解一个参数:Expect: 100-continue

> <a href="https://blog.csdn.net/MitKey/article/details/52042537">参考</a>100-continue 是用于客户端在发送 post 数据给服务器时，征询服务器情况，看服务器是否处理 post 的数据，如果不处理，客户端则不上传 post 是数据，反之则上传。在实际应用中，通过 post 上传大数据时，才会使用到 100-continue 协议。<p>客户端策略:
如果客户端有 post 数据要上传，可以考虑使用 100-continue 协议。在请求头中加入 {“Expect”:”100-continue”}
如果没有 post 数据，不能使用 100-continue 协议，因为这会让服务端造成误解。
并不是所有的 Server 都会正确实现 100-continue 协议，如果 Client 发送 Expect:100-continue 消息后，在 timeout 时间内无响应，Client 需要立马上传 post 数据。
有些 Server 会错误实现 100-continue 协议，在不需要此协议时返回 100，此时客户端应该忽略。<p>服务端策略:
正确情况下，收到请求后，返回 100 或错误码。
如果在发送 100-continue 前收到了 post 数据（客户端提前发送 post 数据），则不发送 100 响应码(略去)。

这个参数也不是必须有的, 当content部分长度超过, 客户端才会向服务器端发送这个参数。 在terminal下面通过curl发送包含数据请求, 当数据部分长度>=1025时, 客户端发送的header里面才会有这个参数。
+ 如上因为header中包含Content-Length, 说明接下来需要读取定长为66735的一个帧。
这里会设置状态为READ_FIXED_LENGTH_CONTENT
4)  读取内容
因为header读取完成之后, 将nextState设置成了READ_FIXED_LENGTH_CONTENT, 那么会连续接收并读取chunkSize长度的byte。这里有个设置, 我们设置了maxChunkSize, 意味着每次读取的chunked的长度必须<Math.min(readableLength, maxChunkSize), 每读取maxChunkSize长度的值就向后传递, 同时修改chunkSize的值。读取第二个chunked的动作在MessageToMessageDecoder中发出(该content的readableBytes>0)。
这里对于maxChunkSize的限制不甚理解, 既然已经读取到readableLength长度的值, 为啥还需要再次分割每个chunked为maxChunkSize。
# HttpObjectAggregator和 MessageAggregator
HttpObjectAggregator主要是将HttpRequest和HttpContent合并成FullHttpRequest, 继承自MessageAggregator。
MessageAggregator实现了decode()函数, 继承了MessageToMessageDecoder(很熟悉), 主要实现如下:
```
    protected void decode(final ChannelHandlerContext ctx, I msg, List<Object> out) throws Exception {
        if (isStartMessage(msg)) {//会跑到HttpObjectAggregator里面，只要是HttpMessage类型就行
            handlingOversizedMessage = false;
            if (currentMessage != null) {
                currentMessage.release();
                currentMessage = null;
                throw new MessageAggregationException();
            }
            @SuppressWarnings("unchecked")
            S m = (S) msg; //DefaultHttpRequest
            // Send the continue response if necessary (e.g. 'Expect: 100-continue' header)
            // Check before content length. Failing an expectation may result in a different response being sent.
            Object continueResponse = newContinueResponse(m, maxContentLength, ctx.pipeline());//跑到HttpObjectAggregator里面，第一次返回DefaultFullHttpResponse
            if (continueResponse != null) { //向客户端返回100-continue, 告诉客户端可以发送content了
                // Cache the write listener for reuse.
                ChannelFutureListener listener = continueResponseWriteListener;
                if (listener == null) {
                    continueResponseWriteListener = listener = new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (!future.isSuccess()) {
                                ctx.fireExceptionCaught(future.cause());
                            }
                        }
                    };
                }
                // Make sure to call this before writing, otherwise reference counts may be invalid.
                boolean closeAfterWrite = closeAfterContinueResponse(continueResponse);
                handlingOversizedMessage = ignoreContentAfterContinueResponse(continueResponse);

                final ChannelFuture future = ctx.writeAndFlush(continueResponse).addListener(listener);

                if (closeAfterWrite) {
                    future.addListener(ChannelFutureListener.CLOSE);
                    return;
                }
                if (handlingOversizedMessage) {
                    return;
                }
            } else if (isContentLengthInvalid(m, maxContentLength)) { //检查length是否有效，
                // if content length is set, preemptively close if it's too large
                invokeHandleOversizedMessage(ctx, m);
                return;
            }

            if (m instanceof DecoderResultProvider && !((DecoderResultProvider) m).decoderResult().isSuccess()) {
                O aggregated;
                if (m instanceof ByteBufHolder && ((ByteBufHolder) m).content().isReadable()) {
                    aggregated = beginAggregation(m, ((ByteBufHolder) m).content().retain());
                } else {
                    aggregated = beginAggregation(m, EMPTY_BUFFER);
                }
                finishAggregation(aggregated);
                out.add(aggregated);
                return;
            }
             //同时生成好Compent
            // A streamed message - initialize the cumulative buffer, and wait for incoming chunks.
            CompositeByteBuf content = ctx.alloc().compositeBuffer(maxCumulationBufferComponents);//只有start类型数值才能生成CompositeByteBuf，后面内容部分只管向里面添加即可
            if (m instanceof ByteBufHolder) {
                appendPartialContent(content, ((ByteBufHolder) m).content());
            }
            currentMessage = beginAggregation(m, content); //currentMessage = AggregatedFullHttpRequest
        } else if (isContentMessage(msg)) { //解析内容部分
            if (currentMessage == null) {
                // it is possible that a TooLongFrameException was already thrown but we can still discard data
                // until the begging of the next request/response.
                return;
            }

            // Merge the received chunk into the content of the current message.
            CompositeByteBuf content = (CompositeByteBuf) currentMessage.content();

            @SuppressWarnings("unchecked")
            final C m = (C) msg; //可能是DefaultLastHttpContent
            // Handle oversized message.
            if (content.readableBytes() > maxContentLength - m.content().readableBytes()) {
                // By convention, full message type extends first message type.
                @SuppressWarnings("unchecked")
                S s = (S) currentMessage;
                invokeHandleOversizedMessage(ctx, s);
                return;
            }
            // Append the content of the chunk.
            appendPartialContent(content, m.content()); //把产生的数据添加到末尾
            //
            // Give the subtypes a chance to merge additional information such as trailing headers.
            aggregate(currentMessage, m);  //HttpObjectAggregator.aggregate()    currentMessage=AggregatedFullHttpRequest

            final boolean last;
            if (m instanceof DecoderResultProvider) {
                DecoderResult decoderResult = ((DecoderResultProvider) m).decoderResult();
                if (!decoderResult.isSuccess()) {
                    if (currentMessage instanceof DecoderResultProvider) {
                        ((DecoderResultProvider) currentMessage).setDecoderResult(
                                DecoderResult.failure(decoderResult.cause()));
                    }
                    last = true;
                } else {
                    last = isLastContentMessage(m);
                }
            } else {
                last = isLastContentMessage(m);
            }

            if (last) {  //如果Content是最后一个，那么就开始组合了，向out添加结果后就可以继续发送，否则就直接退出了，
                finishAggregation(currentMessage);

                // All done
                out.add(currentMessage); //把结果放进来意味着继续向下一个处理器发送，否则就直接接收下一个chunked。
                currentMessage = null;
            }
        } else {
            throw new MessageAggregationException();
        }
    }
```
decode函数主要检查该解析请求是否是HttpRequest或者HttpContent, 否则直接返回异常。
1) 若请求是HttpRequest
说明该部分是request最开始的那一部分。
+ 首先检查是否请求中是否包含Expect: 100-continue(在newContinueResponse中检查): 若包含有, 服务器需要向客户端发送可以发送content的response, response中content为空; 反之, 说明不用向客户端发送continue的回复。
+ 生成CompositeByteBuf, 准备存放即将到来的HttpChunk; 生成AggregatedFullHttpRequest, 将CompositeByteBuf和DefaultHttpRequest包含其中。
需要简单介绍下CompositeByteBuf, 通过名字也可以看出, 他是一个复合型的ByteBuf, 它并不是真实的, 它主要由属性`List<Component> components`构成, 每新来一个ByteBuf, 都会添加到components中。 CompositeByteBuf也有自己的writerIndex和readIndex, 表示整个CompositeByteBuf最大可读和最大可写偏移量。

2) 若请求是HttpContent部分
+ 将content添加进CompositeByteBuf中
通过appendPartialContent()添加, conponent添加进CompositeByteBuf的过程如下:
```
     private int addComponent0(boolean increaseWriterIndex, int cIndex, ByteBuf buffer) {
            .....
            if (cIndex == components.size()) {
                wasAdded = components.add(c);
                if (cIndex == 0) {
                    c.endOffset = readableBytes;
                } else {
                    Component prev = components.get(cIndex - 1);
                    c.offset = prev.endOffset;
                    c.endOffset = c.offset + readableBytes;
                }
            }
            .....
            if (increaseWriterIndex) {
                writerIndex(writerIndex() + buffer.readableBytes());
            }
            return cIndex;
        }
    }
```
每个Component结构如下:
```
        ByteBuf buf;  //该Component实际存储
        final int length;
        int offset; 标记该Component占CompositeByte所有Component byte的起始偏移位置。
        int endOffset;  //标记该Component占CompositeByte所有Component byte的最终偏移位置。
```
在添加的时候, curr.offset = pre.endOffset,curr.endOffset = pre.offset+ readLength, 这样每个Component offset和endOffset指针首位相连。

+ 等待所有的content发送过来
1. 轮训等待所有的部分content发送过来, 封装成Component放进CompositeByte中。
2. 直到检测到content为最后一个content(类型为LastHttpContent), 则将CompositeByte放入out中继续向里面传递。

至此,一个完整地AggregatedFullHttpRequest已经解析出来了,组成如下:
<img src="http://owqu66xvx.bkt.clouddn.com/DefaultLastHttpContent.png" />
# 附
如何将Composite转换为一个连续的堆内buf呢?
通过Unpooled.copiedBuffer(request.content())方法即可。
