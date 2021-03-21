---
title: Optimize the build speed for your Android project
date: 2020-07-07 20:55:00 +0100
categories: [Gradle, Android]
tags: [Android, Gradle, Optimizations]
---

We've all came to a decision to go and do something else while waiting for the build.gradle to finish, something like: people get married or divorced, learn to fly a plane, build a rocket ship out of LEGO or take one hour poo.

You downloaded Android studio and you just created an empty project and you had it run for the first time right?

Well this should look familiar to you if you profile your build with

```groovy
./gradlew --profile --offline --rerun-tasks assembleDebug
```


24 seconds, that's not good.



Alright here we go configuring gradle.


Our first step will be configuring on demand, what does this mean?
>Configuration on demand mode attempts to configure only projects that are relevant for requested tasks, i.e. it only executes the build.gradle file of projects that are participating in the build. This way, the configuration time of a large multi-project build can be reduced. In the long term, this mode will become the default mode, possibly the only mode for Gradle build execution. 

Inside your gradle.properties
```groovy
org.gradle.configureondemand=true
```
In case you get some weird error that a task failed mainly using Kotlin DSL scripts or groovy, disable configureOnDemand, you can look at why, at this issue.

It's good to increase the Java heap memory, of course if your RAM allows it, better not develop apps with <16GB RAM
```groovy
org.gradle.jvmargs=-Xmx1536m -XX:+UseParallelGC
```
and we're enabling parallel garbage collection.

You're building Android projects using Kotlin right, well lemme reveal a dirty secret, if you've used KAPT, it's worker is not enabled, yep, to make it so add

```groovy
kapt.use.worker.api = true
```
Get ready for more gradle.properties

Building a multi-module project?

```groovy
org.gradle.parallel=true
```
Get leverage of caching mecnanism? Say no more, in newer versions of Gradle I it's enabled by default, so there goes another valuable lesson, ALWAYS KEEP your GRADLE up to date with the LATEST VERSION, if you're unaffected by these caps, just use:

```groovy
org.gradle.caching=true
```
Using room?

```groovy
room.incremental = true
```
But wait, that's not all if you're using Room, inside your build.gradle find your

```groovy
defaultConfig {

              
}        
```
and add
```groovy

                javaCompileOptions {
           
                annotationProcessorOptions {
               
                arguments = ["room.incremental":"true"]
           
                }
        }

```
Get ready for one more gradle.properties addition, ladies and gentleman, the new citizen, configuration cache and it's brother file watcher

```groovy

org.gradle.unsafe.watch-fs=true

org.gradle.unsafe.configuration-cache=true 
```

If you decide not to use webP images and stick to PNGs
```groovy
                buildTypes {
       
                release {
           
                // Disables PNG crunching for the release build type or maybe
                debug too?
           
                crunchPngs false
       
                }
              }  
```
Never in your life write dependency using the + format, it adds to the build time if you add just one dependency like this and imagine if you add more, your build won't even finish by the time you go to sleep, so avoid something like this

```groovy
implementation 'androidx.appcompat:appcompat:1.+'
```

Many of us use Crashlytics or some form of crash reporting tool, I built my own called Crashy to avoid the slowdowns, leveraging AndroidX Startup and over-configuration of Crashyltics, how can you configure Crashlytics?
Well, no worries, there you go.
```groovy
buildTypes {

  debug {
     manifestPlaceholders = [crashlytics: "false"]
     crunchPngs false //you can read about that a little bit above

     firebaseCrashlytics {
       mappingFileUploadEnabled false
     }
   }

   release {
      manifestPlaceholders = [crashlytics: "true"]
   }
}
``` 


Alright, let's go step by step:
```groovy
manifestPlaceholders = [crashlytics: "false", analytics: "false"] 
```
this just disables the crashyltics and analytics from the manifest, but how do you do that?
```xml
<meta-data
    android:name="firebase_crashlytics_collection_enabled"
    android:value="${crashlytics}" />
```
```xml
 <meta-data
    android:name="firebase_analytics_collection_deactivated"
    android:value="${analytics}"/>
```
Wherever your Crashyltics initialization is, just use
```kotlin
FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(!Buildconfig.DEBUG)
```
we do the same for the analytics
```kotlin 
FirebaseAnalytics.getInstance().setAnalyticsCollectionEnabled(!BuildConfig.DEBUG)
```

`mappingFileUploadEnabled`

If you don't need crash reporting for your debug build,you can speed up your build by disabling mapping file uploading.


And now let's see


We have 4secs build time, including Dagger's kapt, Nav's kapt etc..

