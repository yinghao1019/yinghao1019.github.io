---
title: Azure Function 介紹
date: 2024-01-06 17:10:05
tags:
  - [Azure]
  - [Serverless]
categories:
  - [Azure]
description: Azure Function 是一種無伺服器 (Serverless) 解決方案, 目的在於減少開發人員維護, 管理基礎設施的成本與時間, 僅需撰寫少量的程式碼就能快速建構服務。
             因此最常用來回應事件，如資料庫變更、IoT 資料流、訊息佇列等。本篇文章將帶你認識Azure Function 基礎概念。
---



## 前言

Azure Functions 是一種無伺服器 (Serverless) 解決方案, 目的在於讓開發人員專注於開發業務邏輯的程式, 減少維護基礎設施的成本與時間。除了Azure Function , 其他兩大雲端供應商皆有推出對應的解決方案 (AWS Lambda , Google Cloud Functions) , 各有其優缺點。

而本篇文章將會帶你認識 Azure function 基礎的觀念, 在搭配其他文章來練習實作。

## 功能介紹

Azure Function 是一個在雲端輕鬆執行一小段程式碼或功能的解決方案，你只需要撰寫手頭上問題所需的程式碼，而不需要擔心整個應用程式、執行環境與基礎架構。執行業務程式的單位被稱為Azure Function App , 可以在Function App 中撰寫多個不同情境中要執行的程式,如下圖所示。

<img src="https://i.imgur.com/JUPIMVq.png" alt="img" style="zoom: 50%;" />

<br>

Azure Function 帶來的優勢與功能有:

1. **多種語言選擇**：C#、F#、Node.js、Python、PHP、Batch、Bash 或任何可執行程式。 您可以在 Azure Portal 內設定 Azure Function
2. **使用付費**：只有程式碼執行期間需要付費。Azure Function 也支援 NuGet 與 NPM，您可以加入自己喜歡的 lib
3. **安全性**：透過 OAuth 程序保護 Http 觸發函式 (如： Azure Active Directory，Facebook，Google，Twitter 和 Microsoft 帳戶)
4. **簡單整合**：輕鬆地整合 Azure Service 與 SaaS 產品
5. **靈活開發**：可以在 portal 上編輯程式。或透過 Github、Visual Studio Team Services 和其他支援開發工具設定 持續整合 或 佈署程式碼。
6. **彈性擴展**:  支援依照需求大小來自動擴展執行Function 的host , 在某些方案中還可以調整要擴展的方式

## 方案說明

當你要建立Azure Function App 專案時, 你必須選擇合適的方案 : Consumption Plan , Premium Plan , App Service Plan

不同的 Plan提供了多種不同的優勢, 請依照自己的需求來決定Plan, 以下簡單說明各Plan 的概念, 詳情請看官方文件進行比較。

1. **Consumption (情況方案)**：預設值。只需支付程式碼執行時間相對的費用, 計算單位是以百萬執行來計費。執行期間可以依據需求添加 instance 來自動擴大 CPU 與記憶體。即時在高負載期間也能向外擴展。但缺點為無法與Azure Virtual Network 整合, 因此在管理ip 的黑白名單較為麻煩。
2. **Preminum (高級方案)** : 相較於情況方案, 提供更好的instance 選擇來執行function , 還提供預熱的方法來應對高流量的問題。並且能與Virtual Network 整合, 但費用較為昂貴, 是以類似租vm 的方式進行計算
3. **App Service (應用程式方案)**：相較於Consumption, Premim。 能與其他應用程序共享同個 App Service。也提供設定如何擴展執行Function 的Instance ,但費用相較於Preminum 更為昂貴, 但提供更多的資源來執行Function 

> 請注意 , azure function 免費方案的instance 在經過一段時間未有事件觸發時 , 會進入休眠狀態, 故再次使用功能時會執行效率會比較久 。
>
> 而其他兩種方案都有支援暖機的功能。因此對效能方面有需求的讀者, 請斟酌使用對應的方案

## 應用場景與支援

Azure Function 也支援多種不同的系統整合, 常用來回應事件，如資料庫變更、IoT 資料流、訊息佇列等。以下整理多種不同的應用

- Build a web API
- Process file uploads
- Build a serverless workflow
- Respond to database changes
- Run scheduled tasks
- Create reliable message queue systems
- Analyze IoT data streams
- Process data in real time

### 範例

1. Web application 後端處理： Web App 產生 Request → 將 Request 放入 Service Bus 佇列 → 透過 Azure Function 取出處理後 → 將資料放入 Cosmos DB

   ![img](https://3.bp.blogspot.com/-aiYtyvt_amc/W2cHiMJnTaI/AAAAAAAAgy0/J5imedDEKbkhgUjq4IyhHQ1It5zd8tJJgCLcBGAs/s1600/%25E6%258A%2595%25E5%25BD%25B1%25E7%2589%25871.JPG)

2. 即時檔案處理：將 PDF 檔案上傳至 Blob Storage 內 → 透過 Azure Function 處理 → 送至認知服務 (Cognitive Service) 進行處理 → 將取得資訊放入 SQL DB

   ![img](https://2.bp.blogspot.com/-LaV7Tgc-gWY/W2cHiFNH3AI/AAAAAAAAgy4/bmx2C1atCHAx2Yz9qeCEh4hlBVp6eyqOQCLcBGAs/s1600/%25E6%258A%2595%25E5%25BD%25B1%25E7%2589%25872.JPG)

3. 自動排程工作：Azure Function 每 15分鐘清除資料庫重複資料

![img](https://1.bp.blogspot.com/-PBEuas48vRk/W2cHiWMaiiI/AAAAAAAAgy8/G5PwjEMCHGgfqmJ8w-tohxyaOCFr5dIOQCLcBGAs/s1600/%25E6%258A%2595%25E5%25BD%25B1%25E7%2589%25873.JPG)

## 參考資料

- [Azure Function 結合 Line Bot 玩出新花樣](https://www.tpisoftware.com/tpu/articleDetails/1243)
- [Azure Functions⚡介紹及Serverless入門輕鬆學](https://ithelp.ithome.com.tw/m/articles/10202960)
- [Azure Function: 事件驅動式的雲端應用](https://dotblogs.com.tw/regionbbs/2016/04/01/introduction-to-azure-functions)
- [初探 Azure Functions](https://hackmd.io/@blueskyson/azure-functions)
- [Azure 背景處理服務介紹 2 : WebJob 與 Azure Function](https://dog0416.blogspot.com/2018/08/azure-azure-2-webjob-azure-function.html)
