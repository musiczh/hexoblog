---
title: okHttp	#标题
date: 2021/1/30 00:00:00 						#建立日期
sticky:  #置顶参数
tags:	#标签
 - 
					
categories:	#分类
 - 

updated: 					#更新日期
author: 一只修仙的猿 #作者
toc: true	#是否显示toc
mathjax:  #数学公式

keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字

katex:
aplayer:
highlight_shrink: true       #代码框是否打开
---

## 前言

## OkHttpClient

#### 重要属性

```kotlin
// 分发器，负责请求的线程调度以及性能平衡等；内部维护有一个线程池来执行请求。
internal var dispatcher: Dispatcher = Dispatcher()
// 连接池
internal var connectionPool: ConnectionPool = ConnectionPool()
// 普通拦截器
internal val interceptors: MutableList<Interceptor> = mutableListOf()
// network拦截器
internal val networkInterceptors: MutableList<Interceptor> = mutableListOf()
// 创建eventListener，用于记录okHttp的行为
internal var eventListenerFactory: EventListener.Factory = EventListener.NONE.asFactory()
// 请求失败是否要进行重试
internal var retryOnConnectionFailure = true
// 权限不足会返回4xx；可以在请求的时候访问接口获取token再添加请求头
internal var authenticator: Authenticator = Authenticator.NONE
// 是否进行重定向
internal var followRedirects = true
// 这个也是判断是否进行重定向，但是这个是更加具体到https和http的互换
internal var followSslRedirects = true
// cookie存储器；没有默认实现，我们可以自己继承接口来实现cookie存储
internal var cookieJar: CookieJar = CookieJar.NO_COOKIES
// 缓存类。根据请求来存储响应数据。Cache类本身已经实现，开发者直接new Cache(file,capacity)就可以了
internal var cache: Cache? = null
// 域名解析
internal var dns: Dns = Dns.SYSTEM
// 代理服务器。本地请求代理，代理代替用户向目标发起请求。
internal var proxy: Proxy? = null
internal var proxySelector: ProxySelector? = null
// 和上面的authenticator类似，不过这个是相对于代理服务器来说
internal var proxyAuthenticator: Authenticator = Authenticator.NONE
// 创建socket工厂
internal var socketFactory: SocketFactory = SocketFactory.getDefault()
internal var sslSocketFactoryOrNull: SSLSocketFactory? = null
internal var x509TrustManagerOrNull: X509TrustManager? = null
// 支持的https以及安全连接版本列表。TSL、SSL；加密算法、哈希算法等
internal var connectionSpecs: List<ConnectionSpec> = DEFAULT_CONNECTION_SPECS
// 协议列表。提供给okHttp，客户端能支持的http协议列表
internal var protocols: List<Protocol> = DEFAULT_PROTOCOLS
// 验证对方主机。https使用
internal var hostnameVerifier: HostnameVerifier = OkHostnameVerifier
// 指定域名的公钥。请求一个服务器的时候，会直接验证本地设置的固定公钥
// 用于自签名证书，避免本地根证书验证
internal var certificatePinner: CertificatePinner = CertificatePinner.DEFAULT
// 整理证书，形成一个证书链表，第一个就是对方网站证书，最后一个是本地信任的证书。
internal var certificateChainCleaner: CertificateChainCleaner? = null
internal var callTimeout = 0
// 连接时间
internal var connectTimeout = 10_000
// 下载超时
internal var readTimeout = 10_000
// 写入超时
internal var writeTimeout = 10_000
// ping的间隔
internal var pingInterval = 0
internal var minWebSocketMessageToCompress = RealWebSocket.DEFAULT_MINIMUM_DEFLATE_SIZE
internal var routeDatabase: RouteDatabase? = null
```

#### newCall

```kotlin
// 创建一个RealCall对象
// 这里的request是最初的request，后续需要经过一系列修改形成最终的request
// forWebSocket拓展http，让服务端可以反过来给客户端做推送，例如股票app可以不断地给客户端推动，代替轮询
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

## RealCall

#### enqueue&execute

```kotlin
RealCall.kt
override fun enqueue(responseCallback: Callback) {
   	// 使用CAS尝试设置executed变量为true，失败则抛出异常
  	check(executed.compareAndSet(false, true)) { "Already Executed" }
  	callStart()
    // 回调分发器来执行 
  	client.dispatcher.enqueue(AsyncCall(responseCallback))
}

override fun execute(): Response {
    // 使用CAS尝试设置executed变量为true，失败则抛出异常
    check(executed.compareAndSet(false, true)) { "Already Executed" }
	
    timeout.enter()
    callStart()
    try {
        // 分发记录请求
        client.dispatcher.executed(this)
        // 通过拦截责任链来获取结果
        return getResponseWithInterceptorChain()
    } finally {
        // 结束调用分发器的finshed方法
        client.dispatcher.finished(this)
    }
}
```



#### RealCall.AsyncCall

```kotlin
fun executeOn(executorService: ExecutorService) {
    client.dispatcher.assertThreadDoesntHoldLock()
    // 是否成功执行
    var success = false
    try {
        // 使用线程池执行该任务
        executorService.execute(this)
        success = true
    } catch (e: RejectedExecutionException) {
        // 异常处理
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        noMoreExchanges(ioException)
        // 回调onFailure方法
        responseCallback.onFailure(this@RealCall, ioException)
    } finally {
        if (!success) {
            // 线程池执行中出现异常，不再执行
            client.dispatcher.finished(this) // This call is no longer running!
        }
    }
}


override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        // 开始计算时间
        timeout.enter()
        try {
            // 通过拦截责任链来获取请求结果
            val response = getResponseWithInterceptorChain()
            signalledCallback = true
            // 回调callBack方法
            responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
            ...
        } catch (t: Throwable) {
            ...
        } finally {
            // 调用分发器的结束方法
            client.dispatcher.finished(this)
        }
    }
}
```



#### getResponseWithInterceptorChain

```kotlin
@Throws(IOException::class)
internal fun getResponseWithInterceptorChain(): Response {
    // 按顺序添加所有的拦截器
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
        interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    // 创建一个拦截责任链，并赋值相关属性
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
        // 执行请求获取返回值
        val response = chain.proceed(originalRequest)
        if (isCanceled()) {
            response.closeQuietly()
            throw IOException("Canceled")
        }
        return response
    } catch (e: IOException) {
        ...
    } finally {
        ...
    }
}
```



## RealInterceptorChain

#### proceed

```kotlin
override fun proceed(request: Request): Response {
    // 检查数组是否越界
    check(index < interceptors.size)
    calls++
    ...
    // 复制一个chain，数组下标+1
    val next = copy(index = index + 1, request = request)
    val interceptor = interceptors[index]

    @Suppress("USELESS_ELVIS")
    // 拦截器执行责任链,递归思路，依次调用所有的拦截器
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")
	...
	// 检查response.body是否为null
    check(response.body != null) { "interceptor $interceptor returned a response with no body" }

    return response
}
```





## Dispatcher

#### 关键变量：

- readyAsyncCalls ：准备执行的异步请求
- runningAsyncCalls：正在执行的异步请求
- runningSyncCalls：正在执行的同步请求
- maxRequests：最大请求数；可以自己设置，如果想让请求按顺序，可以设置为1
- maxRequestsPerHost：单个host最大请求数（不懂）；可以自己设置。

#### 线程池

```kotlin
private var executorServiceOrNull: ExecutorService? = null

@get:Synchronized
@get:JvmName("executorService") val executorService: ExecutorService
get() {
    if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
                                                   SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
    }
    return executorServiceOrNull!!
}
```

#### equeue&execute

```kotlin
Dispatcher.kt
internal fun enqueue(call: AsyncCall) {
      synchronized(this) {
          	// 加入到准备异步请求队列
            readyAsyncCalls.add(call)
			// 寻找是否存在相同host的AsyncCall，重用callsPerHost变量对象
          	// 这样所有相同host的AsyncCall都拥有相同的callsPerHost对象
          	// 这个callsPerHost对象是一个AtomicInteger对象，线程安全自增 
            if (!call.call.forWebSocket) {
              	val existingCall = findExistingCallWithHost(call.host)
            	if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
            }
      }
    	// 执行准备队列中的请求
      promoteAndExecute()
}

@Synchronized internal fun executed(call: RealCall) {
    // 把同步请求记录到队列中
    runningSyncCalls.add(call)
}

```



#### promoteAndExecute

```kotlin
private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
        // 获取准备执行的异步请求列表
        val i = readyAsyncCalls.iterator()
        while (i.hasNext()) {
            val asyncCall = i.next()
			// 判断请求总数是否超出限制
            if (runningAsyncCalls.size >= this.maxRequests) break 
            // 判断单个host的请求数是否超出限制
            if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue 
			// 符合条件的请求移出队列
            i.remove()
            // 增加单个host的请求数
            asyncCall.callsPerHost.incrementAndGet()
            // 放入即将执行的队列
            executableCalls.add(asyncCall)
            // 放入运行中的任务队列
            runningAsyncCalls.add(asyncCall)
        }
        // 是否有请求正在执行，包括了同步和异步请求；也就是两个running队列的长度之和	
        isRunning = runningCallsCount() > 0
    }

    for (i in 0 until executableCalls.size) {
        // 获取前面添加的每个任务，使用线程池进行执行
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }

    // 返回是否正在执行请求
    return isRunning
}
```





#### finish

```kotlin
internal fun finished(call: AsyncCall) {
    // 减少单个host的请求数
    call.callsPerHost.decrementAndGet()
    finished(runningAsyncCalls, call)
}

internal fun finished(call: RealCall) {
    finished(runningSyncCalls, call)
}

// 最终都回调到这个方法
private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
        if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
        idleCallback = this.idleCallback
    }
	// 继续执行剩下的请求
    val isRunning = promoteAndExecute()
	// 如果没有请求正在执行，则调用idleCallBack
    if (!isRunning && idleCallback != null) {
        idleCallback.run()
    }
}
```



## Interceptor

###  RetryAndFollowUpInterceptor

重试与重定向拦截器。主要负责的任务有两个：

- 重试：当请求出现异常如IOException、RouteException时，自动进行重试
- 重定向：请求结果返回时，需要根据响应码判断是否需要进行重定向。当需要进行重定向时，自动进行重定向。重定向的次数最多只有20次。

```kotlin
class RetryAndFollowUpInterceptor(private val client: OkHttpClient) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val realChain = chain as RealInterceptorChain
        var request = chain.request
        val call = realChain.call
        var followUpCount = 0
        var priorResponse: Response? = null
        var newExchangeFinder = true
        var recoveredFailures = listOf<IOException>()
        while (true) {
            call.enterNetworkInterceptorExchange(request, newExchangeFinder)

            var response: Response
            var closeActiveExchange = true
            try {
                // 如果请求已经取消，则抛出异常
                if (call.isCanceled()) {
                    throw IOException("Canceled")
                }

                try {
                    // 通过下一个拦截器来获得请求结果
                    response = realChain.proceed(request)
                    newExchangeFinder = true
                } catch (e: RouteException) {
                    // 路由异常，请求可能还没发出去
                    if (!recover(e.lastConnectException, call, request,requestSendStarted = false)) {
                        throw e.firstConnectException.withSuppressed(recoveredFailures)
                    } else {
                        recoveredFailures += e.firstConnectException
                    }
                    newExchangeFinder = false
                    continue
                } catch (e: IOException) {
                    // io异常，请求可能已经发出去了
                    // ConnectionShutdownException这个异常http2才会抛出
                    if (!recover(e, call, request, 
                                 requestSendStarted = e !is ConnectionShutdownException)) {
                        throw e.withSuppressed(recoveredFailures)
                    } else {
                        recoveredFailures += e
                    }
                    newExchangeFinder = false
                    continue
                }

                // Attach the prior response if it exists. Such responses never have a body.
                if (priorResponse != null) {
                    response = response.newBuilder()
                    .priorResponse(priorResponse.newBuilder()
                                   .body(null)
                                   .build())
                    .build()
                }

                val exchange = call.interceptorScopedExchange
                // 判断是否需要进行重定向
                val followUp = followUpRequest(response, exchange)

                if (followUp == null) {
                    if (exchange != null && exchange.isDuplex) {
                        call.timeoutEarlyExit()
                    }
                    closeActiveExchange = false
                    return response
                }

                val followUpBody = followUp.body
                if (followUpBody != null && followUpBody.isOneShot()) {
                    closeActiveExchange = false
                    return response
                }

                response.body?.closeQuietly()
				// 限制重定向次数为20次
                if (++followUpCount > MAX_FOLLOW_UPS) {
                    throw ProtocolException("Too many follow-up requests: $followUpCount")
                }
                // 修改request，进行重定向
                request = followUp
                priorResponse = response
            } finally {
                call.exitNetworkInterceptorExchange(closeActiveExchange)
            }
        }
    }
    
```



#### recover

```kotlin	
// 判断是否要进行重试
    private fun recover(
        e: IOException,
        call: RealCall,
        userRequest: Request,
        requestSendStarted: Boolean
    ): Boolean {
        // 配置client时候设置了不可重试
        if (!client.retryOnConnectionFailure) return false

        // We can't send the request body again.
        if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

        // 协议异常，不需要重试
        if (!isRecoverable(e, requestSendStarted)) return false

        // 没有更多的路由可以选择
        if (!call.retryAfterFailure()) return false

        // For failure recovery, use the same route selector with a new connection.
        return true
    }
}
```





### BridgeInterceptor

桥接拦截器的作用是补充头部信息使request变为规范的http请求可以被发送。其次就是cookie和存取以及gzip压缩。

- 补充请求头，如cookie、contentType，gzip等
- 存储cookie，解压缩。

```kotlin
// 默认是没有实现的CookieJar，可以在构建client的时候自定义
class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val userRequest = chain.request()
        val requestBuilder = userRequest.newBuilder()

        val body = userRequest.body
        if (body != null) {
            val contentType = body.contentType()
            // 设置contentType请求类型
            if (contentType != null) {
                requestBuilder.header("Content-Type", contentType.toString())
            }
            // 设置Content-Length和Transfer-Encoding，请求体解析方式或内容长度
            val contentLength = body.contentLength()
            if (contentLength != -1L) {
                requestBuilder.header("Content-Length", contentLength.toString())
                requestBuilder.removeHeader("Transfer-Encoding")
            } else {
                requestBuilder.header("Transfer-Encoding", "chunked")
                requestBuilder.removeHeader("Content-Length")
            }
        }
        // 设置Host
        if (userRequest.header("Host") == null) {
            requestBuilder.header("Host", userRequest.url.toHostHeader())
        }
		// 设置Connection为Keep-Alive，表示长连接
        if (userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive")
        }
        // 添加gzip，如果添加了这个请求头，那么也就有责任对返回的结果进行解压缩
        var transparentGzip = false
        if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
            transparentGzip = true
            requestBuilder.header("Accept-Encoding", "gzip")
        }
        // 添加cookie
        val cookies = cookieJar.loadForRequest(userRequest.url)
        if (cookies.isNotEmpty()) {
            requestBuilder.header("Cookie", cookieHeader(cookies))
        }
        // 添加User-Agent，请求的用户信息
        if (userRequest.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", userAgent)
        }
        
        // ----------------------------------------------------------------------------
		// 执行请求，获取响应信息
        val networkResponse = chain.proceed(requestBuilder.build())
        // ----------------------------------------------------------------------------

        // 保存cookie
        cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

        val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)
		// 如果前面设置了gzip请求头，那么需要对响应进行解压缩
        if (transparentGzip &&
            "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
            networkResponse.promisesBody()) {
            val responseBody = networkResponse.body
            if (responseBody != null) {
                // 使用okio的GzipSource来进行解压
                val gzipSource = GzipSource(responseBody.source())
                val strippedHeaders = networkResponse.headers.newBuilder()
                .removeAll("Content-Encoding")
                .removeAll("Content-Length")
                .build()
                responseBuilder.headers(strippedHeaders)
                val contentType = networkResponse.header("Content-Type")
                responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
            }
        }

        // 最后返回结果
        return responseBuilder.build()
    }

}
```





### CacheInterceptor

```kotlin
// cache默认是null
class CacheInterceptor(internal val cache: Cache?) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val call = chain.call()
        // 通过cache的get方法判断是否命中缓存
        val cacheCandidate = cache?.get(chain.request())

        val now = System.currentTimeMillis()
		// 获取缓存策略
        val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
        // 获取策略中的两个属性
        val networkRequest = strategy.networkRequest
        val cacheResponse = strategy.cacheResponse

        cache?.trackResponse(strategy)
        val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

        if (cacheCandidate != null && cacheResponse == null) {
            // 缓存的body不再使用，直接关闭
            cacheCandidate.body?.closeQuietly()
        }

        // 如果两者都是null，返回504
        if (networkRequest == null && cacheResponse == null) {
            return Response.Builder()
            .request(chain.request())
            .protocol(Protocol.HTTP_1_1)
            .code(HTTP_GATEWAY_TIMEOUT)
            .message("Unsatisfiable Request (only-if-cached)")
            .body(EMPTY_RESPONSE)
            .sentRequestAtMillis(-1L)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build().also {
                listener.satisfactionFailure(call, it)
            }
        }

        // 如果networkRequest == null，这里则表示cacheResponse！=null
        // 直接把缓存返回
        if (networkRequest == null) {
            return cacheResponse!!.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .build().also {
                listener.cacheHit(call, it)
            }
        }

        // 到这里已经把networkRequest==null的情况处理完成了
        // 说明需要进行网络请求
        if (cacheResponse != null) {
            listener.cacheConditionalHit(call, cacheResponse)
        } else if (cache != null) {
            listener.cacheMiss(call)
        }

        var networkResponse: Response? = null
        try {
            // 网路请求获取响应
            networkResponse = chain.proceed(networkRequest)
        } finally {
            // 如果请求失败，关闭缓存主体的body
            if (networkResponse == null && cacheCandidate != null) {
                cacheCandidate.body?.closeQuietly()
            }
        }

        // 根据响应码进行操作
        if (cacheResponse != null) {
            if (networkResponse?.code == HTTP_NOT_MODIFIED) {
                val response = cacheResponse.newBuilder()
                .headers(combine(cacheResponse.headers, networkResponse.headers))
                .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
                .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build()

                networkResponse.body!!.close()

                // Update the cache after combining headers but before stripping the
                // Content-Encoding header (as performed by initContentStream()).
                cache!!.trackConditionalCacheHit()
                cache.update(cacheResponse, response)
                return response.also {
                    listener.cacheHit(call, it)
                }
            } else {
                cacheResponse.body?.closeQuietly()
            }
        }

        val response = networkResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

        if (cache != null) {
            if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
                // 调用cache的put方法进行缓存
                val cacheRequest = cache.put(response)
                return cacheWritingResponse(cacheRequest, response).also {
                    if (cacheResponse != null) {
                        // This will log a conditional cache miss only.
                        listener.cacheMiss(call)
                    }
                }
            }

            if (HttpMethod.invalidatesCache(networkRequest.method)) {
                try {
                    cache.remove(networkRequest)
                } catch (_: IOException) {
                    // The cache cannot be written.
                }
            }
        }

        return response
    }
```



#### Cache.get()

```kotlin
internal fun get(request: Request): Response? {
    val key = key(request.url)
    val snapshot: DiskLruCache.Snapshot = try {
        cache[key] ?: return null
    } catch (_: IOException) {
        return null // Give up because the cache cannot be read.
    }

    val entry: Entry = try {
        Entry(snapshot.getSource(ENTRY_METADATA))
    } catch (_: IOException) {
        snapshot.closeQuietly()
        return null
    }

    val response = entry.response(snapshot)
    if (!entry.matches(request, response)) {
        response.body?.closeQuietly()
        return null
    }

    return response
}
```



#### CacheStrategy

##### Factory.init()

```kotlin
// 主要是对cacheResponse的内容进行保存
init {
  if (cacheResponse != null) {
    this.sentRequestMillis = cacheResponse.sentRequestAtMillis
    this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis
    val headers = cacheResponse.headers
    for (i in 0 until headers.size) {
      val fieldName = headers.name(i)
      val value = headers.value(i)
      when {
        fieldName.equals("Date", ignoreCase = true) -> {
          servedDate = value.toHttpDateOrNull()
          servedDateString = value
        }
        fieldName.equals("Expires", ignoreCase = true) -> {
          expires = value.toHttpDateOrNull()
        }
        fieldName.equals("Last-Modified", ignoreCase = true) -> {
          lastModified = value.toHttpDateOrNull()
          lastModifiedString = value
        }
        fieldName.equals("ETag", ignoreCase = true) -> {
          etag = value
        }
        fieldName.equals("Age", ignoreCase = true) -> {
          ageSeconds = value.toNonNegativeInt(-1)
        }
      }
    }
  }
}
```

##### Factory.compute()

```kotlin
// 该方法返回缓存策略者，拦截器可根据这个拦截策略来进行缓存判断
fun compute(): CacheStrategy {
    // 完成真正的缓存判断
    val candidate = computeCandidate()
    // 如果可以使用缓存，那么netWorkResquest==null
    // 如果networkRequest！=null，但是又只能使用缓存，则无法继续下一步
    if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
    }
    return candidate
}
```

##### Factory.computeCandidate()

```kotlin
private fun computeCandidate(): CacheStrategy {
    // 没有缓存响应。如果存在缓存响应，但不一定符合要求，如超出时间等
    if (cacheResponse == null) {
        return CacheStrategy(request, null)
    }

    // httpa缓存。如果失去了握手信息，那么这个缓存不能使用
    if (request.isHttps && cacheResponse.handshake == null) {
        return CacheStrategy(request, null)
    }

    // 判断是否可以进行缓存
    // 这里需要根据响应码和头部信息是否禁止缓存来判断缓存是否无效
    if (!isCacheable(cacheResponse, request)) {
        return CacheStrategy(request, null)
    }
	
    // 根据请求头来判断是否直接向服务发起请求
    val requestCaching = request.cacheControl
    if (requestCaching.noCache || hasConditions(request)) {
        return CacheStrategy(request, null)
    }

    val responseCaching = cacheResponse.cacheControl

    // 获取缓存响应从缓存到现在的时间
    val ageMillis = cacheResponseAge()
    // 获取这个缓存响应的有效时长
    var freshMillis = computeFreshnessLifetime()
    
    if (requestCaching.maxAgeSeconds != -1) {
        // 如果请求头中指定了 max-age最小有效时长，那么两者取最小
        freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
    }
    
	// Cache-Control:min-fresh 请求中指定了该参数。请求认为的缓存剩下的最小有效时长
    var minFreshMillis: Long = 0
    if (requestCaching.minFreshSeconds != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
    }
    
    // Cache-Control:must-revalidate 过期则必须再向源服务器进行确认
	// Cache-Control:max-stale 缓存过期后还能使用指定的时长,未指定表示缓存过期时间无限
	// 前者会忽略后者，所以判断了不必须向服务器确认，再获得请求头中的max-stale
    var maxStaleMillis: Long = 0
    if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
    }
	
    // 不需要和服务器验证 & 缓存的存在时长-请求要求剩下的有效时长<缓存的有效时长+缓存过期可使用的有效时长
    if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        val builder = cacheResponse.newBuilder()
        // 如果缓存过期，则需要增加请求头
        if (ageMillis + minFreshMillis >= freshMillis) {
            builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
        }
        val oneDayMillis = 24 * 60 * 60 * 1000L
        // 缓存存在时间超过一天且响应中没有设置过期时间，需要添加警告
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
            builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
        }
        return CacheStrategy(null, builder.build())
    }

    // 判断是否是需要验证
    // 如果不是则发起普通的请求
    val conditionName: String
    val conditionValue: String?
    when {
        etag != null -> {
            conditionName = "If-None-Match"
            conditionValue = etag
        }

        lastModified != null -> {
            conditionName = "If-Modified-Since"
            conditionValue = lastModifiedString
        }

        servedDate != null -> {
            conditionName = "If-Modified-Since"
            conditionValue = servedDateString
        }

        else -> return CacheStrategy(request, null) // No condition! Make a regular request.
    }

    val conditionalRequestHeaders = request.headers.newBuilder()
    conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

    val conditionalRequest = request.newBuilder()
    .headers(conditionalRequestHeaders.build())
    .build()
    return CacheStrategy(conditionalRequest, cacheResponse)
}
```



##### Factory.hasConditions

```kotlin
// 判断请求头是否设置了下面两个标志
private fun hasConditions(request: Request): Boolean =
request.header("If-Modified-Since") != null || request.header("If-None-Match") != null
```



##### CacheStrategy.isCacheable

```kotlin
// 缓存策略类的伴生对象方法，根据缓存的response响应code和request的头部信息判断缓存是否无效
fun isCacheable(response: Response, request: Request): Boolean {
    when (response.code) {
        HTTP_OK,
        HTTP_NOT_AUTHORITATIVE,
        HTTP_NO_CONTENT,
        HTTP_MULT_CHOICE,
        HTTP_MOVED_PERM,
        HTTP_NOT_FOUND,
        HTTP_BAD_METHOD,
        HTTP_GONE,
        HTTP_REQ_TOO_LONG,
        HTTP_NOT_IMPLEMENTED,
        StatusLine.HTTP_PERM_REDIRECT -> {
            // 这些响应头可以进行缓存，除非响应头或请求头设置了禁止使用缓存
        }

        HTTP_MOVED_TEMP,
        StatusLine.HTTP_TEMP_REDIRECT -> {
            // 这些响应码不一定可以进行缓存，需要进一步判断
            if (response.header("Expires") == null &&
                response.cacheControl.maxAgeSeconds == -1 &&
                !response.cacheControl.isPublic &&
                !response.cacheControl.isPrivate) {
                return false
            }
        }
        else -> {
            // 其他的响应码无法进行缓存
            return false
        }
    }

    // 判断响应头是否设置了禁止缓存或者请求头设置了禁止使用缓存
    return !response.cacheControl.noStore && !request.cacheControl.noStore
}
```



### ConnectInterceptor

```kotlin
// 这个拦截器主要负责与服务器创建连接
object ConnectInterceptor : Interceptor {
    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val realChain = chain as RealInterceptorChain
        // 创建Exchange对象，用于后续发送请求头和请求体
        // 这一步会进行DNS，创建socket连接
        val exchange = realChain.call.initExchange(chain)
        val connectedChain = realChain.copy(exchange = exchange)
        // 后续如果有netWorkIntercepter会在下一步执行
        // 也就是说networkInterceptor是在创建好socket之后再被执行
        return connectedChain.proceed(realChain.request)
    }
}
```



#### RealCall.initExchange()

```kotlin
// 创建Exchange对象；这个对象内部维持了连接，可以直接发送数据
// 下一步拦截器就会使用这个对象来发送请求头和请求体
internal fun initExchange(chain: RealInterceptorChain): Exchange {
    synchronized(this) {
        check(expectMoreExchanges) { "released" }
        check(!responseBodyOpen)
        check(!requestBodyOpen)
    }
    val exchangeFinder = this.exchangeFinder!!
    // 通过ExchangeFinder来获取一个ExchangeCodec
    // ExchangeCodec用于编码和解码http请求，有两个具体的实现类：
    // http1ExchangeCodec和http2ExchangeCodec,对应http1协议和http2协议
    // 这个对象才是真正做IO的对象，Exchange本身只是做了封装
    val codec = exchangeFinder.find(client, chain)
    // 通过ExchangeCodec来构建Exchange对象并返回
    val result = Exchange(this, eventListener, exchangeFinder, codec)
    this.interceptorScopedExchange = result
    this.exchange = result
    synchronized(this) {
        this.requestBodyOpen = true
        this.responseBodyOpen = true
    }
    if (canceled) throw IOException("Canceled")
    return result
}
```

#### ExchangeFinder.find()

```kotlin
// 创建一个ExchangeCodec对象
fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
): ExchangeCodec {
    try {
        // 创建RealConnection对象，这个对象是连接的实体
        // 内部会进行DNS访问、创建socket连接、保存protocol、handShake等信息
        val resultConnection = findHealthyConnection(
            connectTimeout = chain.connectTimeoutMillis,
            readTimeout = chain.readTimeoutMillis,
            writeTimeout = chain.writeTimeoutMillis,
            pingIntervalMillis = client.pingIntervalMillis,
            connectionRetryEnabled = client.retryOnConnectionFailure,
            doExtensiveHealthChecks = chain.request.method != "GET"
        )
        // 最后通过这个realConnection来创建一个ExchangeCodec对象
        return resultConnection.newCodec(client, chain)
    } catch (e: RouteException) {
        trackFailure(e.lastConnectException)
        throw e
    } catch (e: IOException) {
        trackFailure(e)
        throw RouteException(e)
    }
}
```



#### RealConnection.newCodec()

```kotlin
// 创建具体的ExchangeCodec对象
internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
    val socket = this.socket!!
    val source = this.source!!
    val sink = this.sink!!
    val http2Connection = this.http2Connection
	// 根据协议来创建具体连接对象
    return if (http2Connection != null) {
        Http2ExchangeCodec(client, this, chain, http2Connection)
    } else {
        socket.soTimeout = chain.readTimeoutMillis()
        source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
        sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
        Http1ExchangeCodec(client, this, source, sink)
    }
}
```



#### ExchangeFinder.findHealthyConnection()

```kotlin
// 寻找一个可用的连接
private fun findHealthyConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    doExtensiveHealthChecks: Boolean
): RealConnection {
    while (true) {
        // 获取连接的真正方法
        val candidate = findConnection(
            connectTimeout = connectTimeout,
            readTimeout = readTimeout,
            writeTimeout = writeTimeout,
            pingIntervalMillis = pingIntervalMillis,
            connectionRetryEnabled = connectionRetryEnabled
        )

        // Confirm that the connection is good.
        if (candidate.isHealthy(doExtensiveHealthChecks)) {
            return candidate
        }

        // If it isn't, take it out of the pool.
        candidate.noNewExchanges()

        // Make sure we have some routes left to try. One example where we may exhaust all the routes
        // would happen if we made a new connection and it immediately is detected as unhealthy.
        if (nextRouteToTry != null) continue

        val routesLeft = routeSelection?.hasNext() ?: true
        if (routesLeft) continue

        val routesSelectionLeft = routeSelector?.hasNext() ?: true
        if (routesSelectionLeft) continue

        throw IOException("exhausted all routes")
    }
}
```



#### ExchangeFinder.findConnection()

```kotlin
private fun findConnection(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean
): RealConnection {
    if (call.isCanceled()) throw IOException("Canceled")

    // 尝试去复用Call中的连接
    val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
    if (callConnection != null) {
        ...
    }

    ...

    // 在连接池中尝试获取一个连接
    if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
        val result = call.connection!!
        eventListener.connectionAcquired(call, result)
        return result
    }

    // 在连接池中获取失败，先找路由
    val routes: List<Route>?
    val route: Route
    if (nextRouteToTry != null) {
        // 判断有没有下一个路由。这里在重试拦截器那里被配置，所以只有重试的情况会走进来
        routes = null
        route = nextRouteToTry!!
        nextRouteToTry = null
    } else if (routeSelection != null && routeSelection!!.hasNext()) {
        // Use a route from an existing route selection.
        routes = null
        route = routeSelection!!.next()
    } else {
        // 前面没有获取到路由，那么这里就需要创建一个新的路由连接
        var localRouteSelector = routeSelector
        if (localRouteSelector == null) {
            localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
            this.routeSelector = localRouteSelector
        }
        // 根据DNS获取IP地址，寻找下一个路由；这里是一个阻塞过程，需要等待DNS请求返回
        val localRouteSelection = localRouteSelector.next()
        routeSelection = localRouteSelection
        routes = localRouteSelection.routes

        if (call.isCanceled()) throw IOException("Canceled")

        //把获取到的新的路由添加到连接池中，再次尝试获取新的路由连接
        if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
            val result = call.connection!!
            eventListener.connectionAcquired(call, result)
            return result
        }
		
        route = localRouteSelection.next()
    }

    // 前面如果没有获取到连接，这一步会创建新的连接
    // connect方法会进行TSL和TCP，创建连接
    val newConnection = RealConnection(connectionPool, route)
    call.connectionToCancel = newConnection
    try {
        newConnection.connect(
            connectTimeout,
            readTimeout,
            writeTimeout,
            pingIntervalMillis,
            connectionRetryEnabled,
            call,
            eventListener
        )
    } finally {
        call.connectionToCancel = null
    }
    call.client.routeDatabase.connected(newConnection.route())

    // 如果此时遇到同个主机的请求，那么合并连接，复用连接
    if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
        val result = call.connection!!
        nextRouteToTry = route
        newConnection.socket().closeQuietly()
        eventListener.connectionAcquired(call, result)
        return result
    }
	
    // 把新的连接保存到连接池中
    synchronized(newConnection) {
        connectionPool.put(newConnection)
        call.acquireConnectionNoEvents(newConnection)
    }

    eventListener.connectionAcquired(call, newConnection)
    return newConnection
}
```



### CallServerInterceptor

```kotlin
// 这一步主要就是发送请求头和请求体，然后构造完整的response对象并返回给上一层
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val realChain = chain as RealInterceptorChain
        val exchange = realChain.exchange!!
        val request = realChain.request
        val requestBody = request.body
        val sentRequestMillis = System.currentTimeMillis()

        var invokeStartEvent = true
        var responseBuilder: Response.Builder? = null
        var sendRequestException: IOException? = null
        try {
            // 发送请求
            exchange.writeRequestHeaders(request)
			// 如果有请求体，发送请求体
            if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
                // 根据请求头判断是否需要访问服务判断是否可以发送请求体
                if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
                    // 发送请求头
                    exchange.flushRequest()
                    // 读取response
                    responseBuilder = exchange.readResponseHeaders(expectContinue = true)
                    exchange.responseHeadersStart()
                    invokeStartEvent = false
                }
                if (responseBuilder == null) {
                    if (requestBody.isDuplex()) {
                        // Prepare a duplex body so that the application can send a request body later.
                        exchange.flushRequest()
                        val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
                        requestBody.writeTo(bufferedRequestBody)
                    } else {
                        // Write the request body if the "Expect: 100-continue" expectation was met.
                        val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
                        requestBody.writeTo(bufferedRequestBody)
                        bufferedRequestBody.close()
                    }
                } else {
                    exchange.noRequestBody()
                    if (!exchange.connection.isMultiplexed) {
                        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
                        // from being reused. Otherwise we're still obligated to transmit the request body to
                        // leave the connection in a consistent state.
                        exchange.noNewExchangesOnConnection()
                    }
                }
            } else {
                exchange.noRequestBody()
            }

            if (requestBody == null || !requestBody.isDuplex()) {
                exchange.finishRequest()
            }
        } catch (e: IOException) {
            if (e is ConnectionShutdownException) {
                throw e // No request was sent so there's no response to read.
            }
            if (!exchange.hasFailure) {
                throw e // Don't attempt to read the response; we failed to send the request.
            }
            sendRequestException = e
        }

        try {
            if (responseBuilder == null) {
                responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
                if (invokeStartEvent) {
                    exchange.responseHeadersStart()
                    invokeStartEvent = false
                }
            }
            var response = responseBuilder
            .request(request)
            .handshake(exchange.connection.handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build()
            var code = response.code
            if (code == 100) {
                // Server sent a 100-continue even though we did not request one. Try again to read the
                // actual response status.
                responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
                if (invokeStartEvent) {
                    exchange.responseHeadersStart()
                }
                response = responseBuilder
                .request(request)
                .handshake(exchange.connection.handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build()
                code = response.code
            }

            exchange.responseHeadersEnd(response)

            response = if (forWebSocket && code == 101) {
                // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
                response.newBuilder()
                .body(EMPTY_RESPONSE)
                .build()
            } else {
                response.newBuilder()
                .body(exchange.openResponseBody(response))
                .build()
            }
            if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
                "close".equals(response.header("Connection"), ignoreCase = true)) {
                exchange.noNewExchangesOnConnection()
            }
            if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
                throw ProtocolException(
                    "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
            }
            return response
        } catch (e: IOException) {
            if (sendRequestException != null) {
                sendRequestException.addSuppressed(e)
                throw sendRequestException
            }
            throw e
        }
    }
}
```













1. 简述okHttp的使用流程

   > 1. 首先，配置OkHttpClient
   > 2. 创建request请求
   > 3. 使用client对request创建call对象
   > 4. 使用call对象来发起异步或同步请求

2. 讲讲okHttp的原理

   > okHttp可以发起两种请求：同步、异步。
   >
   > 异步请求会被添加分发器的队列中，等待线程池来获取任务执行；任务的执行数量受到分发器的参数限制；任务执行时会调用拦截器链来获取响应
   >
   > 同步请求会直接调用拦截器链来获取响应
   >
   > 拦截器链按顺序分为：
   >
   > 1. 普通拦截器：开发者自己添加
   > 2. 重试或重定向拦截器：在遇到问题可以自动进行重试、或者需要重定向时自动进行重定向
   > 3. 桥接拦截器：补充请求头，进行cookie缓存
   > 4. 缓存拦截器：判断是否命中缓存，根据响应和请求头的缓存策略来决定是否发起请求
   > 5. 连接拦截器：使用dns寻找ip并建立tcp连接和tls连接
   > 6. 网络拦截器：开发者自己添加
   > 7. 请求服务拦截器：发起真正的http请求并获取响应
   >
   > 不同的拦截器依次分工，负责不同的工作，请求响应则依次通过这些拦截器，获取最终的响应。

3. okHttp中的线程池

   > 其配置核心线程为0，线程总数无限，队列长度为0，所以每一个任务都会被马上执行；

4. 如何进行okHttp缓存

   > okHttp默认不进行缓存，需要开发者自己创建Cache对象并设置给OkHttpClient。创建Cache对象时，可以指定文件目录和大小；
   >
   > okHttp会根据响应和请求的缓存策略来进行缓存操作；
   >
   > 我们可以通过给request配置cacheControl参数来设置我们的请求缓存策略；
   >
   > cache对象内部使用的是DiskLruCache来实现数据持久化，他是一个最近最少使用原则的本地存储，也就是当缓存满了之后，会删除最近最少使用的缓存。

5. 缓存策略如何？

   > 

6. okHttp每一次请求都会发起新的连接吗？

   > 不会。在连接拦截中，每一次的连接都会存放在连接池中，相同ip主机的请求会共用连接。
   >
   > 连接池的最大连接数量是5个且空闲存活时间是5分钟，超过会被清理。



## 最后



> 全文到此，原创不易，觉得有帮助可以点赞收藏评论转发。
> 笔者才疏学浅，有任何想法欢迎评论区交流指正。
> 如需转载请评论区或私信交流。
>
> 另外欢迎光临笔者的个人博客：[传送门](https://qwerhuan.gitee.io)

