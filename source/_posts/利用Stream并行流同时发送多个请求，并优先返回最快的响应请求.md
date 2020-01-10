---
title: 利用Stream并行流同时发送多个请求，并优先返回最快的响应请求
date: 2019/12/30 13:47:00
tags: 
- Java
---

# 利用Stream并行流同时发送多个请求，并优先返回最快的响应请求

## 环境
- JDK8
- 请求库: OkHttp3

## 说明
记录编写爬虫时的一个需求，请求多个相同功能的API，直接返回一个最快的即可

## 编码
```java
// 略过初始化client..
private OkHttpClient client;
/**
 * @param requests Okhttp3.Request 批量请求组成的List
 */
public String response(List<Request> requests) {
    // 保存与requests相同大小的Call集合
    List<Call> calls = new ArrayList<>(requests.size());
    AtomicReference<String> responseResult = new AtomicReference<>();
    // 关键部分，使用并行流
    requests.parallelStream() 
            .forEach(request -> {
                // 把所有Call对象加入List中，便于后续的取消请求操作
                Call call = client.newCall(request);
                calls.add(call);
                try (Response response = call.execute()) {
                    // 到这一步时，说明已有响应返回，但也有可能会有多个响应同时返回，通过加锁保证并发安全
                    synchronized (this) {
                        if (calls.isEmpty()) {
                            return;
                        } else {
                            // 删除当前返回的Call对象
                            calls.remove(call);
                            // 在对List中剩下的Call进行取消请求
                            calls.forEach(Call::cancel);
                            calls.clear();
                        }
                    }
                    ResponseBody body = response.body().string();
                    // 设置响应结果
                    responseResult.set(body);
                } catch (IOException e) {
                    // 请求被取消、超时、等因素则会抛出此异常
                    log.warn("Request Exception: {}", e.getMessage());
                }
            });
    // parse 
    return responseResult.get();
}
```

## 实战测试
这里用高德地图和腾讯地图API中的**IP定位**功能，进行测试

```java
@Slf4j
public class MapRequest {
    private final OkHttpClient client;
    
    public MapRequest() {
        this.client = new OkHttpClient.Builder()
                // 打印日志拦截器
                .addInterceptor(new LoggingInterceptor())
                // 3秒超时
                .readTimeout(3, TimeUnit.MINUTES)
                .build();
    }
    
    /**
     * 高德地图Ip定位API
     * @return okhttp3.Request
     */
    private Request getAmapMapRequest() {
        HttpUrl httpUrl = new HttpUrl.Builder()
                .scheme("https")
                .host("restapi.amap.com")
                .addPathSegments("v3/ip")
                .addQueryParameter("ip", "ip")
                .addQueryParameter("key", "key").build();
        return new Request.Builder().url(httpUrl).build();
    }
    
    /**
     * 腾讯地图Ip定位API
     * @return okhttp3.Request
     */
    private Request getTencentRequest() {
        HttpUrl httpUrl = new HttpUrl.Builder()
                .scheme("https")
                .host("apis.map.qq.com")
                .addPathSegments("ws/location/v1/ip")
                .addQueryParameter("ip", "ip")
                .addQueryParameter("key", "key")
                .build();
        return new Request.Builder()
                .url(httpUrl)
                .addHeader("Referer", "https://lbs.qq.com/webservice_v1/guide-ip.html")
                .build();
    }
    
    public String response(List<Request> requests) {
        // 与上面编码部分相同
    }
    
    public String get() {
        Request amapRequest = this.getAmapRequest();
        Request tencentRequest = this.getTencentRequest();
        String response = this.response(Arrays.asList(amapRequest, tencentRequest));
        log.info("Response: {}", response);
        return response;
    }
}
```
### 运行截图
![](https://blog-1256602811.cos.ap-nanjing.myqcloud.com/request.png)



## 优化
由于使用并行流，线程池使用的是整个程序共用的**FokeJoinPool.commonPool**，由于并行请求为I/O密集型应用，当数量过多时会影响整个程序的性能，所以这里自定义一个ForkJoinPool线程池来使用。

```java
@Slf4j
public class MapRequest {
    // 自定义线程池
    private final ForkJoinPool forkJoinPool = new ForkJoinPool(4);
    // ... 其他代码一样
    public String response(List<Request> requests) {
        ForkJoinTask<String> forkJoinTask = forkJoinPool.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                AtomicReference<String> responseResult = new AtomicReference<>();
                ArrayList<Call> calls = new ArrayList<>(requests.size());
                requests.parallelStream()
                        .forEach(request -> {
                            // 把所有Call对象加入List中，便于后续的取消请求操作
                            Call call = client.newCall(request);
                            calls.add(call);
                            Instant now = Instant.now();
                            try (Response response = call.execute()) {
                                log.info("time: {}", Duration.between(now, Instant.now()).toMillis());
                                // 到这一步时，说明已有响应返回，但也有可能会有多个响应同时返回，通过加锁保证原子性操作
                                synchronized (this) {
                                    if (calls.isEmpty()) {
                                        return;
                                    } else {
                                        // 删除最先返回的Call对象
                                        calls.remove(call);
                                        // 在对List中剩下的Call进行取消请求
                                        calls.forEach(Call::cancel);
                                        calls.clear();
                                    }
                                }
                                String body = response.body().string();
                                // 设置响应结果
                                responseResult.set(body);
                            } catch (IOException e) {
                                // 请求被取消、超时、等因素则会抛出此异常
                                log.warn("Request Exception: {}", e.getMessage());
                            }
                        });
                return responseResult.get();
            }
        });
        try {
            return forkJoinTask.get();
        } catch (InterruptedException | ExecutionException e) {
            log.error("Exception: {}", e.getMessage());
            return null;
        }
    }
}
```

