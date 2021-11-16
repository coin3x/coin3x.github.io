---
layout: post
date: 2021-11-17 00:00:00 +0800
title: "消失的通知紀錄"
tags: android notification
---
Android 11 在設定裡新增了通知紀錄的選項，你可以看到最近剛滑掉的通知，還有二十四小時內的通知紀錄，以往都要靠新增桌面小工具的方式才能叫出通知紀錄的頁面。

<figure>
<img src="{{site.baseUrl}}/assets/disappeared-notification-history/notification-history.png">
<figcaption>新的通知紀錄介面，雖然是 Android 12 的</figcaption>
</figure>

<figure style="display: flex;flex-direction: column;">
<video style="max-height: 600px;" src="{{site.baseUrl}}/assets/disappeared-notification-history/adding-notification-log-widget.mp4" controls></video>
<figcaption>如何透過桌面小工具存取舊的通知紀錄</figcaption>
</figure>

通知紀錄有個問題，有時候二十四小時內的那個 section 會突然消失，還蠻困擾的。

我發現在 section 消失的情況下點開通知紀錄，logcat 會噴錯：
```
W/NotificationBackend: Error calling NoMan
W/NotificationBackend: java.lang.NullPointerException: Attempt to invoke virtual method 'void android.graphics.drawable.Icon.writeToParcel(android.os.Parcel, int)' on a null object reference
W/NotificationBackend:   at android.os.Parcel.createExceptionOrNull
W/NotificationBackend:   at android.os.Parcel.createException
W/NotificationBackend:   at android.os.Parcel.readException
W/NotificationBackend:   at android.app.INotificationManager$StubProxy.getNotificationHistory
W/NotificationBackend:   at com.android.settings.notification.NotificationBackend.getNotificationHistory
...
W/NotificationBackend: Caused by: android.os.RemoteException: Remote stack trace:
W/NotificationBackend:   at android.app.NotificationHistory.writeNotificationToParcel
W/NotificationBackend:   at android.app.NotificationHistory.writeToParcel
W/NotificationBackend:   at android.app.INotificationManager$Stub.onTransact
W/NotificationBackend:   at android.os.Binder.execTransactInternal
W/NotificationBackend:   at android.os.Binder.execTransact
```

### 可以解釋一下嗎
先來了解設定 app 是怎麼讀取通知紀錄的。

系統的通知由 `NotificationManagerService` 負責管理，所以紀錄也是和它要，透過 `getNotificationHistory` 這個方法。不過 `NotificationManagerService` 跟設定 app 不是同一個 process，它們要怎麼互相溝通？答案是用 Android 的 Binder 機制。

Binder 機制如何運作不是本文重點，要知道的只有在透過 Binder 傳遞資料時只能傳送字串、整數之類的基本資料型態，如果是比較複雜的物件就得先 serialize 成 Parcel 才能傳送，而通知紀錄就是一個例子。

Android 的通知紀錄用 `HistoricalNotification` 這個 class 儲存，其中有一個 `Icon` 型態的 `mIcon` 欄位負責儲存通知的圖示。

### 「價值十億美元的失誤」
看一下錯誤訊息會發現，系統在嘗試將一個 Icon 型態的值寫入 Parcel 時存取到了一個空的欄位，導致紀錄無法傳回設定 app。利用 Android 開源的好處，觀察一下相關的原始碼後會發現那個欄位就是 `mIcon`。

但 `mIcon` 應該是空的嗎？為什麼寫入時沒做檢查？再看一下會建立 `HistoricalNotification` 的相關原始碼，只有 `NotificationManagerService#maybeRecordInterruption` 跟從通知紀錄檔 deserialize 回 `HistoricalNotification` 的邏輯。

`maybeRecordInterruption` 就是系統儲存 app 通知的主要邏輯，在儲存時會特別檢查 icon 是不是空的。所以是 deserialize 的過程出了問題嗎？

<figure>
<img alt="Android Code Search, showing the readIcon method of the class NotificationHistoryProtoHelper, where the code path for Bitmap icon was left todo." src="{{ site.baseUrl }}/assets/disappeared-notification-history/proto-helper.png">
<figcaption>looks sus, <a href="https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/notification/NotificationHistoryProtoHelper.java;l=220;drc=master">source</a></figcaption>
</figure>

通知的 icon 可以有很多類型，可以是 app 內的 resource，或是 bitmap。觀察讀取和寫入 icon 的原始碼會發現，要是原本通知 icon 是 bitmap 類型的話，因為還沒實作儲存 bitmap 的邏輯，在經過一次寫入讀取的循環後，mIcon 就會變成空值。

### 結論
Google 拜託快點修好，這個問題已經從 11 拖到 12 了吧，儲存 bitmap 麻煩就算了，不能先檢查一下空值嗎
