---
title: Java Apache HttpClient4.X Connection Reset 問題
date: 2024-08-02 23:19:36
tags:
  - [Java]
  - [HTTP]
  - [Apache]
categories:
  - [Java]
description: 本篇文章提到Java 常用呼叫API 工具Apache HTTP Client 常發生的Connection reset 問題, 並列出可能的原因與對應之解法, 希望讀者可透過參考此篇文章來快速找出如何解決connection  reset的方法。
---
## 前言

若要透過Java 呼叫 API 端點， 可透過多種不同的程式工具進行呼叫，像是Apache HTTP Client 、OkHTTP、WevFlux等。今天將介紹使用Apache HTTP  Client 開發所遇到的坑，以及對應的解決辦法。

## 發生原因

由於因業務邏輯需求要呼叫多次API 。可能會重複使用建立的HTTP Client Instance 來去呼叫API, 若API 的時間連線時間太長， 可能會出現TCP Idle connection ， 導致程式無法順利執行完成，拋出Socket Exception:  Connection Reset 問題

以下為發生的程式碼範例

```java
public static void main(String[] args) throws IOException {
        HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();
        List<String> urlList = new ArrayList<>();
        //  multiple urls using single HttpClient
        try (CloseableHttpClient httpClient = httpClientBuilder.build()) {
            for (String url : urlList) {

                HttpGet getRequest = new HttpGet(url);
                try (CloseableHttpResponse response = httpClient.execute(getRequest)) {
                    HttpEntity entity = response.getEntity();
                    if (entity != null && response.getStatusLine().getStatusCode() == 200) {
                        String result = EntityUtils.toString(entity);
                        //  handle response logic ...
                    }
                } catch (IOException e) {
                    log.error(e.getMessage(), e);
                }
            }
        } catch (IOException e) {
            log.error(e.getMessage(), e);
        }
    }
```

<br/>

##   解決辦法

### 解法一

單次 request 就 重新new  一個  `CloseableHttpClient`  。

然後使用完畢後就close (try with resources )。這樣的做法為每次進行呼叫API 都重新進行一次TCP Connection。

依照官法文件說法 ,每次重新建立一個新的HTTP Client Instance是成本很高的事情, 因此假設程式若有效能需求的情況下。此解決方法可能不盡理想

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@Slf4j
public class HttpClientExample {
    public static void main(String[] args) throws IOException {
        HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();

        List<String> urlList = new ArrayList<>();
        for (String url : urlList) {
            try (CloseableHttpClient httpClient = httpClientBuilder.build()) {
                HttpGet getRequest = new HttpGet(url);
                try (CloseableHttpResponse response = httpClient.execute(getRequest)) {
                    HttpEntity entity = response.getEntity();
                    if (entity != null && response.getStatusLine().getStatusCode() == 200) {
                        String result = EntityUtils.toString(entity);
                        //  handle response logic ...
                    }
                } catch (IOException e) {
                    log.error(e.getMessage(), e);
                }

            } catch (IOException e) {
                log.error(e.getMessage(), e);
            }
        }

    }
}

```



### 解法二

使用connection pool 的方式來驗證connection 是否為idle connection, 同時也能減少多次呼叫API 時所吃的資源與效能。

設定Connection Pool 參數來定期驗證TCP Connection是否為過期或無效的Connection。

```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setValidateAfterInactivity(500);
CloseableHttpClient httpclient = HttpClients.custom()
            .setConnectionManager(cm)
            .evictExpiredConnections()
            .evictIdleConnections(5L, TimeUnit.SECONDS)
    	    .setRetryHandler(DefaultHttpRequestRetryHandler.INSTANCE)
            .build();

```

補充說明

- max per route : connection manager 預設每個domain 最大的connection數量為5

- evictExpiredConnections 與 evictIdleConnections 用於設置在背景中清理過期的connection

- setValidateAfterInactivity : 每次取得連線時, 假設該連線空閒超過該時間, 則會驗證是否可用。默認值為2000ms

- 設定重試機制: 預設試3次, 但假設pool 中的max per route 是五個connection ,可能還是會出現例外。最好的保險是重是次數需大於MaxPerRoute ,保證都失效後, 重新進行連接

  

## 參考資料

- [Handle java.net.SocketException: Connection reset more gracefully](https://issues.apache.org/jira/browse/HTTPCLIENT-2282)
- [apache httpclient 連接配接池 工具_HttpClient連接配接池的一些思考](https://www.laitimes.com/article/486os_4ou1m.html)
- [HttpClient连接池的连接淘汰策略分析，以及解决HttpNoResponse异常](https://www.cnblogs.com/wusanga/p/17392445.html)
- [Apache HttpClient throws java.net.SocketException: Connection reset if I use it as singletone](https://stackoverflow.com/questions/70175836/apache-httpclient-throws-java-net-socketexception-connection-reset-if-i-use-it)
- [HTTP Client](https://hc.apache.org/httpclient-legacy/performance.html) 
