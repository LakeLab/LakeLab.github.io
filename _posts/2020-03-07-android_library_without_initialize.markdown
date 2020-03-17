---
title: "Android library without initialize code likes firebase"
categories: Android
date: 2020-03-17 18:45:00 +0900
tags:
  - Android
  - Notifier
  - Library
---

If you have ever integrated firebase, you already know there is no initialize code. How can they get the context without initializing code? 

[The firebase blog](https://firebase.googleblog.com/2016/12/how-does-firebase-initialize-on-android.html) already posted about this topic. But this post is about a real use case with an android library, [Notifier](https://github.com/LakeLab/Notifier).
 
## Purpose
Initialize an android library with an Application Context by using Content Provider. 

## TLDR
* Make Content Provider & Inject Context
* Register your Content Provider in your library manifest
* Use the context, or Initialize your library from your Content provider

### Make Content Provider & Inject Context
You just need to extend the ContentProvider and implement abstract methods without any special implementations. (You can implement abstract methods for this content provider, If your library needs use contents provider)

```java
public class ContextInjections extends ContentProvider {

    @SuppressLint("StaticFieldLeak")
    private static Context global;

    public static Context getApplicationContext() {
        return global;
    }

    @Override
    public boolean onCreate() {
        global = getContext();
        return true;
    }
}
```
*The full code of ContextInjections class in Notifier is [here](https://github.com/LakeLab/Notifier/blob/master/notifier/src/main/java/com/lakelab/notifier/ContextInjections.java)*.

### Register your Content Provider in your library manifest
There can be only one ContentProvider on an Android device with a given "authority" string. So, if your library is used in more than one app on a device, you have to make sure that they get added with two different authority strings.
For that, You can use [${applicationId}](https://developer.android.com/studio/build/manifest-build-variables.html) for your authorities like this:  
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.lakelab.notifier">

    <application>
        <provider
            android:name=".ContextInjections"
            android:authorities="${applicationId}.ContextInjections"
            android:exported="false" />
    </application>

</manifest>
```

### Use the context, or Initialize your library from your Content provider

You can use context in your library as follow.
```java 
Context context = ContextInjections.getApplicationContext();
```

Or You can initialize your library in onCreate method on the Content provider, which you made.
```java
@Override
public boolean onCreate() {
    Somelibrary.init(getContext);
    return true;
}
```

## C.F
### Another way for the process that is not in the main process
For an application that must run in another process, You should either serve another way to initialize your library or make sure to avoid calling anything that requires the initialization and the context. Because Your ContentProvider onCreate won't never get invoked in another process if that process won't create any ContentProviders.

## References
- [How does Firebase initialize on Android?](https://firebase.googleblog.com/2016/12/how-does-firebase-initialize-on-android.html)
- [Notifier](https://github.com/LakeLab/Notifier)