---
title: Gradle peace in multi module projects
date: 2021-07-04 19:00:00 +0100
categories: [Gradle]
tags: [Gradle]
---

<img src="/assets/img/gradle_abstraction/gradle_sad_reality.jpg" class="center">


## Complications
---
Hello fellow Gradle user, you've probably copy pasted the configuration in every module you've had, no worries, today you'll forget copy pasting most of the code in the gradle configuration.

Gradle is slow we get that, but building multi modules helps that, since gradle rebuilds only the modules affected of changes.

One day you woke up and chose violence, copy pasting the same config over and over again in every `build.gradle` file you had for every new module you created.

You decided to create a new module (hope this post doesn't hurt your ego)

<img src="/assets/img/gradle_abstraction/1.png" class="center">

then another module
<img src="/assets/img/gradle_abstraction/2.png" class="center">

and you had to duplicate this in every gradle file, you hated your life because *Gradle* doesn't simplify this out of the box, but don't worry, we all hate *Gradle* because it's complicated for no reason and a build system shouldn't be, but hey, maybe *Bazel* will get more attention in the future.

## Solution
---

One day I woke up mad (it was a Tuesday, doing a code review to myself, hope i'm not the only one doing this to himself) and decided to live up to my motto, "a good programmer doesn't write the same code twice", in this case I didn't want to write the same Gradle config twice, so I chose to fight.

Then in my project `build.gradle` I chose *peace*

```groovy
subprojects {

    switch (it.name) {
        case "app":
            apply plugin: 'com.android.application'
            apply plugin: 'kotlin-android'
            apply plugin: 'kotlin-kapt'
            apply plugin: 'kotlin-parcelize'
            dependencies {
                implementation fileTree(dir: 'libs', include: ['*.jar'])
            }
            applyAndroid(it, true)
            break

        //setup gradle for Kotlin libs
        case ["enums", "regex"]:
            apply plugin: 'java-library'
            apply plugin: 'kotlin'
            applyKotlinModule(it)
            break

        default:
            //setup gradle for libraries
            apply plugin: 'com.android.library'
            apply plugin: 'kotlin-android'
            apply plugin: 'com.github.dcendents.android-maven'
            apply plugin: 'kotlin-parcelize'
            applyAndroid(it, false)

            dependencies {
                implementation fileTree(dir: 'libs', include: ['*.jar'])
                // Dependencies for local unit tests
                testImplementation "junit:junit:$junitVersion"
                testImplementation "org.hamcrest:hamcrest-all:$hamcrestVersion"
                testImplementation "androidx.test.ext:junit-ktx:$androidXTestExtKotlinRunnerVersion"
                testImplementation "androidx.test:core-ktx:$androidXTestCoreVersion"
                testImplementation "org.robolectric:robolectric:$robolectricVersion"
                testImplementation "androidx.arch.core:core-testing:$archTestingVersion"
                testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines"

                // AndroidX Test - Instrumented testing
                androidTestImplementation "androidx.test.ext:junit:$androidXTestExtKotlinRunnerVersion"
                androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"
            }

            break
    }

}
```

```groovy
def applyKotlinModule(project) {
    project.java {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    project.dependencies {
        implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    }
}
```


```groovy
def applyAndroid(project, buildConfigCase) {
    project.android {
        compileSdkVersion compileVersion

        defaultConfig {
            minSdkVersion minVersion
            targetSdkVersion compileVersion
            versionCode verCode
            versionName verName
            testInstrumentationRunner testRunner
        }

        compileOptions {
            sourceCompatibility = 1.8
            targetCompatibility = 1.8
        }

        kotlinOptions {
            jvmTarget = "1.8"
        }

        testOptions.unitTests {
            includeAndroidResources = true
        }

        buildFeatures {
            aidl = false
            renderScript = false
            resValues = false
            shaders = false
            buildConfig = buildConfigCase
        }
    }

}
```

What I did was check for every subproject's name and create the configuration for that, if it's the `:app` then there can only be one and everything else was a library, either a Kotlin or Android based one.

You can check this approach in my open source [project](https://github.com/FunkyMuse/KAHelpers/blob/master/build.gradle).


### Feature modules (feature folders)

One day you wanted to call a function and have the proper dependencies included?

```groovy
void applyComposeUIDeps(project) {
    project.dependencies {
        implementation "androidx.compose.ui:ui:$compose_version"
        implementation "androidx.compose.ui:ui-tooling:$compose_version"
    }
}
```

I gotchu
<img src="/assets/img/gradle_abstraction/3.png" class="center">

### Drawbacks

When it comes to feature folders you need to exempt them when you're using this approach, since you're basing this approach on names of the modules every name has to be unique, that's the drawback, sometimes you may end up with longer names.

### End result
<img src="/assets/img/gradle_abstraction/4.png" class="center">

some of your feature `build.gradle` modules will look really short

<img src="/assets/img/gradle_abstraction/5.png" class="center">


You can check this approach in my other open source [project](https://github.com/FunkyMuse/Aurora/blob/main/build.gradle).

You can go forward and do many more of these abstractions and forget about copy pasting gradle config once and for all.


## Goodbye

Don't forget to drink water, it's summer and it's hot.

Thanks for reading, stay cool and also cool.


<img src="/assets/img/gradle_abstraction/goodbye.jpg" class="center">
