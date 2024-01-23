---
title: Spring 非同步程式設計
date: 2024-01-20 23:51:47
tags:
  - [Spring Boot]
  - [Java]
  - [Async]
categories:
  - [Spring]
description:
---



## 前言

為了改善應用程式部分功能的效能 , 像是呼叫多個第三方API , 資料處理...等等, 往往會用非同步設計的方式達到並行處理的效果. 
但若採用Thread, Runbbable ,Callable 這些寫法來實現, 會產出許多笨重且較難以維護的程式碼, 因此Spring @Async 
來改善程式的可讀性與設計方式。
本篇會將介紹Spring @Async 與Java 8 CompletableFuture的組合來介紹非同步程式設計的方式。 


## 參考資料
- [JDK8 - CompletableFuture 非同步處理簡介](https://www.tpisoftware.com/tpu/articleDetails/1484)
- [How To Do @Async in Spring](https://www.baeldung.com/spring-async)
- [Spring Boot @Async 非同步方法範例](https://matthung0807.blogspot.com/2020/06/spring-boot-async-methods-example.html)
- [今晚我想來個Spring Async非同步的感覺](https://ithelp.ithome.com.tw/articles/10278638?sc=iThelpR)
- [CompletableFuture](https://popcornylu.gitbooks.io/java_multithread/content/async/cfuture.html)
- [CompletableFuture 使用介绍](https://www.javadoop.com/post/completable-future)

