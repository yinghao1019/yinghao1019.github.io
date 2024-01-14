---
title: RestTemplate 效能調教
date: 2024-01-09 22:29:41
tags:
  - [Http]
  - [Java]
  - [Apache]
  - [RestTemplate]
categories:
  - [Spring Boot]
description: 本文講解如何在Spring Boot 專案中優化RestTemplate 的效能, 通過將Http Client 替換成Apache Http Client 、
  設定connection pool, keep-alive 策略等, 讓讀者更清楚RestTemplate 優化的方向。
---

## 前言
當後端需要跟其他第三方服務、系統溝通時, 最常用的方式為透過Rest API 進行溝通。Spring Boot 專案中, 提供了一個好用的
call API 工具 - RestTemplate , RestTemplate 提供了簡單的介面簡化了我們呼叫API的程式碼。

但是, RestTemplate 在發送大量請求時往往會發生效能瓶頸, 故本篇文章將教學如何進行效能調教來解決RestTemplate 速度慢的問題。



## 設定ClientHttpRequestFactory

RestTemplate 底層預設發送 HTTP Request  的工具為使用 HttpURLConnection 來進行發送。此工具未支援HTTP Connection 
Pool 來縮短消耗的時間 , 我們可將其換成現今Java 有支援Connection Pool 的工具 , 像是 OkHttp、Apache HttpClient、
WebClient、FeignClient 等等。

而本篇將採用設定Apache HttpClient作為RestTemplate 發送HTTP Request 的工具 。

安裝 dependency
```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.13</version> <!-- or the latest version -->
</dependency>
```

建立 HttpComponentsClientHttpRequestFactory 來替換HTTP Client

```java

    @Bean
    public CloseableHttpClient httpClient() {
        return HttpClientBuilder.create()
                .setMaxConnTotal(100)      // Set required maximum total connections
                .setMaxConnPerRoute(20)    // Set required maximum connections per route
                .build();
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient()));
    }
```

 <br/>

## 設定 Connection Pool Manager

在OSI 網路七層協定中 , HTTP 是建構於TCP 之上的通訊協定, 如果每次請求都需要重新建立一次 Connection , 想必會非常的消耗資源 (TCP 三向交握的關係) 。因此Apache Http Client 提供了Connection Pool (連線池的機制) 。

概念上與AP Server 在跟資料庫連線時的Connection Pool 類似, 都是先Keep 住Connection 不立即關閉 , 等下次有Request 來時就會借用那個Connection 來發送請求 , 大幅減少了每個Request 都需要花費建立Connection 的資源 。

至於如何設定請看下面程式碼說明

```java
    @Bean
    public PoolingHtppClientConnectionManager customizedPoolingHtppClientConnectionManager(){
      /*
      設定每個Connection 在Connection Pool 中的維持時間 , 範例為 5分鐘 , 此參數須小心設定
      */
      PoolingHtppClientConnectionManager connManager = new PoolingHtppClientConnectionManager(5, TimeUnit.MINUTES);
      connManager.setMaxTotal(100);    // Set required maximum total connections
      connManager.setDefaultMaxPerRoute(20);     // Set required maximum connections per route
    }
```

- max total : connection Pool 最大的總連線數量, 預設為20

- defaultMaxPerRoute: 每個路徑最大的連線數量, 預設為2

  

## 設定 Keep Alive  機制

在HTTP 中, Keep -Alive的機制為讓一個Connection 在一定時間內可以發送多個Request 。為了避免server side 已關閉連線,  但client 端卻還是維持該Connection 的情況 (**Connection Reset by Peer**)。透過設定Keep -Alive Header 來告知Http Client Connection 要維持的時間。

 通常若server端未給予keep alive 的timeout , 預設可以設定 30 或 60 秒來維持

```Java
   public ConnectionKeepAliveStrategy connectionKeepAliveStrategy() {
        return new DefaultConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
                long keepAliveDuration = super.getKeepAliveDuration(response, context);
                if (keepAliveDuration < 0) {
                    return DEFAULT_KEEP_ALIVE_TIME_MILLIS;
                }
                return keepAliveDuration;
            }
        };
    }
```

設定排程檢查並關閉 無效 or Idle timeout 的連線

```java
	@Bean
    public Runnable idleConnectionMonitor(final PoolingHttpClientConnectionManager connectionManager) {
        return new Runnable() {
            @Override
            @Scheduled(fixedDelay = 10000)
            public void run() {
                try {
                    if (connectionManager != null) {
                        LOGGER.trace("run IdleConnectionMonitor - Closing expired and idle connections...");
                        connectionManager.closeExpiredConnections();
                        connectionManager.closeIdleConnections(CLOSE_IDLE_CONNECTION_WAIT_TIME_SECS, TimeUnit.SECONDS);
                    } else {
                        LOGGER.trace("run IdleConnectionMonitor - Http Client Connection manager is not initialised");
                    }
                } catch (Exception e) {
                    LOGGER.error("run IdleConnectionMonitor - Exception occurred. msg={}, e={}", e.getMessage(), e);
                }
            }
        };
    }	
```



## 設定請求參數

```java
    private RequestConfig requestConfig(){
        RequestConfig requestConfig =
                RequestConfig.custom()
                        .setConnectionRequestTimeout(REQUEST_TIMEOUT)
                        .setConnectTimeout(CONNECT_TIMEOUT)
                        .setSocketTimeout(SOCKET_TIMEOUT)
                        .build();
        return requestConfig;
    }
```

- connection request timeout : client 端從connection pool 取得連線的最大時效
- connection timeout : 建立連線前的最大時效
- socket timeout : client 等待server 端回應資料的最大時效

## 完整設定範例

```java
@Configuration
@Log4j2
@RequiredArgsConstructor
public class HttpClientConfig {

    // Determines the timeout in milliseconds until a connection is established.
    private static final int CONNECT_TIMEOUT = 30000;
    // The timeout when requesting a connection from the connection manager.
    private static final int REQUEST_TIMEOUT = 30000;
    // The timeout for waiting for data
    private static final int SOCKET_TIMEOUT = 60000;

    private static final int DEFAULT_KEEP_ALIVE_TIME_MILLIS = 20 * 1000;
    private static final int CLOSE_IDLE_CONNECTION_WAIT_TIME_SECS = 30;

    private final EnvironmentConfig environmentConfig;

    @Value("${http.conn-pool.max-total-conn:100}")
    private Integer maxTotalConnection;

    @Value("${http.conn-pool.max-per-route-conn:20}")
    private Integer maxPerRouteConnection;

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager() {
        PoolingHttpClientConnectionManager poolingConnectionManager =
                new PoolingHttpClientConnectionManager();
        poolingConnectionManager.setMaxTotal(maxTotalConnection);
        poolingConnectionManager.setDefaultMaxPerRoute(maxPerRouteConnection);
        return poolingConnectionManager;
    }

    public ConnectionKeepAliveStrategy connectionKeepAliveStrategy() {
        return new DefaultConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
                long keepAliveDuration = super.getKeepAliveDuration(response, context);
                if (keepAliveDuration < 0) {
                    return DEFAULT_KEEP_ALIVE_TIME_MILLIS;
                }
                return keepAliveDuration;
            }
        };
    }
    
    @Bean
    public CloseableHttpClient httpClient() {
        RequestConfig requestConfig =
                RequestConfig.custom()
                        .setConnectionRequestTimeout(REQUEST_TIMEOUT)
                        .setConnectTimeout(CONNECT_TIMEOUT)
                        .setSocketTimeout(SOCKET_TIMEOUT)
                        .build();
        DefaultProxyRoutePlanner proxyRoutePlanner = proxyRouter();

        HttpClientBuilder httpClientBuilder =
                HttpClients.custom()
                        .setDefaultRequestConfig(requestConfig)
                        .setConnectionManager(poolingConnectionManager())
                        .setKeepAliveStrategy(connectionKeepAliveStrategy())
                        .setConnectionManagerShared(true);
        if (proxyRoutePlanner != null) {
            httpClientBuilder.setRoutePlanner(proxyRoutePlanner);
        }
        return httpClientBuilder.build();
    }

    @Bean
    public Runnable idleConnectionMonitor(
            final PoolingHttpClientConnectionManager connectionManager) {

        return new Runnable() {
            @Override
            @Scheduled(fixedDelay = 10000)
            public void run() {
                try {
                    if (connectionManager != null) {
                        log.trace(
                                "run IdleConnectionMonitor - Closing expired and idle connections...");
                        connectionManager.closeExpiredConnections();
                        connectionManager.closeIdleConnections(
                                CLOSE_IDLE_CONNECTION_WAIT_TIME_SECS, TimeUnit.SECONDS);
                    } else {
                        log.trace(
                                "run IdleConnectionMonitor - Http Client Connection manager is not initialised");
                    }
                } catch (Exception e) {
                    log.error(
                            "run IdleConnectionMonitor - Exception occurred. msg={}, e={}",
                            e.getMessage(),
                            e);
                }
            }
        };
    }
 /*
      設定thread pool ,讓idle monitor的thread 能被執行
*/
	@Bean
	public TaskScheduler taskScheduler() {
		ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
		scheduler.setThreadNamePrefix("poolScheduler");
		scheduler.setPoolSize(50);
		return scheduler;
	}
}
```



## 參考資料
- [Apache 官方文件 -  Connection management](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html)
- [謎之聲對 Connection 說道: 你已經死了！](https://medium.com/starbugs/%E8%AC%8E%E4%B9%8B%E8%81%B2%E5%B0%8D-connection-%E8%AA%AA%E9%81%93-%E4%BD%A0%E5%B7%B2%E7%B6%93%E6%AD%BB%E4%BA%86-b53d27c7ecb7)
- [什麼是Keep-Alive模式](https://byvoid.com/zht/blog/http-keep-alive-header/)
- [[30 天學會 Web 前端效能優化] 5. 淺談 HTTP 協定](https://ithelp.ithome.com.tw/articles/10156092)
- [Apache HttpClient Connection Management](https://www.baeldung.com/httpclient-connection-management)
- [Configuring HttpClient with Spring RestTemplate](https://howtodoinjava.com/spring-boot2/resttemplate/resttemplate-httpclient-java-config/)
- [REST client with desired NFRs using Spring RestTemplate](https://www.dhaval-shah.com/rest-client-with-desired-nfrs-using-springs-resttemplate/)
- [How to improve performance of Spring RestTemplate?](https://medium.com/@nitinvohra/how-to-improve-performance-of-spring-resttemplate-6af37e0a0f33)
- [Using RestTemplate with Apaches HttpClient](https://springframework.guru/using-resttemplate-with-apaches-httpclient/)
- [Avoid Spring RestTemplate Default Implementation to Prevent Performance Impact](https://dev.to/akdevcraft/never-use-spring-resttemplate-default-implementation-2ghj)
- [Apache Http Client Connection Management](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html#d5e418)

