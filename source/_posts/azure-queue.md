---
title: Azure Queue Storage 介紹與操作
date: 2024-01-05 11:46:58
tags:
  - [Queue]
  - [Async]
  - [Azure]
categories:
  - [Azure]
description: Azure Queue Storage 適合用於儲存大量的訊息, 但又不會需要長期保存時的情境, 主要被應用在非同步的訊息交換, 由本篇文章帶你認識如何進行操作已及Queue的原理介紹。
---



## 前言

Azure Queue Storage 適合用於儲存大量的訊息, 但又不會需要長期保存時的情境, 一個Queus Storage 可以純存約百萬則訊息 ,

存取訊息的方式支援HTTP 與HTTPS 。 如果要使用Azure Queue Storage, 則需在Azure Storage Account中進行建立與管理。

<img src="https://3.bp.blogspot.com/-bykMEF0qNHA/WwqssskmT4I/AAAAAAAAgJM/2ujpeckpHlskoBXWooE-Ry5lKaEJ8OuIACLcBGAs/s640/801.jpg" alt="Azure Queue" style="zoom:67%;" />

<br/>

## 特性

### **Azure Queue Storage** 

- FIFO : 先儲存的會優先被取出來處理, 但由於是軟刪除, 所以發生concurrency 不一定會保持此特性
- 每一筆訊息最大容量為64 KB , 儲存期限為7天
- 一個queue 可容納 500TB的資訊
- 易於與Azure 其他服務整合, 擴張 (Azure Function )
- 支援HTTP ,HTTPS 通訊協定存取

### **儲存的訊息**

- 可以是UTF -8 字串 或是 二進位制 ( Byte Arrays) 的格式
- XML 文件, CSV ,TSV檔案等

<br/>

## 應用時機

![image-20240105162500590](C:\Users\Howard.YH.Hung\AppData\Roaming\Typora\typora-user-images\image-20240105162500590.png)



主要被應用在於非即時回應的非同步處理 , 想降低不同系統之間的耦合性, 或是做為後端伺服器的系統緩衝。

- Email , SMS 的訊息發送
- 後端Server 紀錄Log 資料的管道
- 微服務系統之間的溝通橋樑

<br/>

## 實作介紹

因為筆者主要使用Java 開發 ,接下來Queue的實作皆會已Java 語法進行介紹常用的操作:

1. Storage Account 建立 Queue
2. 設定Queue 連線與授權資訊
3. 傳送訊息至 Queue
4. 取出Queue中的訊息
5. 更新Queue的訊息
6. 刪除Queue的訊息
7. 計算Queue 中訊息的數量
8. Queue 的例外處理

<br/>

1.  Storage Account 建立 Queue 

   登入Azure Portal 後, 選擇Storage Account , 點選側邊欄的Queue 後按上面的 + 進行新增

   <img src="" alt="queue operation" style="zoom: 67%;" />

2. 設定Queue 連線與授權資訊

   設定Queue 服務的url 與認證方式 , 認證方式主要有三種 (分別為 SAS Token , Connection String , Storage Account Key) , 

   取得認證資訊方式可參考官方文件[Authenticate the client](https://learn.microsoft.com/en-us/java/api/overview/azure/storage-queue-readme?view=azure-java-stable#enqueue-message-into-a-queue) 

   ```java
   String queueURL = String.format("https://%s.queue.core.windows.net/%s", ACCOUNT_NAME, queueName);
   QueueClient queueClient = new QueueClientBuilder().endpoint(queueURL).sasToken(SAS_TOKEN).buildClient();
   ```

3. 傳送訊息至 Queue

   ```java
   String queueURL = String.format("https://%s.queue.core.windows.net", ACCOUNT_NAME);
   QueueClient queueClient = new QueueClientBuilder().endpoint(queueURL).sasToken(SAS_TOKEN).queueName("myqueue")
           .buildClient();
   
   queueClient.sendMessage("myMessage");
   ```

4. 取出Queue中的訊息

   取出Queue的Message 有提供其他額外參數設定, 由以下多做說明:

   - maxMessages: 批次從queue 中取出訊息的最大數量 。最大為32 筆, 超過設定會出現exception 
   - visibilityTimeout: 取出後隔多少時間才能再次看到該 Message, 預設為30秒
   - timeout: 與queue連線多久沒回應的時效限制
   - context: 使用服務前, 需要額外新增近Http Pipeline的資訊

   ```java
   String queueURL = String.format("https://%s.queue.core.windows.net", ACCOUNT_NAME);
   QueueClient queueClient = new QueueClientBuilder().endpoint(queueURL).sasToken(SAS_TOKEN).queueName("myqueue")
           .buildClient();
   queueClient.receiveMessages(10).forEach(message ->
       System.out.println(message.getBody().toString()));
   ```

   

5. 更新Queue的訊息

   ```java
   // @param messageId: Message 的Id
   // @param popReceipt: 用於辨識哪個Message 要被更新
   // @param visibilityTimeout: 更新後多久才能再看到
   queueClient.updateMessage(messageId, popReceipt, "new message", visibilityTimeout);
   ```

   

6. 刪除Queue的訊息

   ```java
   String queueURL = String.format("https://%s.queue.core.windows.net", ACCOUNT_NAME);
   QueueClient queueClient = new QueueClientBuilder().endpoint(queueURL).sasToken(SAS_TOKEN).queueName("myqueue")
           .buildClient();
   
   queueClient.deleteMessage(messageId, popReceipt);
   ```

   

7. 計算Queue 中訊息的數量

   ```java
   QueueClient queueClient =
           QueueUtils.createQueueClient(queueName, QueueUtils.getDefaultConnString());
   queueClient.getProperties().getApproximateMessagesCount();
   ```

   

8. Queue 的例外處理

   Azure SDK 有提供額外的Error Code 讓開發人員進行處理, 詳情請見 [Queue Storage error codes](https://learn.microsoft.com/en-us/rest/api/storageservices/queue-service-error-codes)

   ```java
   String queueServiceURL = String.format("https://%s.queue.core.windows.net", ACCOUNT_NAME);
   QueueServiceClient queueServiceClient = new QueueServiceClientBuilder().endpoint(queueServiceURL)
       .sasToken(SAS_TOKEN).buildClient();
   try {
       queueServiceClient.createQueue("myQueue");
   } catch (QueueStorageException e) {
       logger.error("Failed to create a queue with error code: " + e.getErrorCode());
   }
   ```

   

<br/>

## 補充說明

除了Azure Queue Storage , 微軟雲端有再推出另外一種Queue 型態的服務 - Azure Service Bus 。相較於Queue Storage , Service Bus 的功能與Rabbit MQ , Kafaka , GCP PubSub  較為類似 , 比較偏向Producer - Consumer 的架構 , 所以如果希望要較為即時的非同步處理功能, 請珍惜生命 , 不要走錯棚  !

## 參考資料

- [Azure Queue Storage 介紹與操作](https://dog0416.blogspot.com/2018/05/azure-azure-queue-storage.html)
- [Azure Queue Storage介紹 - IT幫幫忙](https://ithelp.ithome.com.tw/articles/10205084)
- [Azure Storage Queue client library for Java](https://learn.microsoft.com/en-us/java/api/overview/azure/storage-queue-readme?view=azure-java-stable#enqueue-message-into-a-queue)

