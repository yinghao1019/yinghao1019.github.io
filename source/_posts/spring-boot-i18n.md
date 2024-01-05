---
title: SpringBoot i18n 國際化設定與包裝
date: 2024-01-05 16:49:27
tags:
  - [Java]
  - [Spring Boot]
  - [i18n]
categories:
  - [Spring Boot]
description:
---



## 前言
### **多語系網站的好處**

在網路世界裡，使用者沒辦法親眼看到公司實際的樣子，官網就是一個形象的展現，網站有不同的語系除了更顯國際化，頁面的瀏覽上也顯得更親切；
對很多人來說，使用自己不熟悉的語言，會感覺陌生，容易產生不信任感。若能讓潛在客戶使用自己語言瀏覽網站，也能讓訪客覺得這家公司是一間相當有規模的公司，提高在網站上的體驗，也有機會提高轉化率。

### Spring Boot 多語系介紹, 環境配置

Spring Boot自動配置[MessageSource](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/MessageSource.html)做i18n多國語言訊息，不用再自己配置MessageSource的bean且預設會尋找classpath根目錄下名稱為messages的properties作為訊息來源。如果想要使用其他名稱或路徑，可在Spring Boot配置檔application.properties設定。

### Spring套件安裝

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
		
<!-- https://mvnrepository.com/artifact/com.alibaba/fastjson -->
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.73</version>
</dependency>
```

## 系統設定

1. 設定Message Source 讀取位置
2. 設定 Resolver
3. 

## 讀取Locale Message



## 參考資料

- [Spring boot 的 I18n 設定與進一步包裝](https://bingdoal.github.io/backend/2021/12/i18n-internationalization-in-spring-boot/)
- [Spring Boot快速配置多語系(國際化)](https://www.tpisoftware.com/tpu/articleDetails/2347)
- [Spring Boot 多語系設置( i18n – 國際化)](https://polinwei.com/spring-boot-i18n/)
- [Spring Boot MessageSource 自動配置](https://matthung0807.blogspot.com/2020/06/spring-boot-messagesource-auto-config.html)
- [Guide to Internationalization in Spring Boot |Baledung](https://www.baeldung.com/spring-boot-internationalization)
- [How to Internationalize a Spring Boot Application](https://reflectoring.io/spring-boot-internationalization/)
- [Spring Boot internationalization i18n: Step-by-step with examples](https://lokalise.com/blog/spring-boot-internationalization/)

