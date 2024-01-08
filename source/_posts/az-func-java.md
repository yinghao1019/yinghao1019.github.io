---
title: Azure Function 實作教學
date: 2024-01-08 09:22:25
tags:
  - [Azure]
  - [Serverless]
  - [Java]
categories:
  - [Azure]
description: 本篇文章專注於教學如何使用Java 建立 Azure function 專案 , 從基本的專案環境建置 、如何開發業務邏輯程式、部屬到Azure Function App 皆會有詳盡的介紹 。幫助讀者快速的了解、建構出Azure Function 的服務 。
---



## 前言

本篇文章專注於教學如何使用Java 建立 Azure function 專案 , 從基本的專案環境建置 、如何開發業務邏輯程式、部屬到Azure Function App 皆會有詳盡的介紹 。幫助讀者快速的了解、建構出Azure Function 的服務 。

在開始進行教學前, 建議先閱讀Azure Function 的基本概念會比較好上手:

<ol>
<li>{% post_link az-func-basic %}</li>
</ol>

<br>

## 開發環境

Azure function 支援多種方式 (Maven , Eclipse , Vs Code )建立專案, 而本文選擇使用Intellj Idea 進行開發:

- JDK 8 , 11 or 17
- 安裝 Maven 3.5.0 +
- 安裝 Azure CLI 2.4 或更新版本
- 安裝 Azure Functions Core Tools 4.x 版

在使用Intellj 開發前, 需安裝額外的外掛程式並登入

Plugin 搜尋  Azure ToolKit for Intellj  -> 安裝並重啟 -> 上方列選擇 tool  -> azure sign in 讓local 環境取得雲端資源存取權限

<img src="C:\Users\Howard.YH.Hung\AppData\Roaming\Typora\typora-user-images\image-20240108150903026.png" alt="image-20240108150903026" style="zoom:67%;" />

<br>

## 建立專案

如同建立 spring Boot 專案, 在new project 選擇Azure Functions並填入專案相關資訊即可創立。



### 專案結構

```
FunctionsProject
 | - src
 | | - main
 | | | - java
 | | | | - FunctionApp
 | | | | | - MyFirstFunction.java
 | | | | | - MySecondFunction.java
 | - target
 | | - azure-functions
 | | | - FunctionApp
 | | | | - FunctionApp.jar
 | | | | - host.json
 | | | | - MyFirstFunction
 | | | | | - function.json
 | | | | - MySecondFunction
 | | | | | - function.json
 | | | | - bin
 | | | | - lib
 | - pom.xml
 | - host.json
 | - local.settings.json
```

此為Java 的專案範例結構 , 由於是透過maven 進行管理 , 結構上大致與Spring Boot 等Maven 專案結構相同。各資料夾說明如下:

-  FunctionApp : 原始碼根目錄. 依照業務功能需求可再拆成sub package 管理, 要執行的Azure Function 也會存放於此。
-  host.json :  執行Azure Function App 的系統參數設定。 ex: log 設定, 監控設定 , trigger 的全域設定
-  local.settings.json : 本機 Debug 模式時所讀取的設定, 設定資訊如同Azure Function 的App Settings.
-  target : 執行maven 後的打包檔, 包含azure function 對應的資訊, 第三方套件等 .... 

<br>

## 範例程式碼說明

撰寫程式碼前, Azure Function 有兩個核心的概念需認識:

- Trigger  : 用於設定撰寫的程式碼被觸發的條件, Azure 支援多種不同的方式. Ex: Http , Timer , Queue ,Blob
- Binding : 當希望程式被觸發後有額外的輸入或輸出 , 可用此功能做設定 



```java
package com.howhow.functions.handler;

import com.microsoft.azure.functions.*;
import com.microsoft.azure.functions.annotation.*;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class DemoFunction {
  @FunctionName("QueueOutputPOJOList")
  public HttpResponseMessage QueueOutputPOJOList(
      @HttpTrigger(
              name = "req",
              methods = {HttpMethod.POST},
              authLevel = AuthorizationLevel.ANONYMOUS)
          HttpRequestMessage<Optional<String>> request,
      @QueueOutput(
              name = "itemsOut",
              queueName = "test-output-java-pojo",
              connection = "AzureWebJobsStorage")
          OutputBinding<List<TestData>> itemsOut,
      final ExecutionContext context) {
    context.getLogger().info("Java HTTP trigger processed a request.");

    String queueMessageId = request.getBody().orElse(null);
    itemsOut.setValue(new ArrayList<TestData>());
    if (queueMessageId != null) {
      TestData testData1 = new TestData();
      testData1.id = "msg1" + queueMessageId;
      TestData testData2 = new TestData();
      testData2.id = "msg2" + queueMessageId;

      itemsOut.getValue().add(testData1);
      itemsOut.getValue().add(testData2);

      return request.createResponseBuilder(HttpStatus.OK).body("Hello, " + queueMessageId).build();
    } else {
      return request
          .createResponseBuilder(HttpStatus.INTERNAL_SERVER_ERROR)
          .body("Did not find expected items in CosmosDB input list")
          .build();
    }
  }

  public static class TestData {
    public String id;
  }
}
```

此範例程式為建立一個HTTP 端口,  並將Request Body 收到的資料新增至設定Azure Queue Storage 中

- 第 11 行 :  此Annotation 用來定義Function 的名字。 若未使用此標註, 則在編譯時就不會產生這個Function 的 Route 
- 第 13 行 : 用於設定該function 要被哪種條件觸發 , 每個function都需要設定Trigger 的 Annotation 。 範例中則是透過HTTP 作為觸發 , 並綁定HttpRequestMessage 來取得相關請求資訊
- 第18行 : 用於設定該function 額外的輸入與輸出資訊 , 每個Function 可設定 0 至多個Binding 及相關綁定的參數 . 

<br/>

### 讀取Trigger  額外資訊

在業務邏輯程式若需要讀取除了Trigger  其他的額外資訊 , 可以透過  `@BindingName` 來設定要取得的Trigger meta data. 

各Trigger 可使用的Meta data 詳情請查看官方文件

```java
public class Function {
    
  @FunctionName("metadata")
  public static String metadata(
      @HttpTrigger(
              name = "req",
              methods = {HttpMethod.GET, HttpMethod.POST},
              authLevel = AuthorizationLevel.ANONYMOUS)
          Optional<String> body,
      @BindingName("name") String queryValue) {
    return body.orElse(queryValue);
  } 
}
```

在上述範例中，`queryValue` 會取得從HTTP 請求中的查詢字串參數 `name`。

以下是另一個範例，示範如何從Queue Trigger 取得相關Message 的 `Id`。

```java
  @FunctionName("QueueTriggerMetadata")
  public void QueueTriggerMetadata(
      @QueueTrigger(
              name = "message",
              queueName = "test-input-java-metadata",
              connection = "AzureWebJobsStorage")
          String message,
      @BindingName("Id") String metadataId,
      @QueueOutput(
              name = "output",
              queueName = "test-output-java-metadata",
              connection = "AzureWebJobsStorage")
          OutputBinding<TestData> output,
      final ExecutionContext context) {
    context
        .getLogger()
        .info(
            "Java Queue trigger function processed a message: "
                + message
                + " with metadaId:"
                + metadataId);
    TestData testData = new TestData();
    testData.id = metadataId;
    output.setValue(testData);
  }

```

<br/>

### Log 輸出

如果要在程式寫入Log 資訊, 則是透過ExecutionContext` 中定義的 `getLogger 。而Execution Context 除了Logger, 也能取得其他額外資訊,詳請請看官方文件

```java
  public String echo(
      @HttpTrigger(
              name = "req",
              methods = {HttpMethod.POST},
              authLevel = AuthorizationLevel.ANONYMOUS)
          String req,
      ExecutionContext context) {
    if (req.isEmpty()) {
      context
          .getLogger()
          .warning(
              "Empty request body received by function "
                  + context.getFunctionName()
                  + " with invocation "
                  + context.getInvocationId());
    }
    return String.format(req);
  }
```

<br/>

### 讀取外部參數 、環境變數

在專案開發中, 有些設定會依照不同環境來變動, 像是連線, 認證資訊 等 。這些資訊通常都會設定在App Settings , 

App Settings的資訊在執行期間會被視為環境變數, 因此可使用`System.getenv("AzureWebJobsStorage")` 來存取這些設定。

```java
  public String echo(
      @HttpTrigger(
              name = "req",
              methods = {HttpMethod.POST},
              authLevel = AuthorizationLevel.ANONYMOUS)
          String req,
      ExecutionContext context) {
    context.getLogger().info("My app setting value: " + System.getenv("myAppSetting"));
    return String.format(req);
  }
```

<br>

## 測試與除錯

### Plugin Debug

Intellj 的Azure Plugin 支援本機Debug  , 用法如同SpringBoot 除錯, 會模擬一個service 來進行除錯。

但在本機除錯時, plugin 會存取專案根目錄中的 `local .settings.json` 來做為 App Setting 

1. 在執行除錯前, 可至 設定中調整要讀取的App Setting  , 此服務佔用的port 或jvm 相關的參數

   <img src="C:\Users\Howard.YH.Hung\AppData\Roaming\Typora\typora-user-images\image-20240108112756092.png" alt="image-20240108112756092" style="zoom:67%;" />

2. 設定中斷點後執行Debug , 即可查看該中斷點以前的資訊

   ![]()

   

### 手動觸發Function 

開發的Azure Function 條件若需要在特定情境中才能被觸發, 但這個情境又難以達成時, 可以透過發送HTTP 請求的方式來進行觸發。

Ex: 排程, Queue ,Blob

1. 設定要觸發Function 的端點 , 規則如下:

   ![Define the request location: host name + folder path + function name](https://learn.microsoft.com/en-us/azure/azure-functions/media/functions-manually-run-non-http/azure-functions-admin-url-anatomy.png)

   - hostname : 設定你的function  網域名稱。本機的話就是Localhost

   - Function Name : 使用@FunctionName 設定的名字

   - HTTP method : 方法統一採用POST , content-type 則是 application/json

   - 認證資訊 : 部屬的function 因資安考量, 不會開放給所有人觸發。因此需從function 取得Mater key 並在 Http Header 中設定

     x-functions-key

     

2. 設定要帶的參數

   有些Trigger 會綁定一些輸入資訊, 此時在Body 中帶入 {"input":"${相關資訊}"}

   ![Postman body settings.](https://learn.microsoft.com/en-us/azure/azure-functions/media/functions-manually-run-non-http/functions-manually-run-non-http-body.png)

3. 如果有成功回應, 代表 function 被觸發

   ![Send a request with Postman.](https://learn.microsoft.com/en-us/azure/azure-functions/media/functions-manually-run-non-http/functions-manually-run-non-http-send.png)

<br/>

## 部屬到 Azure Function  App

Azure Function 支援多種部屬方式, 包括持續, 手動部屬的選項 .同時也支援容器化的部屬方法

**CD (持續部屬)**

- GitHub Action
- Azure Pipeline
- Jenkins 

**手動部屬**

- Azure CLI
- REST API
- Containers 

本篇文章僅會教學使用GitHub Action 完成CD , 使用Azure CLI 手動部屬的方法 , 剩下的方法請參照[官方文件閱讀](https://learn.microsoft.com/en-us/azure/azure-functions/functions-continuous-deployment) 

### GitHub Action 

1. 設定GitHub Workflow能存取Azure Function 資源的權限

   - 下載 publish profile 

   - 於Github Secret 中新增 `AZURE_FUNCTIONAPP_PUBLISH_PROFILE` 變數 , 並將`publish profile` 的內容貼至對應的值中

     ![Download publish profile](https://learn.microsoft.com/zh-tw/azure/azure-functions/media/functions-how-to-github-actions/get-publish-profile.png)

2. 於專案路徑 `/.github/workflows/` 設定github workflow 的 Yaml 檔 

   ```yaml
   name: Deploy Java project to Azure Function App
   
   on:
     push:
       branches:
         - main
     workflow_dispatch:
   
   env:
     AZURE_FUNCTIONAPP_NAME: 'your-app-name'   # set this to your function app name on Azure
     POM_XML_DIRECTORY: '.'                    # set this to the directory which contains pom.xml file
     JAVA_VERSION: '8'                         # set this to the java version to use (e.g. '8', '11', '17')
   
   jobs:
     build-and-deploy:
       runs-on: windows-latest
       environment: dev
       steps:
       - name: 'Checkout GitHub Action'
         uses: actions/checkout@v3
   
       - name: Setup Java Sdk ${{ env.JAVA_VERSION }}
         uses: actions/setup-java@v1
         with:
           java-version: ${{ env.JAVA_VERSION }}
   
       - name: 'Restore Project Dependencies Using Mvn'
         shell: pwsh
         run: |
           pushd './${{ env.POM_XML_DIRECTORY }}'
           mvn clean package
           popd
   
       - name: 'Run Azure Functions Action'
         uses: Azure/functions-action@v1
         id: fa
         with:
           app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
           package: '${{ env.POM_XML_DIRECTORY }}' # if there are multiple function apps in same project, then this path will be like './${{ env.POM_XML_DIRECTORY }}/target/azure-functions/${{ env.POM_FUNCTIONAPP_NAME }'
           publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}
           respect-pom-xml: true
   ```

3. 於Action 上檢視是否成功執行

   ![]()

### Azure CLI 部屬

1. 執行mvn clean install 打包專案

   ```bash
   mvn clean install
   ```

2. 將`./target/azure-functions`  中的project 資料夾轉成zip檔

3. 執行Azure cli 的以下指令進行部屬

   - -g : function 的resource group
   - -n: function app 的name
   - --src: 本機zip檔的位置

   ```bash
   az functionapp deployment source config-zip -g <resource_group> -n \
   <app_name> --src <zip_file_path>
   ```

4. 登入azure portal 查看部屬情況

   ![]()

<br/>

##  監控 Azure Function 

當程式上線後, 若發生重大異常, 錯誤, 開發人員很難去得知。因此Azure Function 提供整合Application Insight 的方法來進行持續監控 , 

透過設定好的預警門檻, 來觸發對應的信件通知系統來完成自動化的監控。

 筆者將會於下篇文章教學如何實現 Azure function 整合 Application insight的監控功能。

## 參考資料

- [Continuous deployment for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-continuous-deployment)
- [Azure Functions Java developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption)
- [Create your first Java function in Azure using IntelliJ](https://learn.microsoft.com/en-us/azure/azure-functions/functions-create-maven-intellij)
- [初探 Azure Functions](https://hackmd.io/@blueskyson/azure-functions#%E5%B8%83%E7%BD%B2%E5%88%B0-Azure-Functions)

