---
title: How long will you go to protect your Android app from being tampered?
date: 2021-07-10 22:20:00 +0100
categories: [Android, Security]
tags: [Android, Security]
---

<img src="/assets/img/10/1.jpg" class="center">


## The big problem
---

Security of your app matters, whether you're an indie developer trying to place a product and make a living out of your application, a big bank, a giant dating app or just an enthusiast who likes building Android apps.

We're stuck in apocaliptic world of `.apk` files that can easily be decompiled and it doesn't require a lot of knowledge to tamper with the existing code.

An `APK` file contains all of a program's code (such as `.dex` a.k.a classes files), `libs`, `resources`, `assets`, `certificates`, and `manifest file`, also that `APK` file is vulnerable.

If you noticed how cringey the meme is, that's how cringey it is to reverse-engineer an Android `.apk`, it's cringely funny and easy.

[dex2jar](https://github.com/pxb1988/dex2jar) converts `.apk` to `.jar`, using [jd-gui](https://java-decompiler.github.io/) you can even look at the `Java/Kotlin` code (most of the time at least) and if you want to extract the resources (you probably need images, translations etc...) [apktool](https://ibotpeaches.github.io/Apktool/) is here to help you.

You don't even need to use the aforementioned tools, there are other websites that had this implemented and will do the heavy burden of writing scripts or doing the job yourself, just drop them an `.apk` and within minutes they're done.


## Finding or just salvaging a solution, is there a silver bullet?
---

You might think, of course there's something that has to be done in order to protect our applications, matter of fact there is, but you can just mitigate the problem untill someone outsmarts you.

**Is there a silver bullet?**

*NO!*


### R8/Proguard
---

You might have heard of proguard, it will obfuscate the code by giving random meaningless names to all method, classes and variables, you might see something like `a` `b` classes or mix up of chars up to several letters that you can barely even understand, this also does optimize the code.


### Dexguard
---

ProGuard is a generic optimizer for Java bytecode.

DexGuard is a specialized tool for the protection of Android applications.

At least that's what they claim, that is a paid option and comes at a hefty price, all in all these two solutions do not entirely protect your app from being tampered.

Keep in mind that these tools alone do not solve your problem, they just slow it down.


### Kotlin messing up things
---
We all love Kotlin alright, it enabled us to make apps that Java never did, especially writing non-confusing lambdas and all other handy features.

But that comes with a price.

One day you're enabling `R8` and `proguard`, minifying and obfuscating code and your class is dead simple

- The `Intrinsics` way and The **meta-data** way

You've been exposed by the Kotlin compiler.

```kotlin
class DeadSimpleClass {

    lateinit var leastImportantVariable: String

    private val lazyVariable by lazy { 
        "someString"
    }

    fun someFunction(someAbnormalString: String, aList: List<String>) {
        //
    }

    fun String.doSomething() {
        //
    }
}
```

when we decompile this class to `Java` bytecode we get

```java
import java.util.List;
import kotlin.Lazy;
import kotlin.LazyKt;
import kotlin.Metadata;
import kotlin.jvm.functions.Function0;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;

@Metadata(
   mv = {1, 4, 3},
   bv = {1, 0, 3},
   k = 1,
   d1 = {"\u0000$\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000e\n\u0002\b\t\n\u0002\u0010\u0002\n\u0002\b\u0002\n\u0002\u0010 \n\u0002\b\u0002\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002J\u001c\u0010\r\u001a\u00020\u000e2\u0006\u0010\u000f\u001a\u00020\u00042\f\u0010\u0010\u001a\b\u0012\u0004\u0012\u00020\u00040\u0011J\n\u0010\u0012\u001a\u00020\u000e*\u00020\u0004R\u001b\u0010\u0003\u001a\u00020\u00048BX\u0082\u0084\u0002¢\u0006\f\n\u0004\b\u0007\u0010\b\u001a\u0004\b\u0005\u0010\u0006R\u001a\u0010\t\u001a\u00020\u0004X\u0086.¢\u0006\u000e\n\u0000\u001a\u0004\b\n\u0010\u0006\"\u0004\b\u000b\u0010\f¨\u0006\u0013"},
   d2 = {"LDeadSimpleClass;", "", "()V", "lazyVariable", "", "getLazyVariable", "()Ljava/lang/String;", "lazyVariable$delegate", "Lkotlin/Lazy;", "leastImportantVariable", "getLeastImportantVariable", "setLeastImportantVariable", "(Ljava/lang/String;)V", "someFunction", "", "someAbnormalString", "aList", "", "doSomething", "untitled"}
)
public final class DeadSimpleClass {
   public String leastImportantVariable;
   private final Lazy lazyVariable$delegate;

   @NotNull
   public final String getLeastImportantVariable() {
      String var10000 = this.leastImportantVariable;
      if (var10000 == null) {
         Intrinsics.throwUninitializedPropertyAccessException("leastImportantVariable");
      }

      return var10000;
   }

   public final void setLeastImportantVariable(@NotNull String var1) {
      Intrinsics.checkNotNullParameter(var1, "<set-?>");
      this.leastImportantVariable = var1;
   }

   private final String getLazyVariable() {
      Lazy var1 = this.lazyVariable$delegate;
      Object var3 = null;
      boolean var4 = false;
      return (String)var1.getValue();
   }

   public final void someFunction(@NotNull String someAbnormalString, @NotNull List aList) {
      Intrinsics.checkNotNullParameter(someAbnormalString, "someAbnormalString");
      Intrinsics.checkNotNullParameter(aList, "aList");
   }

   public final void doSomething(@NotNull String $this$doSomething) {
      Intrinsics.checkNotNullParameter($this$doSomething, "$this$doSomething");
   }

   public DeadSimpleClass() {
      this.lazyVariable$delegate = LazyKt.lazy((Function0)null.INSTANCE);
   }
}
```
Notice something?

The variables and method names are still present!
These names could expose our business logic to an outsider and this information may be used in a harmful way. 

It’s not a major security issue, but still an issue.

We can just **NOT** fix the `Instrinsics` way by adding a proguard rules

`-assumenosideeffects`: ProGuard removes calls to such methods, if it can determine that the return values aren’t used. 

```
-assumenosideeffects class kotlin.jvm.internal.Intrinsics { *; }
```

*DO NOT ADD THIS TO THE RULES!*

Remove all Intrinsics calls will lead to errors, imagine we remove `equals` call for data classes and wonder why our diff utils aren't working for example.


**Only applicable when optimizing.**

- `Data class` way

```kotlin
data class DeadSimpleClass(val funky:String, val muse:String, val someOtherString:String)
```

```java
import kotlin.Metadata;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

@Metadata(
   mv = {1, 4, 3},
   bv = {1, 0, 3},
   k = 1,
   d1 = {"\u0000\"\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0000\n\u0002\u0010\u000e\n\u0002\b\f\n\u0002\u0010\u000b\n\u0002\b\u0002\n\u0002\u0010\b\n\u0002\b\u0002\b\u0086\b\u0018\u00002\u00020\u0001B\u001d\u0012\u0006\u0010\u0002\u001a\u00020\u0003\u0012\u0006\u0010\u0004\u001a\u00020\u0003\u0012\u0006\u0010\u0005\u001a\u00020\u0003¢\u0006\u0002\u0010\u0006J\t\u0010\u000b\u001a\u00020\u0003HÆ\u0003J\t\u0010\f\u001a\u00020\u0003HÆ\u0003J\t\u0010\r\u001a\u00020\u0003HÆ\u0003J'\u0010\u000e\u001a\u00020\u00002\b\b\u0002\u0010\u0002\u001a\u00020\u00032\b\b\u0002\u0010\u0004\u001a\u00020\u00032\b\b\u0002\u0010\u0005\u001a\u00020\u0003HÆ\u0001J\u0013\u0010\u000f\u001a\u00020\u00102\b\u0010\u0011\u001a\u0004\u0018\u00010\u0001HÖ\u0003J\t\u0010\u0012\u001a\u00020\u0013HÖ\u0001J\t\u0010\u0014\u001a\u00020\u0003HÖ\u0001R\u0011\u0010\u0002\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\u0007\u0010\bR\u0011\u0010\u0004\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\t\u0010\bR\u0011\u0010\u0005\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\n\u0010\b¨\u0006\u0015"},
   d2 = {"LDeadSimpleClass;", "", "funky", "", "muse", "someOtherString", "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V", "getFunky", "()Ljava/lang/String;", "getMuse", "getSomeOtherString", "component1", "component2", "component3", "copy", "equals", "", "other", "hashCode", "", "toString", "untitled"}
)
public final class DeadSimpleClass {
   @NotNull
   private final String funky;
   @NotNull
   private final String muse;
   @NotNull
   private final String someOtherString;

   @NotNull
   public final String getFunky() {
      return this.funky;
   }

   @NotNull
   public final String getMuse() {
      return this.muse;
   }

   @NotNull
   public final String getSomeOtherString() {
      return this.someOtherString;
   }

   public DeadSimpleClass(@NotNull String funky, @NotNull String muse, @NotNull String someOtherString) {
      Intrinsics.checkNotNullParameter(funky, "funky");
      Intrinsics.checkNotNullParameter(muse, "muse");
      Intrinsics.checkNotNullParameter(someOtherString, "someOtherString");
      super();
      this.funky = funky;
      this.muse = muse;
      this.someOtherString = someOtherString;
   }

   @NotNull
   public final String component1() {
      return this.funky;
   }

   @NotNull
   public final String component2() {
      return this.muse;
   }

   @NotNull
   public final String component3() {
      return this.someOtherString;
   }

   @NotNull
   public final DeadSimpleClass copy(@NotNull String funky, @NotNull String muse, @NotNull String someOtherString) {
      Intrinsics.checkNotNullParameter(funky, "funky");
      Intrinsics.checkNotNullParameter(muse, "muse");
      Intrinsics.checkNotNullParameter(someOtherString, "someOtherString");
      return new DeadSimpleClass(funky, muse, someOtherString);
   }

   // $FF: synthetic method
   public static DeadSimpleClass copy$default(DeadSimpleClass var0, String var1, String var2, String var3, int var4, Object var5) {
      if ((var4 & 1) != 0) {
         var1 = var0.funky;
      }

      if ((var4 & 2) != 0) {
         var2 = var0.muse;
      }

      if ((var4 & 4) != 0) {
         var3 = var0.someOtherString;
      }

      return var0.copy(var1, var2, var3);
   }

   @NotNull
   public String toString() {
      return "DeadSimpleClass(funky=" + this.funky + ", muse=" + this.muse + ", someOtherString=" + this.someOtherString + ")";
   }

   public int hashCode() {
      String var10000 = this.funky;
      int var1 = (var10000 != null ? var10000.hashCode() : 0) * 31;
      String var10001 = this.muse;
      var1 = (var1 + (var10001 != null ? var10001.hashCode() : 0)) * 31;
      var10001 = this.someOtherString;
      return var1 + (var10001 != null ? var10001.hashCode() : 0);
   }

   public boolean equals(@Nullable Object var1) {
      if (this != var1) {
         if (var1 instanceof DeadSimpleClass) {
            DeadSimpleClass var2 = (DeadSimpleClass)var1;
            if (Intrinsics.areEqual(this.funky, var2.funky) && Intrinsics.areEqual(this.muse, var2.muse) && Intrinsics.areEqual(this.someOtherString, var2.someOtherString)) {
               return true;
            }
         }

         return false;
      } else {
         return true;
      }
   }
}
```

As you see, we still have the metadata and now we have the  `public String toString()` exposing us as well with the variable names and also the copy method.

We can't beat `String`s.

**DO NOT PUT** sensitive info in shared preferences, they're just basically `XML` that can be easily read, use [Encrypted shared preferences](https://developer.android.com/topic/security/data#edit-shared-preferences).

Also do not use unencrypted SQL database like Room, use a [cipher](https://github.com/sqlcipher/android-database-sqlcipher) to make your database tables unreadable or some other way of encryption.

### We can't beat `String`s
---

Every app connects to a service for some way of data source, it has to use an url/ip to fetch it from the source.

What seems to be the problem here?
We're exposing the `URL` to everyone out there, since everyone can decompile the app, everyone can catch the url and has a clear idea where the source is coming from.

There are ways to make this pain go away by security through obscurity but you can't really fix this issue, some of the noble things you may have tried is
1. Adding the url to `build.gradle` thinking it's hidden, nope it's not, the calling site where you provide the `url` to the function for ex. `Retrofit.Builder().setBaseUrl(BuildConfing.BASE_URL)` is gonna reveal your beloved and hidden `String`.
2. Scattering the URL as bytes and assembling it in one or from more places, again security through obscurity.
3. Reading the "secure" and hidden string from `.so` *C/C++* library and also in combination with #2, `IDA pro` or `Ghidra` can just print all the strings from the `.so`, so you're not really secure.
   
The best way to beat this is to use `https` for your communication with some token `auth` for your endpoints.

Another helpful things are `Certificate` Pinning, `Public Key` Pinning & `Hash` Pinning.

- `Certificate` Pinning - We store the certificate in our application and when the certificate expires, we would update our application with the new certificate. 
   At runtime, we retrieve the server’s certificate in the callback, inside the callback, we compare the newly received certificate with the certificate shipped within our app, if it matches, we can trust the host else we will throw a SSL certificate error.

**The downside to pinning a certificate**
Each time our server rotates a certificate, we need to update our application, imagine this happening frequently.

- `Public Key` Pinning - We generate a keypair, we put the private key in our server and the public key inside our app. 
   We check the extracted public key with its copy of the public key, if it matches like in the certificate pinning, we can trust the host else we will throw a SSL certificate error. 
   By using public key pinning, we can avoid frequent application updates as the public key can remain same for longer periods.

**The downside to Public Key pinning**
1. Extracting the key is a process on it's own 
2. The key is *static* and may not align or even violate with the key rotation policies.

- `Hash Pinning` - we pin the hash of the `Public Key` of our server’s certificate and match it with the hash of the certificate’s public key received during a *network request*. When we receive a certificate can hash it with a secure hashing algorithm (`SHA-256` > `Base64` for better readability) to provide anonymity to a certificate or public key.

The problem still exists, the strings need to be provided at the calling site for your network service provider, that makes them visible, doesn't matter if you use `Base64` or shifting bytes, determined person can uncover them, just how far they're willing to go too?

But there are other problems that exist besides this one, these affect your app as well, every layer of security matters.

### App modifiers

As you might or might not be aware, `rooting` is a process of getting super-user privileges on `Android` since it runs on a `Linux` kernel, with that in mind you can use apps like `Magisk`, `Xposed` or the infamous `Lucky patcher` to modify or hook into other apps, but with root you can even modify system settings and even tamper the kernel.

`Android` allows you to see which apps are installed on your device, if you detect that `Magisk` is installed you can just crash the app, you can try to run a `sudo` command as well and if it happens then you can crash the app to disallow `rooted` phones to use your app.

The problem arises when `Magisk` hides itself, **YES**, it can hide it's name, although that's also detectable.

Android 11 "came with a way to improve security" and disables a way for you to read every installed application on your phone, unless you

**Query specific packages - you must mention in the manifest which ones you'll query**
```xml
<manifest package="com.funkymuse.app">
    <queries>
        <package android:name="com.funkymuse.vigilante" />
        <package android:name="com.funkymuse.aurora" />
    </queries>
</manifest>
```

**Use a permission**

`<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"/>`

**Query using intent filter**
```xml
<manifest package="com.funkymuse.app">
    <queries>
        <intent>
            <action android:name="android.intent.action.SEND" />
            <data android:mimeType="image/png" />
        </intent>
    </queries>
    ...
</manifest>
```

and as you know each app must have a main intent action, we can safely abuse **Query using intent filter** and use

```xml
 <queries>
        <intent>
            <action android:name="android.intent.action.MAIN" />
        </intent>
    </queries>
```

## Security checks that can matter to you
---

### Inspecting `Throwable` errors

The best way to inspect if there's something modified is by throwing an error and catching the trace then simply combing through the trace to check for bad guys.

### Detect third-party developer's kernel

From a security standpoint `Release-Keys` generally means the kernel is more secure, which is not always the case.

`Test-Keys` means it was signed with a custom key generated by a third-party developer.

Usually custom roms have `test-keys` if you want to strictly speaking disallow custom roms then you can detect it by 
```kotlin
fun isTestKeyBuild(): Boolean {
    val keys = Build.TAGS
    return keys != null && keys.contains("test-keys")
}
```

### Detect debugger

```kotlin
fun detectDebugger(): Boolean = Debug.isDebuggerConnected()

fun Context.isDebuggable(): Boolean = applicationContext.applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE != 0
```

`Debug.threadCpuTimeNanos` indicates the amount of time that the current thread has been executing code

```kotlin
fun isDebuggerAttached(): Boolean {
    val start = Debug.threadCpuTimeNanos()
    for (i in 0..999999) continue
    val stop = Debug.threadCpuTimeNanos()
    return stop - start >= 10000000
}
```

You can also check the `Settings.Global.WAIT_FOR_DEBUGGER` if it returns 1 then it will wait for debugger because it's been enabled in the dev-options inside the Settings.

### CRC check

You can use a file that you ship in the strings or from storage and do a `CRC` check to see if it's modified.

For example you can calculate `CRC` on `.dex` files based on some layout resource, string etc... or coming from a file that you receive from a server and store offline like a time bomb.


### Time bomb

This one can be done different ways, sometimes some modifiers can set custom time to go back and do time based manipulations on a time verification based auth

I would suggest using the Calendar class and having your application checking the current date against your expiration date in your `onResume` places.

### Zygote tampering

Zygote is called only once when your app is forked on launch, which means that if it's called more than once your app is modified.

```kotlin
/**
 * If the count is more than 2 then the app is modified
 * @return Int
 */
fun zygoteCallCount(): Int {
    var zygoteInitCallCount = 0
    try {
        throw Exception("PiracyChecker")
    } catch (e: Exception) {
        for (stackTraceElement in e.stackTrace) {
            if (stackTraceElement.className == "com.android.internal.os.ZygoteInit") {
                zygoteInitCallCount++
            }
            if (stackTraceElement.className == "com.saurik.substrate.MS$2" && stackTraceElement.methodName == "invoked") {
                zygoteInitCallCount++
            }
            if (stackTraceElement.className == "de.robv.android.xposed.XposedBridge" && stackTraceElement.methodName == "main") {
                zygoteInitCallCount++
            }
            if (stackTraceElement.className == "de.robv.android.xposed.XposedBridge" && stackTraceElement.methodName == "handleHookedMethod") {
                zygoteInitCallCount++
            }
        }
    }
    return zygoteInitCallCount
}
```
### Is device an emulator

Emulators are flexible, you can disable your app to work on emulators, this is a good idea especially for banking apps.

```kotlin
fun isEmulator(): Boolean {
    return (Build.BRAND.startsWith("generic") && Build.DEVICE.startsWith("generic")
            || Build.FINGERPRINT.startsWith("generic")
            || Build.FINGERPRINT.startsWith("unknown")
            || Build.HARDWARE.contains("goldfish")
            || Build.HARDWARE.contains("ranchu")
            || Build.MODEL.contains("google_sdk")
            || Build.MODEL.contains("Emulator")
            || Build.MODEL.contains("Android SDK built for x86")
            || Build.MANUFACTURER.contains("Genymotion")
            || Build.PRODUCT.contains("sdk_google")
            || Build.PRODUCT.contains("google_sdk")
            || Build.PRODUCT.contains("sdk")
            || Build.PRODUCT.contains("sdk_x86")
            || Build.PRODUCT.contains("vbox86p")
            || Build.PRODUCT.contains("emulator")
            || Build.PRODUCT.contains("simulator")) || isAlsoEmulator()
}
```

### Check if Developer option is enabled and/or check if USB debugging is enabled

Sometimes some of the requirements need of you to check that as well, especially some banking clients i've worked for had us perform this check

```kotlin
fun Context.isDevModeEnabled(): Boolean = Settings.Secure.getInt(contentResolver, Settings.Global.DEVELOPMENT_SETTINGS_ENABLED, 0) != 0
```

```kotlin
fun Context.isADBEnabled() : Boolean = Settings.Secure.getInt(contentResolver, Settings.Secure.ADB_ENABLED, 0) == 1
```
Link to some of the [security functions](https://github.com/FunkyMuse/KAHelpers/blob/master/security/src/main/java/com/crazylegend/security/PiracyCheckers.kt)

### Inspect SHA-XXX/MD5 signature (Verify your app's signing certificates (signatures))

The app signatures will be broken if the .apk is altered in any way — unsigned apps cannot typically be installed

A server side validation can be additionally added for inspecting the signing signature, this is a great idea to check it every time the app is launched doesn't matter if it's done without a server or additionally as a server side validation.

### App ID check

If the app ID doesn't match the one you've set, crash the app, this is a smart idea to have it done also via server side and have that call tied somewhere where the crucial functionality is a must.

Do note that if you try to access these identifiers in an app that targets Android 10 (API level 29) or higher, a SecurityException occurs.

###  Network security configuration

This one is awesomely explained by [Google](https://developer.android.com/training/articles/security-config).

### Unlocked bootloaders and SafetyNet attestations

Another great example explained by [Google](https://developer.android.com/training/safetynet/attestation).

### Server side country validation

If you run a server to validate things, you can check if your user changed the country on the device, do note that this has also be checked if the user is using `VPN` as an internet connection.

### Server side device ID validation

If you have the Device ID of the user, you can always check if with the one the user registered with and/or you've later on stored on the server, this is also nice since the ID is unique to a device, like IMEI.

*Do note that this requires a permission*

### Verify who's the installer of your app

If you haven't distributed your application via any other store than Play store, then you have to check who's the one that installed your app, if that doesn't match with the package id, crash it.

### Verify Google Play Licensing (LVL)

Google Play offers a licensing service that lets you enforce licensing policies for applications that you publish on Google Play. With Google Play Licensing, your application can query Google Play to obtain the licensing status for the current user.

Check out the [Google Developers page](https://developer.android.com/google/play/licensing/index.html).

### 

You can always test your app's security with [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF).

