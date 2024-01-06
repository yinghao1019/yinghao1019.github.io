---
title: SpringBoot i18n 國際化設定
date: 2024-01-05 16:49:27
tags:
  - [Java]
  - [Spring Boot]
  - [i18n]
categories:
  - [Spring Boot]
description: 開發人員有可能會碰到需要開發多國語言的專案 , 透過不同語系的方式來提升使用者操作體驗。故本篇文章介紹了對於Spring Boot 人員如何進行設定與使用 i18n 套件來實現多語言的專案
---



## 前言
### **多語系網站的好處**

在網路世界裡，使用者沒辦法親眼看到公司實際的樣子，官網就是一個形象的展現，網站有不同的語系除了更顯國際化，頁面的瀏覽上也顯得更親切；
對很多人來說，使用自己不熟悉的語言，會感覺陌生，容易產生不信任感。若能讓潛在客戶使用自己語言瀏覽網站，也能讓訪客覺得這家公司是一間相當有規模的公司，提高在網站上的體驗，也有機會提高轉化率。

### Spring Boot 多語系介紹, 環境配置

Spring Boot自動配置 [MessageSource](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/MessageSource.html) 做i18n多國語言訊息，不用再自己配置`MessageSource`的bean且預設會尋找classpath根目錄下名稱為messages的properties作為訊息來源。 如果想要使用其他名稱或路徑，可在Spring Boot配置檔 application.properties設定。

### 套件安裝

springBoot 對 i18n的支援是含在 `spring-context-support` 裡的, 因此僅需引用 *web-starter*  模組, 不需要再額外引用其他的 `xxx - starter`

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```yaml
implementation 'org.springframework.boot:spring-boot-starter-web'
```

<br>

## 系統設定

1. ### 設定Message Source 讀取位置 

   在spring boot 中, 需要在properties設定讀取建立的 Message檔案路徑位置 。

   application.properties
   
   ```properties
   spring.messages.basename=i18n/messages
   spring.messages.encoding=UTF-8
   spring.messages.cache-duration=3600
   ```
   
   預設會是 basename 代表語系檔的路徑以及檔名，依照這裡的設定須將語系檔放在 resource/i18n 之下，且命名為 message.properties
   
   - message_zh_TW . properties
   - message_en_US . properties
   - message.properties

![](https://i.imgur.com/e3HHl7o.png)

2. ### 設定解析器(Resolver)與攔截器 (Interceptor)

   如果不想使用用設的[*MessageSourceAutoConfiguration*](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/context/MessageSourceAutoConfiguration.java)  。  

   可以自行客製化設定解析語系名稱的清單, 語系暫存的位置 以及要如何切換語系的方法

   ```java
   @Configuration
   public class LocaleConfig {
       
       @Bean
       public ReloadableResourceBundleMessageSource messageSource() {
           ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
           messageSource.setBasename("classpath:i18n/messages");
           messageSource.setDefaultEncoding("UTF-8");
           messageSource.setCacheSeconds(3600); // Cache for an hour
           return messageSource;
       }
       /**
       * Spring Boot 預設採用 AcceptHeader Resolver
       */
       @Bean
       public LocaleResolver localeResolver() {
           // 設定支援的 Locales，若 Accept-Language 沒設定或不在這清單內就會使用預設的 Locale
           List<Locale> supportedLocales = new ArrayList<>();
           supportedLocales.add(Locale.TAIWAN);
           supportedLocales.add(Locale.ENGLISH);
   
           AcceptHeaderLocaleResolver acceptHeaderLocaleResolver = new AcceptHeaderLocaleResolver();
           acceptHeaderLocaleResolver.setDefaultLocale(Locale.TAIWAN); // 預設 Locale
           acceptHeaderLocaleResolver.setSupportedLocales(supportedLocales);
           return acceptHeaderLocaleResolver;
       }
       
       /**
       * 默認攔截器 客製化設定切換語系時的參數名.
       */
       @Bean
       public LocaleChangeInterceptor localeChangeInterceptor() {
           LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
           lci.setParamName("lang");
           return lci;
       }
       
       @Override
       public void addInterceptors(InterceptorRegistry registry){
           registry.addInterceptor(localeChangeInterceptor());
       }
   }
   ```
   
   <br>
   
   程式中是使用 AcceptHeaderLocaleResolver 來提取當前Client 想用的語系, 其實還有其他常見的解析方式
   
   | Locale Resolver           | 說明                                                         |
   | ------------------------- | ------------------------------------------------------------ |
   | **CookieLocaleResolver**  | 將使用者的語系偏好資料暫存於 瀏覽器的Cookie 中               |
   | **SessionLocaleResolver** | 將使用者的語系偏好資料暫存於 Session 中                      |
   | **FixedLocaleResolver**   | 固定當前的語系 , 不會依照使用者設定的語系偏好改變, 用於Debug |
   
   <br>

## 讀取Locale Message

當設定完成後 , 若要在程式讀取設定的訊息, 可注入 `MessageSource` 的資源，透過 key 值去對應訊息，

如果後面帶入的語系不存在就會用預設的 message.properties 內容

```java
@Slf4j
@RestController
public class DemoController {

    @Autowired
    private MessageSource messageSource;
    @Autowired
    private UserDao userDao;

    @GetMapping("/{userId}")
    @ResponseStatus(HttpStatus.OK)
    public User getOneUser(@PathVariable("userId") Long userId) throws Exception {
        Optional<User> userOption = userDao.findById(userId);
        if (userOption.isEmpty()) {
            String msg = messageSource.getMessage(
                "user.controller.not.found.by.id",
                new String[]{userId.toString()},
                Locale.TAIWAN);
            log.error(msg);
        }
        return userOption.get();
    }
}
```

上面的範例是直接寫入 Locale.TAIWAN 來套用語系，實際情境下通常是透過 API 帶入語系參數來決定訊息的語系，

因此可以改寫成

```java
// @param code: 設定要取Message 的Key
// @param args: 作為變數替換Message 中的區塊
// @param Locale: 讀取Message 的語系檔
String msg = messageSource.getMessage("user.controller.not.found.by.id",
                                      new String[]{userId.toString()}, LocaleContextHolder.getLocale())
```

`LocaleContextHolder.getLocale()` 會從 request 的 header 欄位 Accept-Language 來解析語系。

如果語系找不到會先抓主機的預設語系，若預設語系也不存在才會去抓 `message.properties`

### **Message 參數化**

Message 支援帶入程式的變數來動態替換訊息 , 在訊息來源檔中的要嵌入的變數位置, 將變數傳入至 `MessageSource.getMessage()` 的第二個參數做替換

```properties
demo.message=MessageSource自動配置
demo.message.args=帶參數的訊息，參數0={0}, 參數1={1}
```

<br>

## 建立成共用元件

使用原生的語法來取message 比較不方便, 而且而且語系的參數基本上就是帶入 `LocaleContextHolder.getLocale()`。 

因此將使用`MessageSource`的邏輯封裝成Utils來簡化程式碼

```java
public class LocaleUtils {

    private static MessageSource messageSource;

    private LocaleUtils() {
    }

    public static void setMessageSource(MessageSource messageSource) {
        LocaleUtils.messageSource = messageSource;
    }

    public static String get(String msgKey) {
        return LocaleUtils.get(msgKey, (Object) null);
    }

    public static String get(String msgKey, Object... args) {
        try {
            return messageSource.getMessage(msgKey, args, getLocale());
        } catch (Exception e) {
            throw new InternalServerErrorException("翻譯失敗:" + msgKey + ", " + e.getMessage());
        }
    }

    public static MessageDTO getMessage(String msgKey) {
        return LocaleUtils.getMessage(msgKey, (Object) null);
    }

    public static MessageDTO getMessage(String msgKey, Object... args) {
        MessageDTO messageDTO = new MessageDTO();
        messageDTO.setMessage(LocaleUtils.get(msgKey, args, getLocale()));
        return messageDTO;
    }


    private static Locale getLocale() {
        return LocaleContextHolder.getLocale();
    }
}
```



## 參考資料

- [Spring boot 的 I18n 設定與進一步包裝](https://bingdoal.github.io/backend/2021/12/i18n-internationalization-in-spring-boot/)
- [Spring Boot快速配置多語系(國際化)](https://www.tpisoftware.com/tpu/articleDetails/2347)
- [Spring Boot MessageSource i18n 範例](https://matthung0807.blogspot.com/2020/06/spring-boot-messagesource-i18n-example.html)
- [Spring Boot 多語系設置( i18n – 國際化)](https://polinwei.com/spring-boot-i18n/)
- [Spring Boot MessageSource 自動配置](https://matthung0807.blogspot.com/2020/06/spring-boot-messagesource-auto-config.html)
- [Guide to Internationalization in Spring Boot |Baledung](https://www.baeldung.com/spring-boot-internationalization)
- [How to Internationalize a Spring Boot Application](https://reflectoring.io/spring-boot-internationalization/)
- [Spring Boot internationalization i18n: Step-by-step with examples](https://lokalise.com/blog/spring-boot-internationalization/)
- [Spring Boot REST: Internationalization (i18n) Example](https://howtodoinjava.com/spring-boot/rest-i18n-example/)
