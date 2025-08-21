# 需求描述： 在小程序和 Web 网站应用中接入阿里云百炼智能体，实现流式聊天对话
#### 参考文档：<a href="https://bailian.console.aliyun.com/#/home">https://bailian.console.aliyun.com/#/home</a>
#### 环境：JDK8 + Spring Boot2  小程序和web网站应用
#### 流式输出规范
     数据格式：遵循 SSE 协议，以 data: 开头
     分割规则：每个 chunk 用 \n\n 分隔
     示例：
 ```text
 data: {"id":"1","content":"你好"}

 data: {"id":"2","content":"请问有什么可以帮你？"}

```
#### 异常情况：
##### 小程序收到 chunk 内容不完整
      原因：微信小程序对流式响应不兼容，可能分片断裂
      解决方案：前端增加 累加缓冲区，拼接字符串后再 JSON.parse
      类似
```js
data:
{"id":"sdfs"}
```

##### 流式输出阻塞（前端一次性收到所有数据）
  原因：网关或服务端 buffer 缓存未及时 flush
  解决方案：
    网关需配置 关闭/减小缓存字节
    通过response设置缓存
##### WebClient 调用阿里云百炼报错
  可能原因：网络抖动、连接被关闭、超时
  解决方案：
    配置 WebClient 超时与重试机制


#### 代码：
#### 方案一：
``` java
@PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamChat(@RequestBody ChatRequest request, HttpServletResponse response) {

    return abstractAIProvider.streamChat(request,response);
}

final WebClient webClient;

public AbstractAIProvider() {
    this.webClient = WebClient.builder().build();
}

@Override
public SseEmitter streamChat(ChatRequest request, HttpServletResponse response) {
    // 手动设置响应头
    response.setContentType(MediaType.TEXT_EVENT_STREAM_VALUE);
    response.setHeader("X-Accel-Buffering", "no");
    response.setHeader("Cache-Control", "no-cache");
    response.setHeader("Connection", "keep-alive");

    SseEmitter emitter = new SseEmitter(Duration.ofMinutes(15).toMillis());
    AtomicBoolean isCompleted = new AtomicBoolean(false);
    AtomicBoolean started = new AtomicBoolean(false);
    // 追踪当前正在发送的事件数量
    AtomicInteger pendingSends = new AtomicInteger(0);

    ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r);
        t.setDaemon(true);
        t.setName(request.getConversionId() + "sse-heartbeat-thread");
        return t;
    });
    // 启动心跳，每3秒发送一次
    ScheduledFuture<?> heartbeatFuture = scheduler.scheduleAtFixedRate(() -> {
        if (started.get() || isCompleted.get()){
            return;
        }
        ChatResponse heartbeat = ChatResponse.builder()
                .conversionId(request.getConversionId())
                .content("")
                .isEnd(false)
                .build();
        // 增加待发送计数
        safeSend(emitter, heartbeat, isCompleted,"null");
    }, 0, 3, TimeUnit.SECONDS);


    BusinessThreadPool.PRESCRIPTION_POOL.execute(() -> {
        try {

            // ========== 2. 调用 webClient 主流 ==========
            Object requestBody = prepareRequest(request);
            log.info("SSE-进入模型请求开始对话详情id:{},对话id:{}",request.getConversionItemId(),request.getConversionId());
            //结束心跳检测
            stopHeartBeat(started, heartbeatFuture, scheduler);
            StringBuilder contentBuilder = new StringBuilder();
            HttpHeaders headers = new HttpHeaders();

            headers.setAccept(ListUtil.toList(MediaType.TEXT_EVENT_STREAM));
            Flux<String> eventStream = webClient.post()
                    .uri(getApiUrl())
                    .header("Authorization", "Bearer " + getApiKey())
                    .header("Content-Type", "application/json")
                    .header("X-DashScope-SSE", "enable")
                    .headers(httpHeaders -> httpHeaders.addAll(headers))
                    .body(BodyInserters.fromObject(requestBody))
                    .accept(MediaType.TEXT_EVENT_STREAM)
                    .retrieve()
                    .bodyToFlux(String.class)
                    .timeout(Duration.ofSeconds(30))  // 响应超时：15秒，超过无数据就超时
                    .retryWhen(throwableFlux ->
                        throwableFlux
                                .zipWith(Flux.range(1, 4)) // 最多尝试3次（1,2,3），第4次不再retry
                                .flatMap(tuple -> {
                                    int attempt = tuple.getT2();
                                    log.info("SSE-进入模型请求重试开始对话详情id:{},对话id:{}，重试次数：{}",request.getConversionItemId(),request.getConversionId(),attempt);
                                    return Mono.delay(Duration.ofSeconds(2));
                                })
                    );

            eventStream.subscribe(
                    sse -> {
                        // 增加待发送计数
                        pendingSends.incrementAndGet();
                        try {
                            log.info("SSE-进入模型请求响应分块对话详情id:{},对话id:{},chunk内容:{}",request.getConversionItemId(),request.getConversionId(),sse);

                            safeSend(emitter, resp, isCompleted, request.getConversionItemId());
                            log.info("SSE-进入模型请求响应分块对话详情id:{},对话id:{},chunk内容正常发送成功:{}",request.getConversionItemId(),request.getConversionId(),sse);
                        }finally {
                            // 发送完成后，减少计数
                            pendingSends.decrementAndGet();
                        }

                    },
                    ex -> {
                        log.info("SSE-进入模型请求响应异常对话详情id:{},对话id:{}，超时126s异常进入捕获阶段",request.getConversionItemId(),request.getConversionId(),ex);
                        pendingSends.incrementAndGet();
                        try {

                            safeSend(emitter, errorResp, isCompleted, request.getConversionItemId());
                            log.info("SSE-进入模型请求响应异常对话详情id:{},对话id:{},异常发送完成",request.getConversionItemId(),request.getConversionId());
                        } catch (Exception e) {
                            log.info("SSE-进入模型请求响应异常对话详情id:{},对话id:{},进入异常消除，未发送数据给前端",request.getConversionItemId(),request.getConversionId());
                        }finally {
                            try {
                                // 发送完成后，减少计数
                                pendingSends.decrementAndGet();
                                // 等待所有事件发送完毕
                                waitForPendingSends(pendingSends, request.getConversionItemId());
                                if (isCompleted.compareAndSet(false, true)) {
                                    emitter.completeWithError(ex);
                                }
                            }catch (Exception e){
                                log.info("SSE-进入模型请求响应异常对话详情id:{},对话id:{},进入异常消除，finally未正常结束",request.getConversionItemId(),request.getConversionId());
                            }
                        }
                    },
                    () -> {
                        try {
                            log.info("SSE-进入模型请求响应结束对话详情id:{},对话id:{},",request.getConversionItemId(),request.getConversionId());
                            // 等待所有事件发送完毕
                            waitForPendingSends(pendingSends,request.getConversionItemId());
                            if (isCompleted.compareAndSet(false, true)) {
                                emitter.complete();
                            }
                        } catch (Exception e) {
                            log.info("SSE-进入模型请求响应结束对话详情id:{},对话id:{},进入了异常消除",request.getConversionItemId(),request.getConversionId(),e);
                        }
                    }
            );

        } catch (Exception e) {
            log.info("SSE-进入模型核心处理异常开始对话详情id:{},对话id:{}",request.getConversionItemId(),request.getConversionId(),e);
            //结束心跳检测
            stopHeartBeat(started, heartbeatFuture, scheduler);
            handleError(request.getConversionItemId(), "稍后再试", e);
            // 等待所有事件发送完毕
            waitForPendingSends(pendingSends,request.getConversionItemId());
            if (isCompleted.compareAndSet(false, true)) {
                emitter.completeWithError(e);
            }
        }
    });


    // 连接关闭处理
    emitter.onTimeout(() -> {
        log.info("SSE-进入模型连接超时对话详情id:{},对话id:{},请求参数：{}",request.getConversionItemId(),request.getConversionId(),JSONUtil.toJsonStr(request));

    });

    emitter.onCompletion(() -> {
        log.info("SSE-进入模型请求响应结束,正常执行onCompletion对话详情id:{},对话id:{},",request.getConversionItemId(),request.getConversionId());

    });

    emitter.onError(error -> {
        log.info("SSE-进入模型连接超时对话详情id:{},对话id:{},请求参数：{},通信异常",request.getConversionItemId(),request.getConversionId(),JSONUtil.toJsonStr(request),error);
    });
    return emitter;
}
```
##### 方案二
```java
return Flux.using(
            StringBuilder::new,
            contentBuilder -> {
                try {
                    return webClient.post()
                        .uri(getApiUrl())
                        .header("Authorization", "Bearer " + getApiKey())
                        .header("Content-Type", "application/json")
                        .header("X-DashScope-SSE", "enable")
                        .body(BodyInserters.fromObject(requestBody))
                        .accept(MediaType.APPLICATION_STREAM_JSON)
                        .retrieve()
                        .bodyToFlux(String.class)
                        // 1. 首次响应超时：首次chunk最长30秒
                        .timeout(Duration.ofSeconds(30),
                            Flux.error(new TimeoutException("首次响应超时")))
                        // 2. 后续每条chunk最大间隔60秒
                        .timeout(Duration.ofSeconds(60))
                        .limitRate(1)   # 限制流Rate，背压
                        .map(chunk -> {
                            try {
                                String content = parseChunk(chunk, contentBuilder);
                                return ChatResponse.builder()
                                        .conversionId(request.getConversionId())
                                        .content(content)
                                        .isEnd(false)
                                        .build();
                            } catch (Exception e) {
                                log.warn("解析 chunk 出错", e);
                                return ChatResponse.builder()
                                        .conversionId(request.getConversionId())
                                        .error("解析出错：" + e.getMessage())
                                        .isEnd(false)
                                        .build();
                            }
                        })
                        .concatWith(Flux.defer(() -> Flux.just(
                                ChatResponse.builder()
                                        .conversionId(request.getConversionId())
                                        .content(contentBuilder.toString())
                                        .isEnd(true)
                                        .build()
                        )))
                        .onErrorResume(error ->
                                Flux.defer(() -> {
                                    log.error("通义千问API调用错误: {}", error.getMessage(), error);
                                    handleError(request.getConversionItemId(), contentBuilder.toString(), error);
                                    return Flux.just(ChatResponse.builder()
                                            .conversionId(request.getConversionId())
                                            .error("处理请求时出错: " + error.getMessage())
                                            .isEnd(true)
                                            .build());
                                })
                        )
                        .doOnComplete(() ->
                            postProcessResponse(request.getConversionItemId(), contentBuilder.toString()));
                } catch (Exception e) {
                    log.error("请求构造或执行出错", e);
                    handleError(request.getConversionItemId(), contentBuilder.toString(), e);
                    return Flux.just(ChatResponse.builder()
                                     .conversionId(request.getConversionId())
                                     .error("处理请求时出错: " + e.getMessage())
                                     .isEnd(true)
                                     .build());
                }
            },
            contentBuilder -> {
                // 资源清理
            }
        );
```