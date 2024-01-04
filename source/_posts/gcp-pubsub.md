---
title: Google Cloud Pub/Sub 介紹
date: 2024-01-04 23:58:26
tags:
  - [Message Queue]
  - [Async]
  - [GCP]
categories:
  - [GCP]
description: Cloud Pub/Sub 為 Google 推出的 Message Queue service, 如同Rabbit MQ , Apache Kafka 等Message Queue系統. 但卻簡化了整合與開發的門檻, 透過本篇文章能快速了解 Pub/Sub的原理與使用方式。
---

## 前言
Cloud Pub/Sub 為 Google 推出的 message service，主要用途是讓每個獨立的應用(Application)間能透過 Publish-Subscribe 的模式來進行訊息交換與溝通，一般而言利用 message service 
當作中介層(Middleware)來傳遞訊息，有著以下幾項優/缺點:

<br/>

### 優點

- 透過非同步的訊息傳遞，降低 Publisher、Subscriber 間的耦合度。意即彼此間無需知道對方位置，亦不會任意一方出現問題而導致連鎖反應。
- 當作訊息緩衝區(Buffer)，避免後端消化速度不夠快而無法接收新進的訊息請求。
- 根據不同用途來訂閱/散佈訊息。

### **缺點**

- 由於是非同步處理，因此訊息的即時性/順序性/重覆性無法受到保證。

- 需要熟悉 message service 服務的遞送流程，避免異常或訊息無法正確傳送。

  <br/>

就先前經驗來說，一個高可用/彈性的 message service，通常會考慮以下幾點:

- 訊息傳遞效率
- 可擴展性(Scalability)、可靠性(Reliability)、可用性(Availability)

為方便開發者瞭解及使用，Cloud Pub/Sub 將 AMQP ([Advanced Message Queuing Protocol](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol))  中 Message Broker、Exchange、Queue 等不會直接被開發者接觸的部分隱藏起來大幅降低進入門檻，開發者僅需要瞭解 Topic、Publisher、Subscriber、Push/Pull Subscription 即可撰寫相關程式。

<br/>

## 架構介紹

### ***❖主要名詞認識***

- **Message**: 要傳送的 data

- **Topic**: 主題，是一個可以被訂閱訊息的實體

- **Subscription**: 訂閱，連結 Topic 跟 Subscriber 之間的實體，接收以及處理發佈訊息到 Topic

- **Publisher**: 發布者，提供並且發送訊息到 Topic 的單位

- **Subscriber**: 訂閱者， 訂閱訊息的一個單位

  <br/>

### **❖系統流程**

![](https://i.imgur.com/TsAvWUi.png)

上圖為 [Google 官方](https://cloud.google.com/pubsub/docs/overview) 介紹 Pub/Sub 的流程，主要流程如下:

1. Publisher 首先在 Cloud Pub/Sub 建立傳訊息用的 Topic，然後開始向該 Topic 傳送訊息
2. 當訊息被接收前或尚未收到 Acknowledge(Ack) 時，會被保存起來並等待在次傳送出去
3. Subscriber 向服務註冊訂閱(Subscription)後，所有發送到 Topic 的訊息會轉發給該 Topic 下的所有 Subscriber
4. Subscriber 收到訊息後會回傳 Ack 訊息給 Cloud Pub/Sub，以確認訊息已經收到
5. 當 Ack 被 Cloud Pub/Sub 收到後，將該訊息自 Message Storage 刪除

<br/>

### **❖情境案例**

![](https://i.imgur.com/7latY55.png)

- A, B 同時可以 publish message 到相同的 Topic
- subcriber可以同時接收多個subscription傳遞來的Message
- Topic可以有多個subscription

<br/>

## SubScription

### **❖操作方式**

**對訂閱者來說有兩種處理 Message 的方式，分別為 PULL 跟 PUSH.**

- `[push](<https://cloud.google.com/pubsub/docs/push>)`
- 送一個 request 給 App 的 endpoint 說我要傳訊息來。
- 以這個 endpoint return `[200, 201, 204, or 102]` 來判定為 ack, 如果不是就會一直被打直到這個訂閱所設置的最大 retention time 為止
- 動態調整 push 的 request，根據拿到的狀態碼來調整。
- `[pull](<https://cloud.google.com/pubsub/docs/pull>)`
- 視為被動的取得訂閱佇列 (subscription queue) 中的 Message

**兩者機制要怎麼選用，有以下建議**

- `push`
- 低流量情形 (< 10,000/second)
- Legacy push webhook
- App Engine 的訂閱者
- `pull`
- 大量的訊息 (many more than 1/second)
- 效能跟訊息遞送因素很重視者
- 公開的 Https Endpoint

<br/>

### **❖生命週期**

- 31 天內沒有被 pull or push 就會被自動刪除，或者經過手動操作被刪除
- 訂閱的名字沒有絕對關係，用同樣的名字也會被視為兩者不同的訂閱(情境可能是：刪除前，刪除後)
- 刪除後就算有大量還沒寄出的訊息，或者是 Backlog，都與新建立的無關

<br/>

## 注意事項

使用 Cloud Pub/Sub 時需要小心以下幾點：

- 若 Ack 因為意外或超時而尚未傳到 Cloud Pub/Sub 時該訊息會至多保留 7 天(請參考 [Retry Policy](https://cloud.google.com/pubsub/docs/subscriber))，因此程式撰寫時需考慮到訊息延遲遞送的問題，比方說加上時間戳來過濾超時的訊息。
- 為求傳遞效能，服務不保證訊息的順序性
- 訊息可能會發生重覆(duplicate)的情況，面對訊息重覆有兩種處理方式
  - 若該請求`不能`重覆執行(比方說銀行扣款)，就需要針對訊息夾帶的 uuid 進行重覆性確認
  - 若該請求`可以`重覆執行(比方說查看銀行餘額)，程式便能忽略重覆性確認的流程，除非重覆次數太多導致系統性能受到影響
- 訊息在尚未接收完畢前切勿刪除該 Topic ，避免 Subscriber 無法接收其他存留的訊息
- 建議 Subscriber 在起來時，產生其專用的 uuid 用來向服務註冊訂閱，避免名稱衝突



## 參考資料

- [Google Pubsub官方文件](https://cloud.google.com/pubsub/docs/overview)
- [初探 Google Cloud Pub/Sub](https://tachingchen.com/tw/blog/google-cloud-pubsub-introduction/)
- [認識 Google Cloud Pub/Sub](https://kylinyu.win/pubsub/#-subscriber-操作面)
- [GCP 事件觸發驅動訊息推播 - Cloud Pub/Sub](https://ithelp.ithome.com.tw/articles/10206624)
- [GCP 公有雲_雲端事件消息傳遞服務實戰 - Pub/Sub 組建測試之路](https://ithelp.ithome.com.tw/articles/10249308)

