---
layout: single
classes: wide
header:
  image: /assets/images/2020-03-19-network_debugging_with_stetho-0.png
categories: Android
title: Network debugging in android with Stetho (OkHttp, Retrofit, Glide) + Separating build variants
date: 2020-03-23 02:46:00 +0900
tags:
  - Android
  - Debug
---

## Stetho
[Stetho](http://facebook.github.io/stetho/) is a debug bridge for Android, created by Facebook. Helps you use your Chrome inspector to inspect your app's Network, DB, View Hierarchy, etc.
If you've ever used the Chrome inspector before, You know that tool is very useful for network debugging.  

These are Stetho's specifications.

![image_1](/assets/images/2020-03-19-network_debugging_with_stetho-1.png)

![image_2](/assets/images/2020-03-19-network_debugging_with_stetho-2.png)

![image_3](/assets/images/2020-03-19-network_debugging_with_stetho-3.png)


This post will be about integrating with Android libraries related to Networking (OkHttp - Retrofit, Glide). 

## TL;DR
* Integrating with Stetho
    * Add dependencies & proguard
    * Initialize Stetho and make Stetho injected OkHttpClient & Retrofit 
    * Make a Stetho injected glide module to replace a default glide module
    * **Using GlideApp**
    * Debugging on Chrome inspector
    
* Extra part 
    * Separate build variants (Build type) 
    
## Integrating with Stetho
### Add dependencies & proguard
Add dependencies for network debugging. Stetho, Glide, Retrofit.\
*C.F) DebugImplementation is for separating build variants. This will be explained in extra part.*
```gradle
dependencies {
    //For OkHttp3 & Stetho
    debugImplementation 'com.facebook.stetho:stetho-okhttp3:1.5.1'

    //For Retrofit
    Implementation 'com.squareup.retrofit2:retrofit:2.5.0' 

    //For Glide
    Implementation 'com.github.bumptech.glide:glide:4.11.0'
    debugImplementation "com.github.bumptech.glide:okhttp3-integration:4.11.0"

    // Depending on the language you write for AppGlideModule class, add different annotation processor as follows.
    // For kotlin
    kapt 'com.github.bumptech.glide:compiler:4.11.0'
    // For Java
    annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0'
}
```

If you are using [Proguard](https://developer.android.com/studio/build/shrink-code), Add some required rules for Glide & OkHttp & Retrofit on your proguard-rules.
```
# GLIDE
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.module.AppGlideModule
-keep public enum com.bumptech.glide.load.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}
# for DexGuard only
-keepresourcexmlelements manifest/application/meta-data@value=GlideModule


# Retrofit
# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions
# Retain generic type information for use by reflection by converters and adapters.
-keepclassmembernames,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
# Ignore annotation used for build tooling.
-dontwarn org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement

# Okhttp
-dontwarn okhttp3.**
-dontwarn okio.**
-dontwarn javax.annotation.**
-dontwarn org.conscrypt.**
# A resource is loaded with a relative path so the package of this class must be preserved.
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

```

### Initialize Stetho and make Stetho injected OkHttpClient & Retrofit 

Initialize Stetho with an Application context.\
*C.F) Usually, Stetho is initialized at an Application class.*
```kotlin
 Stetho.initializeWithDefaults(context)
```

Make OkHttpClient which is injected Stetho.
```kotlin
val preparedOkHttpClient = OkHttpClient.Builder().addNetworkInterceptor(StethoInterceptor()).build()
```


### Make a Stetho injected glide module to replace a default glide module
For debugging a glide network, You need replace a default Glide module to a Stetho-injected Glide module.

To replace the default glide module, Create an AppGlideModule for connecting Glide and the Stetho-injected OkHttp, first.
```kotlin
@GlideModule
class OkHttp3GlideModule : AppGlideModule() {
    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
        registry.replace(
            GlideUrl::class.java,
            InputStream::class.java,
            OkHttpUrlLoader.Factory(BaseNetworkTools.preparedOkHttpClient)
        )
    }
}
```

After finishing making GlideModule, Register meta-data at Android manifest in the application tags. 
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.lakelab.android.network">
    <application>
        <meta-data
            android:name="com.bumptech.glide.integration.okhttp3.OkHttpGlideModule"
            tools:node="remove" />
    </application>
</manifest>
```
*C.F) This is for avoiding conflict between your glide module and the default glide module.
 If you want to know the detail about this, You can explore [here](https://github.com/bumptech/glide/wiki/Configuration#conflicting-glidemodules).*

### Using GlideApp
**Using GlideApp instead of normal Glide.**
```kotlin
GlideApp.with(this@MainActivity)
        .load(SAMPLE_IMAGE_ICON)
        .diskCacheStrategy(DiskCacheStrategy.NONE)
        .skipMemoryCache(true).into(am_sample_image)
```
This is because limiting the generated API to applications allows us to have a single implementation of the API in Glide. For more details about this, Please visit [here](http://bumptech.github.io/glide/doc/generatedapi.html#availability).

### Debugging on Chrome inspector
Open Chrome and Browse `chrome://inspect/#devices` for enjoying easy debugging :)

![image_1](/assets/images/2020-03-19-network_debugging_with_stetho-1.png)


## Extra part
It should be a disaster if your application can be debugged on a release build. So They usually use it this way.

```kotlin
if (BuildConfig.DEBUG) {
     // initialize
}
```
But this way can make your codes complex and easy to make confused. So For clear and mistake-proof code, Separating builds variants (Build type) will be introduced in this extra part. 

### Separate build variants (Build type) 

It's already separated by Debug/Release when you make a project. So Only the Separating folder is necessary for separating build variants.
This is my sample project structure for separating build variants.

![image_5](/assets/images/2020-03-19-network_debugging_with_stetho-4.png)

Make two directories debug/release like above and Make classes of the same name in each directory as follows.

```kotlin
// StethoUtils in debug directory
class StethoUtils {
    companion object {
        @JvmStatic
        internal fun initStetho(context: Context) = Stetho.initializeWithDefaults(context)

        @JvmStatic
        internal fun makeBuilderWithStetho() =
                OkHttpClient.Builder().addNetworkInterceptor(StethoInterceptor())
    }
}

// OkHttp3GlideModule in debug directory
@GlideModule
class OkHttp3GlideModule : AppGlideModule() {
    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
        registry.replace(
            GlideUrl::class.java,
            InputStream::class.java,
            OkHttpUrlLoader.Factory(BaseNetworkTools.preparedOkHttpClient)
        )
    }
}
```
```kotlin
// StethoUtils in release directory
class StethoUtils {
    companion object {
        @JvmStatic
        internal fun initStetho(context: Context) {
        }

        @JvmStatic
        internal fun makeBuilderWithStetho() =
            OkHttpClient.Builder()
    }
}

// OkHttp3GlideModule in release directory
@GlideModule
class OkHttp3GlideModule : AppGlideModule() {
    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
    }
}
```

Your manifest even can be separated by the debug variant as follows. It will be merged automatically with your manifest in the main directory.
```xml
<!--AndroidManifest.xml in debug directory-->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.lakelab.android.network">

    <application>
        <meta-data
            android:name="com.bumptech.glide.integration.okhttp3.OkHttpGlideModule"
            tools:node="remove" />
    </application>
</manifest>
```

C.F) You can change your build variant in the menu in Android Studio as follows.

![image_6](/assets/images/2020-03-19-network_debugging_with_stetho-4.png)


Now, Your code is completely separated by debugging and release build variables, And It makes you safe from mistakenly leaking your network information.

## References
- [Stetho official website](http://facebook.github.io/stetho/)
- [Glide4 and Stetho to easily debug your image loading system](https://proandroiddev.com/glide4-and-stetho-to-easily-debug-your-image-loading-system-c274d0d9966b)

## If you'd like to see the full codes for this post
Please feel free to visit [here](https://github.com/LakeLab/android-network).

