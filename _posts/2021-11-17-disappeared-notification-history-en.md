---
layout: post
date: 2021-11-17 02:53:00 +0800
title: "The disappeared notification history"
tags: android notification
---
This article is also available in: [繁體中文](./disappeared-notification-history.html)

Android 11 added a notification history screen. You can open it directly from the notification shade. Different from the old Notification Log that can be accessed by creating a settings shortcut widget, it features a new, smoother interface. It contains two sections, one shows notifications you dismissed recently, and the other shows remaining notifications shown in the last 24 hours.

<figure>
<img src="{{site.baseUrl}}/assets/disappeared-notification-history/notification-history.png">
<figcaption>The new notification history screen (though it’s Android 12’s)</figcaption>
</figure>

<figure style="display: flex;flex-direction: column;">
<video style="max-height: 600px;" src="{{site.baseUrl}}/assets/disappeared-notification-history/adding-notification-log-widget.mp4" controls></video>
<figcaption>Steps for creating a widget for accessing the old notification log</figcaption>
</figure>

It has a problem, though, with the last 24 hours section. Sometimes the section won’t show up at all. Very annoying.

So I started investigating and found that when the section didn’t show up, an error would appear in the logcat:
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

### NoMan what?
Let’s start with how the Settings app reads the notification history.

Notifications in Android are managed by `NotificationManagerService`. By calling its `getNotificationHistory` method you can access the history. And because `NotificationManagerService` and the Settings app are different process, they communicate with each other through the Binder mechanism.

The point is Binder only allows transmitting primitive data types like integers and strings. To send an object, you would need to serialize it into a Parcel. And this is how the notification history will be sent.

Notification history is stored in `HistoricalNotification` objects, which contain an `Icon` type field called `mIcon` that stores the icon of the particular notification.

### The billion dollar mistake
By reading the error log you can find that the system tried to access a null field when writing an `Icon` object into a Parcel, and this is why the call failed. Looking into [related code paths](https://cs.android.com/android/platform/superproject/+/53022318db4a69095cdcc6d4b83bc26ecb12e835:frameworks/base/core/java/android/app/NotificationHistory.java;l=500) it seems like `mIcon` is what the system tried to access.

But why is there no null checks? Should `mIcon` even be null? After more reads of source code, it seems that a `HistoricalNotification` object would only be constructed by `NotificationManagerService#maybeRecordInterruption` and the deserialization logic of `HistoricalNotification`.

`maybeRecordInterruption` is exactly where a notification gets stored in the history, which checks for a null icon. It must be in the deserialization logic then?

<figure>
<img alt="Android Code Search, showing the readIcon method of the class NotificationHistoryProtoHelper, where the code path for a bitmap icon was left to-do." src="{{ site.baseUrl }}/assets/disappeared-notification-history/proto-helper.png">
<figcaption>looks sus, <a href="https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/notification/NotificationHistoryProtoHelper.java;l=220;drc=master">source</a></figcaption>
</figure>

There are many sources for an icon. It could be from the resource of a package, or from a bitmap. By reading the code you may notice the reading and writing logic for an icon of the bitmap type is not implemented yet. After persisting such notification and reading it back, `mIcon` would become null.

### Conclusion?
This problem still exists in Android 12. Someone, please, add a null check or something.
