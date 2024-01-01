---
title: 單元測試基本概念
date: 2023-12-29 09:47:44
tags:
- [unit test]
categories:
- [測試]
---



單元測試的目的在於測試每個class 的function是不是如期運轉,故撰寫目的在於測試在A情境(test case)下的Input/Output是不是如預期所想的。透過寫單元測試,也能幫助我們設計程式的撰寫習慣,避免利用一個function 封裝全部的業務邏輯。
<!-- more -->

## 單元測試流程

1. 釐清對象
2. 設計測試案例(Test case)
3. 測試環境準備(JUnit)
4. 相依物件隔離(Test Double & Mockito)
5. 測試結果(JUnit & AssertJ & Mockito)
6. 測試案例管理(JUnit & JaCoCo)



## 釐清對象與設計測試案例

首先, 撰寫單元測試前 , 需先釐清SUT , DOC 的名詞概念意義 :

- System under test (SUT)  : 要進行功能測試的元件 (Class)
- Depended On Component (DOC) :  測試物件所需要用到其他功能的元件  (Class)

因此在概念上, 僅需思考 SUT 的元件是誰, 以及要如何進行測試 , 其餘 SUT 裡面用到的 DOC 元件則是需要將它進行隔絕。 

這些DOC 元件則會在他們的單元測試中測試,  只要DOC 在自己的單元測試沒問題, 那在SUT裡也不會有問題。

找出了SUT, 就可以開始根據 SUT 設計相對應的 **測試案例(Test Case)** , 為了確保功能的完整性 , 需設計多個測試案例來測試功能。

**測試案例的準則為, 只要該物件的輸入不同, 結果也是不同的情況,  基本上就能拆分出一個測試案例。**

舉例來說,代測功能中具有處理例外的情況, 此時例外情況通常輸入的值會與正常時不同。因此例外與正常執行的情況皆為不同的**測試案例(Test case)。**

而Test Case的設計大多會 Follow 3A Pattern，如下:

```java
@Test
public void testMethodNameReturnWhat()
{
   //Arrange    
    
   //Act   
  
   //Assert
}
```

測試方法的命名方式為test+測試的方法+預期回傳值。 執行內容為測試內容的準備(Arrange)  /執行 (Act)  / 驗證 (Assert)。

- Arrange : 預先設計測試案例 的輸入資料為何,  包含一些測試替身 等。

- Act :  如何執行要測試的功能之實作部分

- Assert: 驗證該方法輸出的資料是否符合預期 or 方始使用DOC 的次數等等

  

## 測試環境準備( JUnit )

當開始執行測試時, 需要做的事前準備輸出報告 (測試案例成功與否) 與環境 (是否要先準備假資料，或是啟動Web container)，與當結束後是否要進行清理(可能測試寫檔需要刪除測試產生的檔案)，這個環節Java通常都使用JUnit套件輔助撰寫，在後面的系列文會有詳細的JUnit介紹。



## 相依物件隔離(Test Double ＆ Mockito)

為了不讓DOC物件影響SUT測試的結果,故需要使用隔離方法來排除使用的DOC物件。

通常隔離的概念為Test Double(測試替身),而隔離的DOC因為被使用的功能主要分為兩類:

Test Double包含了：

物件: Dummy & Stub

作出Test Double的方法: Mock & Spy & Fake

### • Dummy — 取代不在乎細節的物件

要被測試的方法中,如被測試的方法實作並不會呼叫DOC物件的方法 or 不需要DOC物件的資料來完成時作功能,像是只是簡單地去計算數量,則該DOC物件就為此類別。

### • Stub — 讓DOC提供SUT想要的Input/Output的物件

Stub則是DOC的物件內容會影像到SUT的method執行結果,即為Stub。

### • Mock — 都是假的

由於Stub物件會影響我們的測試結果,故利用Mock的方式來模擬DOC物件,故需要設定我們輸入的值以及對應預期測試情境下要輸出的值來進行替換。

### • Spy — 監控DOC與SUT的互動

Spy主要功能為用來檢視Mock與DOC之間的交互作用。

### • Fake — 環境有限制我只好在寫一個

其實有Mock和SPY方法在單元測試就涵蓋了99%的覆蓋了，而Fake就是真實寫一個簡單的邏輯取代原本得邏輯(Mock和Spy都是直接輸出需要的值Fake則要寫邏輯)，舉例:Database的使用，在單元測試使用H2(In-memory database)，或是在內網開發環境不能使用外網時，SSL憑證檢查工具可以寫一個Fake，改寫成去讀隨機以準備好在Resource的憑證.txt

其實在Wiki上這五種都是物件，但是若是以物件的角度去看五種Tese Double很可能會在最後三種Mock/Spy/Fake搞混，因為這三種是可以重疊的，可以對DOC做Mock在做SPY，模擬加上監控，也可以對Fake做Mock只取其中幾個邏輯做模擬，其他方法用簡單的邏輯實踐。



## 測試結果 (JUnit / AssertJ / Mockito)

當測試環境,案例,Test double的模擬方式都定義好了,需要確認這樣的情境下所執行的結果是不是如同我們的預期,藉此來驗證有效性。

**測試結果通常會聚焦在三點:**

1. 待測目標(SUT)的輸出結果是否符合預期
2. 相依物件(DOC)是否有符合預期的被呼叫，而呼叫次數是否準確
3. 整個Test Case是否有涵蓋到完整的SUT?

前兩點，在Java會透過JUnit / AssertJ / Mockito 協助完成，而第三點我們就要討論到Unit Test很重要的觀念，覆蓋率。

覆蓋率就是單元測試執行結束後，SUT有多少行程式碼有執行到，而執行到的程式碼/全部程式碼的百分比就是覆蓋率。而覆蓋率也細分了一些種類，這邊只列三個基本的介紹

1. Statement coverage — 程式碼每一行覆蓋
2. Branch coverage — SUT中的每個if else是不是都有進去過
3. Condition coverage — 每個會產生true or false的判斷是不是都有跑到過

Branch 和 Condition常常會搞混，直接看個範例

```java
if (a > b || b > c) {
    //do something
} else {
    //do something
}
```

上面的程式碼，if其實只要做到a > b就可以進入，這是Branch coverage，而Condition Coverage則是a > b || b > c的這兩個條件都要跑到過才能算是覆蓋成功。

最常用的覆蓋率工具(IDEA/JaCoCo提供的覆蓋率)其實沒有上面如此複雜，非常單純與直觀：

1. Class - 系統裡多少Class被跑到
2. Function - Class裡多少Function被跑到
3. Line - Function裡多少程式被跑到

## **測試案例管理 (JUnit & JaCoCo)**

前面講的都是單元測試個體，這個環節討論的是群，當系統內的單元測試個體都跑完後，觀看整個系統的覆蓋率，以及更改功能後要跑哪些相關連的單元測試都包含在此。而良好CI/CD流程都是建立在完善的測試案例管理，監測覆蓋率，執行單元測試群體，這裡的實做也會在後面的系列文介紹，會包含如何將單元測試分組(跑一組單元測試裡面的個體都會執行)，與如何使用 JaCoCo 產出覆蓋率報表。



## **單元測試常用工具介紹**

- JUnit - Java單元測試的基礎，許多操作都是建立在此之上，包含(Unit Test的環境建立與結果檢驗)
- AssertJ - 是好用的結果檢驗工具，功能比JUnit原生的提供的還多
- Mockito - Test Double建立的好夥伴，Mockito讓開發者不用在自己寫Test Double的物件
- JaCoCo - 覆蓋率報告產生工具
