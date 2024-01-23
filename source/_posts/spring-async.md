---
title: Spring 非同步程式設計
date: 2024-01-20 23:51:47
tags:
  - [Spring Boot]
  - [Java]
  - [Async]
categories:
  - [Spring]
description: 本篇將介紹如何使用 Spring @Async 來實現以往複雜的非同步程式設計。從使用的場合與時機、如何啟用設定到開發非同步程式都有基本的介紹, 相信透過本篇文章, 讀者能更加認識如何使用Spring 來完成非同步的程式設計。
---



## 前言

為了改善應用程式部分功能的效能 , 像是呼叫多個第三方API , 資料處理...等等, 往往會用非同步設計的方式達到並行處理的效果。但若採用Thread, Runbbable ,Callable 這些寫法來實現, 會產出許多笨重且較難以維護的程式碼, 因此Spring @Async 來改善程式的可讀性與設計方式。

本篇會將介紹Spring @Async 與Java 8 CompletableFuture的組合來介紹非同步程式設計的方式。 

## 概念介紹

非同步程式相當於創建一個新的子執行緒 來執行, 執行緒(Thread) 會循序等待每個子執行緒執行完任務，並獲取相關物件值，再次進行相關邏輯運算後，才會回傳最終結果。

而在Spring Boot中, 透過 @Async  來標註方法來告知Spring 需要建立子執行緒來執行 ,並且在方法回傳該執行緒完成的資料, 方法回傳值都透過Future實現 。 因此可以結合JDK 8 推出的 CompletableFuture 功能來做到複雜的商業邏輯處理。

## 應用時機

非同步程式主要是應用於效能優化,但各個資料的狀態互相獨立的場合 , 一個利用資源換取時間的概念。像是加速批次資料的處理, 呼叫多個獨立的API  端點, 資料庫的資料查詢 ,信件通知等等。

## 基本啟用設定

在配置類別加上 `@EnableAsync` 啟用非同步方法功能, 並建立一個名稱為`executor`的執行緒池。當標註@Async 時可使用建立好的執行緒來執行非同步方法。 若都未設定executor , Spring 預設SimpleAsyncTaskExecutor`來執行非同步方法。

```java
@EnableAsync
@Configuration
public class AsyncConfig {
    @Value("${spring.async.core-pool-size:30}")
    private Integer corePoolSize;

    @Value("${spring.async.max-pool-size:50}")
    private Integer maxPoolSize;

    @Value("${spring.async.queue-capacity:80}")
    private Integer queueCapacity;

    @Bean(name = "taskExecutor")
    public Executor executor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setTaskDecorator(new MDCTaskDecorator());
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

```



## 建立非同步的方法

後續在業務邏輯層 (@Service ,@Component ) 標註@Async 來告訴 Spring 該方法是一個非同步執行的任務。若方法需要回傳資料, 將資料封裝在 CompletableFuture.completedFuture(results) 進行回傳 , 方便使用該任務的其他方法做後續的處理(等待Thread 執行, 額外資料轉換等)。

程式範例為模擬判斷找到不同字串的電影 。

### Service 層

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

@Service
public class AsyncService {

  private static final Logger logger = LoggerFactory.getLogger(AsyncService.class);

  private List<String> movies =
      new ArrayList<>(
          Arrays.asList(
              "Forrest Gump",
              "Titanic",
              "Spirited Away",
              "The Shawshank Redemption",
              "Zootopia",
              "Farewell ",
              "Joker",
              "Crawl"));

  @Async
  public CompletableFuture<List<String>> completableFutureTask(String start) {
    logger.warn(Thread.currentThread().getName() + "start this task!");
    List<String> results =
        movies.stream().filter(movie -> movie.startsWith(start)).collect(Collectors.toList());

    try {
      Thread.sleep(1000L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return CompletableFuture.completedFuture(results);
  }    
}
```

###  Controller 層

建立一個API 端點來測試撰寫的非同步方法是不是用不同的Thread 來處理資料。

```java
@RestController
@RequestMapping("/async")
public class AsyncController {
  @Autowired 
  AsyncService asyncService;

  @GetMapping("/movies")
  public String completableFutureTask() throws ExecutionException, InterruptedException {
    long start = System.currentTimeMillis();
    List<String> words = Arrays.asList("F", "T", "S", "Z", "J", "C");
    List<CompletableFuture<List<String>>> completableFutureList =
        words.stream()
            .map(word -> asyncService.completableFutureTask(word))
            .collect(Collectors.toList());
    // CompletableFuture.join（）
    List<List<String>> results = completableFutureList.stream().map(CompletableFuture::join).collect(Collectors.toList());
    // print consume time
    System.out.println("Elapsed time: " + (System.currentTimeMillis() - start));
    return results.toString();
  }
}
```

啟動應用程式後 ,Postman或瀏覽器對`http://localhost:8080/async/movies`發出`GET`請求。

應用程式收到請求並執行後會在console印出下面結果。可以觀察到該端點會新增不同的執行緒來實現非同步執行。

```
My ThreadPoolTaskExecutor-1start this task!
My ThreadPoolTaskExecutor-6start this task!
My ThreadPoolTaskExecutor-5start this task!
My ThreadPoolTaskExecutor-4start this task!
My ThreadPoolTaskExecutor-3start this task!
My ThreadPoolTaskExecutor-2start this task!
Elapsed time: 1010
```



## 總結

此篇文章簡單介紹了SpringBoot 如何改善複雜的非同步程式設計, 並介紹如何啟用Spring Boot 非同步的功能 , 並如何實際開發出非同步的程式, 最後則是有一個小小的Demo 來顯示非同步的功能是如何啟用的, 至於如何處理回傳的CompletableFuture , 筆者後續會再詳細介紹有哪些語法可以使用 !!!


## 參考資料
- [JDK8 - CompletableFuture 非同步處理簡介](https://www.tpisoftware.com/tpu/articleDetails/1484)
- [How To Do @Async in Spring](https://www.baeldung.com/spring-async)
- [Spring Boot @Async 非同步方法範例](https://matthung0807.blogspot.com/2020/06/spring-boot-async-methods-example.html)
- [今晚我想來個Spring Async非同步的感覺](https://ithelp.ithome.com.tw/articles/10278638?sc=iThelpR)
- [CompletableFuture](https://popcornylu.gitbooks.io/java_multithread/content/async/cfuture.html)
- [CompletableFuture 使用介绍](https://www.javadoop.com/post/completable-future)
- [新手也能看懂的 SpringBoot 异步编程指南](https://github.com/CodingDocs/springboot-guide/blob/master/docs/advanced/springboot-async.md)
