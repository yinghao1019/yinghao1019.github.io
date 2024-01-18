---
title: Spring Boot 重試機制
date: 2024-01-15 22:50:19
tags:
  - [Spring Boot]
  - [Java]
categories:
  - [Spring Boot]
description: 網路應用程式中, 往往會出現不預期的錯誤與異常 , 通常透過重試的方式可能就解決了. 因此Java Spring Boot 提供 Spring Retry Module 讓開發人員輕鬆實現重試機制,藉此簡化開發過程。本文將介紹 Spring Retry 的使用方法 ,注意事項以及範例。
---
## 前言

在網路應用程式中,  可能會發生一些情境 , 例如: 傳送訊息失敗，呼叫遠端服務忙碌, 網路不穩定, 等等...  這些程式突然異常的狀況往往讓他自動重試可能就正常了。因此在Spring Boot 中, 提供 Spring Retry 的Module 幫助我們實現重試與復原的機制, 讓撰寫的功能更加穩定, 並且簡化實現重試機制的程式碼。

## Spring Retry 設定

### 安裝spring retry 

spring retry 是基於 AOP 的方式來實現的功能, 因此在安裝時需引入AOP 相關套件

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
    </dependency>
</dependencies>
```

### 啟用Spring Retry

再@Configuration class 上使用@EnableRetry 

```java
@EnableRetry
@Configuration
public class RetryConfig { ... }
```

或主程式上使用

```java
@SpringBootApplication
@EnableRetry
public class DemoApplication {

   public static void main(String[] args) {
      SpringApplication.run(DemoApplication.class, args);
   }
}
```



## 使用 @Retryable 實現重試機制

使用@Retryable非常簡單，只需要在要進行重試的方法上添加@Retryable注解，然後設置相關的重試參數即可。但由於底層使用AOP 的概念實作, 因此重試機制只適用於 Spring 管理的Bean 物件上 , 並且在同個Class 中被呼叫的話也會失效。

### @Retryable的參數介紹

1. value：指定需要重試的異常類型。如果沒有指定，則默認重試所有異常。可以指定多個異常類型，以陣列的形式提供。
2. maxAttempts：指定最多重試次數。默認值為3。
3. backoff：指定重試間隔時間。可以指定一個@Backoff註釋對象，這個對象有兩個屬性：
   - delay：指定重試間隔時間，默認為0毫秒。
   - multiplier：指定每次重試的間隔時間增加倍數，默認為1。
4. include：指定需要重試的異常類型。與value類似，但是是針對於某些異常類型進行重試。
5. exclude：指定不需要重試的異常類型。與value類似，但是是針對於某些異常類型不進行重試。

### 使用方法與範例

```java
@Retryable(value = {RestClientException.class}, maxAttempts = 3, backoff = @Backoff(delay = 1000))
public ResponseEntity<String> sendRequest(String url, HttpMethod method, HttpEntity<?> requestEntity, Class<String> responseType) throws RestClientException {
    return restTemplate.exchange(url, method, requestEntity, responseType);
}
```

上面的範例中, 當方法拋出為 RestClientException 時, Spring Retry 會再次呼叫該方法, 最多執行3 次, 並且每次間隔 1000 ms來進行重試。

### @Recover

當重試至最大的次數時卻還是未成功, 可能會需要額外執行的後備方案來進行錯誤處理 ,像是 紀錄錯誤Log , 將失敗資訊寫入資料庫 ,資料庫的交易 Roll back等等 。

因此 Spring Boot 提供 @Recover 功能提供開發人員撰寫復原的方案機制。

> 該@Recover 的 method 參數與回傳直需要與重試的相同才會被呼叫 

```java
@Recover
public ResponseEntity<String> recover(RestClientException e) {
	logger.error("Recover...");
    // fallback implementation
}
```



## 使用 RetryTemplate 進行重試

如果想要要同一個@Bean class 中的Method 使用retry 機制, 可以透過 RetryTemplate 來開發。

### 設定Retry Template的重試機制

```java
@EnableRetry
@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate retryTemplate() {
    RetryTemplate retryTemplate = new RetryTemplate();

    //Fixed delay of 1 second between retries
    FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
    fixedBackOffPolicy.setBackOffPeriod(1000l);
    retryTemplate.setBackOffPolicy(fixedBackOffPolicy);

    //Retry only 3 times
    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
    retryPolicy.setMaxAttempts(3);
    retryTemplate.setRetryPolicy(retryPolicy);

    return retryTemplate;
  }
}
```

### RetryTemplate  使用範例

透過注入RetryTemplate , 並將想要重試的程式碼區塊利用Lambda 語法作為RetryTemplate.execute()的參數來實現

```java
@Autowired
RetryTemplate retryTemplate;

template.execute(ctx -> {
    return backendAdapter.getBackendResponse(...);
});
```


## Retryable 失效的原因
假設發現 `@Retryable` 不生效，可能有以下一些原因：

1. 忘記加上 `@EnableRetry` 注解：在 Spring Boot 中，如果想要使用 `@Retryable` 注解，必須在 Spring 配置類上加上 `@EnableRetry` 注解，啟用 Spring Retry 的支持。
2. 未include 到例外 ：如果在 `@Retryable` 注解中指定的例外類型與方法中抛出的例外不相符，那麼重試操作就不會發生，可以嘗試在 `@Retryable` 注解中新增需要重試的例外類型。
3. 重試策略設定錯誤：Spring Retry 預設的重試策略是重試 3 次，每次重試之間等待 1 秒。如果需要自定義重試策略，可以實現 `RetryPolicy` 接口，並在 `@Retryable` 注解中指定重試策略。
4. 事務設定錯誤：如果被 `@Retryable` 注解標註的方法中包含事務操作，那麼可能需要進一步設定事務管理器以支援重試操作。
5. 不在 Spring 管理的Bean 上使用：`@Retryable` Annotation 只能用於 Spring 管理的物件上，如果在非 Spring 管理的物件上使用，則重試操作不會生效。

## 參考資料
- [spring-retry 重試機制](https://www.tpisoftware.com/tpu/articleDetails/1407)
- [如何在 SpringBoot 中使用 Retry & Cache](https://ithelp.ithome.com.tw/articles/10191550)
- [Java Spring Boot – @Retryable 重試機制](https://carger.tips/java-spring-boot-retryable-%E9%87%8D%E8%A9%A6%E6%A9%9F%E5%88%B6)
- [Spring Boot Retry Example](https://howtodoinjava.com/spring-boot2/spring-retry-module/)
- [Guide to Spring Retry](https://www.baeldung.com/spring-retry)
- [Spring Boot Retry: How to Handle Errors and Retries in Your Application](https://bootcamptoprod.com/spring-boot-retry/)
