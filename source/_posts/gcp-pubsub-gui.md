---
title: Google Cloud Pub/Sub Console 操作
date: 2024-01-05 10:47:30
tags:
  - [Message Queue]
  - [Async]
  - [GCP]
categories:
  - [GCP]
description: 本篇文章主要介紹如何透過GCP 提供的Console 介面來操作Pub / Sub 服務, 簡單建立topic 及相關的subscription , 並展示發送訊息後, subscription 接收到的內容。
---



## 前言

本篇文章主要介紹如何透過GCP 提供的Console 介面來操作Pub / Sub 服務 , 如果對Pub/Sub 的概念還不熟悉的朋友, 

建議可以閱讀以下的文章進行了解.

<ol>
<li>{% post_link gcp-pubsub %}</li>
</ol>

## 啟用Pub/Sub 管理功能

按一下側邊攔，把 Pub/Sub 功能找出來

![]()

## 設定主題 (Topic)

1. 進入 Pub/Sub 管理頁面有，找到主題(Topic)，選建立主題

   ![]()

2. 設定一下主題名稱，名稱會變成一個主題 ID 提供訂閱使用 /project/專案名稱/topics/主題名稱

   ![]()

3. 確認一下主題生成的狀態

   ![]()

4. 點進去 Topic 名稱後可以看狀態

   ![]()

## 設定訂閱項目 (Subscription)

設定 Subscription 作為接受Topic 傳遞訊息的監聽項目 , 本身有提供篩選機制來接收特內容的訊息。

後續的其他系統整合時也是去監聽Subscription 有無收到Topic 傳來的資料 (Subscriber)

1. 進入 Pub/Sub 管理頁面有，找到訂閱項目( Subscription )，選建立訂閱項目

   ![]()

2. 設定訂閱項目

   - 訂閱項目名稱: 會生成訂閱項目 ID `projects/專案名稱/subscriotions`

   - 選取 Cloud Pub/Sub 主題: 選擇訂閱的 Topic 主題(這邊就選剛剛的 Topic 做測試)

   - 傳送類型:

     - 提取: 程式去提取資料個概念
     - 推送: 設定一個 https 基底的 API 提供給 subscription 在接收資料時呼叫使用，也可適用於 GCP 內部服務開出的 API (如: Cloud Run, GKE, GCS, GCE)

   - 訂閱項目有效期: 訂閱項目的活躍保留天數 (多久沒傳遞訊息，之後會自動刪除的天數)

   - 確認期限: 訊息被 subscriber 捕抓或是提取後，需要多久的反應回饋時間，從提取到確認的時間差(確認服務處理的狀態)，這邊可以作為事件被提取後，作完後續流程的確認，超過時間差就會返回到 Message Pool，提供給其他 Subscriber 提取實作。

   - 訂閱項目篩選器: 篩選你的訊息內容，之後才抓進來到你的 subscription 之中

     - `attributes.<item-name> = <item-info>`
     - `hasPrefix(<item-name>, <item-info>)`

   - 訊息保留時間: 訊息被發送後，到 subscription 且未被 subscriber 提取或是使用，保留在 Message Pool 的時間

   - 無效信件: 他會轉發你的訊息到其他的 topic 之上的設定

     

## 參考資料
- [基於付費公有雲與開源機房自建私有雲之雲端應用服務測試兼叢集與機房託管服務實戰之勇者崎嶇波折且劍還掉在路上的試煉之路](https://ithelp.ithome.com.tw/users/20121070/ironman/3552)

