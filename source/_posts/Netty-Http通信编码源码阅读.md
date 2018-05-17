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
1. 首先通过encode()进行编码, 若out没有编码输出, 则直接抛出异常。encode()是在HttpContentEncoder中实现的。
2. 针对编码输出out,

## HttpContentEncoder的encode()函数
首先需要了解decode(), 在写入的时候, 将header里面的accept-encoding属性取值赋给acceptEncodingQueue, 这样编码的时候就知道需要使用什么编码器了, 本wiki在使用时, 客户端发送的编码器: "gzip,deflat,br"
endoce函数如下:
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
+ 返回EmbeddedChannel, 我们需要注意该channel里面通过ZlibCodecFactory.newZlibEncoder()方式添加了一个handler, 该返回EmbeddedChannel的pipeline结构如下:
<img src="http://owsl7963b.bkt.clouddn.com/GzipPipline.png"/>


