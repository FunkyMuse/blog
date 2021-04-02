---
title: Dagger2 is hard, but it can be easy, part 6
date: 2021-04-02 19:05:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

In the [previous](/posts/dagger-part-5/) post we've encountered subcomponents and their hierarchy, custom scopes and we did not waste any time.

In this post we'll learn when's the right place to inject, named and qualifiers.

## Application's correct place for injection
---
We've seen in the [previous](/posts/dagger-part-5/) post when's the right place to inject the `@Singleton` scope, as a reminder it should be  right before the super.onCreate in your `: Application` inheritance
```kotlin
override fun onCreate() {
        applicationComponent = DaggerSingletonComponent.factory().create(this).also { it.provideGraphInside(this) }
        super.onCreate()
}
```

## Fragment's correct place for injection
---
Go ahead and create a Fragment subcomponent
```kotlin
@Subcomponent(modules = [FragmentModule::class])
@FragmentScoped
interface FragmentComponent {

    @Subcomponent.Factory
    interface Factory {
        fun create(): FragmentComponent
    }

    fun inject(homeFragment: HomeFragment)
}
```

an empty Fragment module if you want to use later on (optional)
```kotlin
@Module
object FragmentModule {


}
```
a scope for our fragment
```kotlin
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class FragmentScoped
```


and add 
```kotlin 
fun fragmentComponentFactory(): FragmentComponent.Factory
```
in our `SingletonComponent` right above/below your
```kotlin
fun activityComponentFactory(): ActivityComponent.Factory
```


a `HomeFragment` class for demonstration

```kotlin
class HomeFragment : Fragment(R.layout.home_fragment) {

    override fun onAttach(context: Context) {
        injector {
            fragmentComponentFactory().create().inject(this@HomeFragment)
        }
        super.onAttach(context)
    }
}
```
as you can see, the correct place to inject a *Fragment* is in `onAttach` since that's called just once when the Fragment is attached and as an analogy destroyed when the fragment is detached.

Our `injector` extension function changed a bit
```kotlin
inline fun FragmentActivity.injector(action: SingletonComponent.() -> Unit) {
    (application as DaggerIsEasyApplication).applicationComponent.action()
}

inline fun Fragment.injector(action: SingletonComponent.() -> Unit) {
    requireActivity().injector(action)
}
```
we changed from `AppcompatActivity.injector` to `FragmentActivity` so that we can
reuse it for our `Fragment.injector` which has a `requireActivity()` that's of a type 
`FragmentActivity`.

<img src="/assets/img/dagger/6/very_nice.jpg" class="center">


## Activity's correct place for injection
---
When it comes to an Activity we can go one way or the other, in the previous post we've bound the instance of `savedInstanceState` and `intent` just to see the different approaches now.

1. onCreate just before super.onCreate like in the `: Application`
as seen in the [previous](/posts/dagger-part-5/) post
```kotlin
 override fun onCreate(savedInstanceState: Bundle?) {
        injector {
            activityComponentFactory().create(savedInstanceState, intent).also { it.inject(this@MainActivity) }
        }
        super.onCreate(savedInstanceState)
 }
```
2. `addOnContextAvailableListener` when it comes to this method we can have something like
```kotlin
class MainActivity : AppCompatActivity() {

    init {
        addOnContextAvailableListener {
            injector {
                activityComponentFactory().create(savedInstanceState, intent).also { it.inject(this@MainActivity) }
            }
        }
    }
}
```
When's `addOnContextAvailableListener` called?

- `A` > If you do your injection inside `onCreate(savedInstanceState: Bundle?)` before `super.onCreate(savedInstanceState)` then this is called before `addOnContextAvailableListener` has a context available.

- `B` > If you do your injection inside `addOnContextAvailableListener` then this is done also before `super.onCreate(savedInstanceState)` but right after `A`

Our example is tied to providing instances of `savedInstanceState, intent` which shows us that it depends on your use case and how you created the whole logic.

Make sure you're having the latest appCompat dependency to get the context listener,
as of writing this blogpost, i'm using
```groovy
implementation "androidx.appcompat:appcompat:1.3.0-rc01"
```

To correct our sample if we're to use the injection in the context available listener we have to do the following
```kotlin
class MainActivity : AppCompatActivity() {

    init {
        addOnContextAvailableListener { availableContext ->
            injector {
                activityComponentFactory().create(availableContext).also { it.inject(this@MainActivity) }
            }
        }
    }
}
```

and correct our `ActivityComponent`'s create function to receive a `Context`

```kotlin
@Subcomponent(modules = [ActivityModule::class])
@ActivityScoped
interface ActivityComponent {

    @Subcomponent.Factory
    interface Factory {
        fun create(@BindsInstance context: Context): ActivityComponent
    }
    
    fun inject(mainActivity: MainActivity)
}
```

then you press F10 and build the application, it crashes.

```kotlin
error: [Dagger/DuplicateBindings] android.content.Context is bound multiple times
```

That means we already have a `Context` bound, when we built our `SingletonComponent`, one way to fix this is
```kotlin
@Component
@Singleton
interface SingletonComponent {
    @Component.Factory
    interface SingletonComponentFactory {
        fun create(@BindsInstance context: Application): SingletonComponent
    }
    fun activityComponentFactory() : ActivityComponent.Factory
    fun provideGraphInside(application: DaggerIsEasyApplication)
}
```
but having `@BindsInstance context: Application` means that we have to change every place where we have our application context to be of a type `Application`.

The other one is to use a Qualifier or Named, these are more elegant solutions.

For this demonstration go ahead and delete `Bundler` and `IntentHandler` classes and remove them from everywhere we've used them so far, we won't be needing them anymore.

## Named
---
When we use named, we have to use `@Named` which is an annotation that takes a string as an argument and you have to write the string yourself which can be prone to errors if you don't expose it as a constant, meaning that writing one string at two places can be prone to a human error (typo etc..).

To avoid our `Context` clash that we had, we have to use `@Named` to the instance
in our case `@Named(APPLICATION_CONTEXT_NAME)`

```kotlin
@Component
@Singleton
interface SingletonComponent {

    companion object {
        const val APPLICATION_CONTEXT_NAME = "applicationContext"
    }

    @Component.Factory
    interface SingletonComponentFactory {
        fun create(@BindsInstance @Named(APPLICATION_CONTEXT_NAME) context: Context): SingletonComponent
    }

    fun activityComponentFactory(): ActivityComponent.Factory
    fun fragmentComponentFactory(): FragmentComponent.Factory
    fun provideGraphInside(application: DaggerIsEasyApplication)
}
```

and in our `ActivityComponent` we have `@Named(ACTIVITY_CONTEXT_NAME)`
```kotlin
@Subcomponent(modules = [ActivityModule::class])
@ActivityScoped
interface ActivityComponent {

    companion object {
        const val ACTIVITY_CONTEXT_NAME = "activityContext"
    }

    @Subcomponent.Factory
    interface Factory {
        fun create(@BindsInstance @Named(ACTIVITY_CONTEXT_NAME) context: Context): ActivityComponent
    }

    fun inject(mainActivity: MainActivity)
}
```

now we've used an `Application`'s `Context` in `SharedPreferencesManager`

the change that has to be done as well

```kotlin
@Singleton
class SharedPreferencesManager @Inject constructor(@Named(SingletonComponent.APPLICATION_CONTEXT_NAME) context: Context) {
.
.
.
}
```

`@Named(SingletonComponent.APPLICATION_CONTEXT_NAME)` means that when Dagger comes into conflicts for the instances of the same type, it can provide them by a name we've *explicitly* instructed it to do.

## Qualifier
---
The qualifier approach is something *I personally* prefer to use, because it requires of you as a developer to create a separate `Qualifier` annotation and I can find it easily into the package where I use that module or class that's using the qualifier or (your use case or preference here).

To create a `@Qualifier` is really easy, just create a new annotation and add the
`@Qualifier` and `@Retention(AnnotationRetention.BINARY)`

Why `BINARY` you might ask yourself, that's because the annotation is stored in a binary form which is inaccessible for reflection.

Go ahead and create `ApplicationContext`
```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class ApplicationContext
```

and respectively `ActivityContext`
```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class ActivityContext
```

now our `SingletonComponent` looks cleaner.

```kotlin
@Component
@Singleton
interface SingletonComponent {

    @Component.Factory
    interface SingletonComponentFactory {
        fun create(@BindsInstance @ApplicationContext context: Context): SingletonComponent
    }

    fun activityComponentFactory(): ActivityComponent.Factory
    fun fragmentComponentFactory(): FragmentComponent.Factory
    fun provideGraphInside(application: DaggerIsEasyApplication)
}
```

Make the change inside the `ActivityComponent`

```kotlin
@Subcomponent(modules = [ActivityModule::class])
@ActivityScoped
interface ActivityComponent {

    @Subcomponent.Factory
    interface Factory {
        fun create(@BindsInstance @ActivityContext context: Context): ActivityComponent
    }

    fun inject(mainActivity: MainActivity)
}
```
now our `SharedPreferenceManager` looks cleaner too!

```kotlin
@Singleton
class SharedPreferencesManager @Inject constructor(@ApplicationContext context: Context) {
.
.
.
}
```
This I see as an absolute win.

## Broadcast receiver's correct place for injection
---
- If you have a receiver that is coming from the `AndroidManifest.xml`, you can't cast the `Context` to an application context since it's mostly of a type `ReceiverRestrictedContext`, you can however create a separate graph.

```kotlin
@Component
@BroadcastScoped
interface BroadcastComponent {

    @Component.Factory
    interface Factory {
        fun create(): BroadcastComponent
    }

    fun inject(bootReceiver: BootReceiver)
}
```
```kotlin
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class BroadcastScoped
```
```kotlin
class BootReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context?, intent: Intent?) {
        //do the injection here
        DaggerBroadcastComponent.create().inject(this)
        // you can have create accept the context and intent as @BindsInstance etc...

        intent ?: return
        context ?: return
        val action = intent.action ?: return

        if (action == Intent.ACTION_BOOT_COMPLETED || action == Intent.ACTION_LOCKED_BOOT_COMPLETED) {
            //some logic of your own
        }
    }
}
```
- Registering the receiver at runtime

For the sake of `A.` and `B.` create a class
```kotlin
class PowerLogger @Inject constructor(private val sharedPreferencesManager: SharedPreferencesManager) {

    fun logChargingStatus(status: Boolean) {
        sharedPreferencesManager.logChargingStatus(if (status) "IS CHARGING" else "CHARGING STOPPED")
    }

}
```

and inside the `sharedPreferenceManager` add a function
```kotlin
fun logChargingStatus(status: String) {
        Log.d(this::class.java.simpleName, status)
    }
```

and Subcomponent that'll inherit from `ApplicationComponent`
```kotlin
@Subcomponent
interface RuntimeBroadcastComponent
{
    @Subcomponent.Factory
    interface Factory {
        fun create(): RuntimeBroadcastComponent
    }

    fun inject(powerReceiver: PowerReceiver)
}
```

don't forget to add
```kotlin
    fun runtimeBroadcastComponentFactory() : RuntimeBroadcastComponent.Factory
```
inside the `ApplicationComponent` just as you did for the Fragment and Activity, this is only needed for `A.`

`A.` If you're using field injection and manually registering the receiver (which is painful to do so, we'll see simpler a bit below)

```kotlin
class PowerReceiver : BroadcastReceiver() {

    @Inject
    lateinit var powerLogger: PowerLogger

    override fun onReceive(context: Context?, intent: Intent?) {
        context ?: return
        intent ?: return
        (context.applicationContext as DaggerIsEasyApplication).applicationComponent.runtimeBroadcastComponentFactory().create().inject(this)

        val isCharging = when (intent.action) {
            Intent.ACTION_POWER_DISCONNECTED -> false
            Intent.ACTION_POWER_CONNECTED -> true
            else -> null
        }
        isCharging?.let { status ->
            powerLogger.logChargingStatus(status)
        }
    }

}
```
inside `MainActivity`
```kotlin
val filter = IntentFilter(Intent.ACTION_POWER_CONNECTED).also {
            it.addAction(Intent.ACTION_POWER_DISCONNECTED)
        }
        registerReceiver(PowerReceiver(),filter)
```

when the power connects/disconnects we have the following

<img src="/assets/img/dagger/6/power_demonstration_1.png" class="center">


`B.` If you're manually registering the receiver then you can use constructor injection to get an instance of your receiver and register it with that instance, since you're using
```kotlin
registerReceiver(broadcastReceiver, filter)
```

which takes a `broadcastReceiver` instance and an intent filter.

If you're using this approach you can go ahead and remove the `RuntimeBroadcastComponent` and remove it as a subcomponent from the `ApplicationComponent` also
change `PowerReceiver` to
```kotlin
class PowerReceiver @Inject constructor(private val powerLogger: PowerLogger) : BroadcastReceiver() {

    override fun onReceive(context: Context?, intent: Intent?) {
        context ?: return
        intent ?: return

        val isCharging = when (intent.action) {
            Intent.ACTION_POWER_DISCONNECTED -> false
            Intent.ACTION_POWER_CONNECTED -> true
            else -> null
        }
        isCharging?.let { status ->
            powerLogger.logChargingStatus(status)
        }
    }

}
```

and `MainActivity`

```kotlin
class MainActivity : AppCompatActivity() {

    init {
        addOnContextAvailableListener { availableContext->
            injector {
                activityComponentFactory().create(availableContext).also { it.inject(this@MainActivity) }
            }
        }
    }

    @Inject
    lateinit var powerReceiver: PowerReceiver

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val filter = IntentFilter(Intent.ACTION_POWER_CONNECTED).also {
            it.addAction(Intent.ACTION_POWER_DISCONNECTED)
        }
        registerReceiver(powerReceiver,filter)
    }

}
```
As you can see this is easier approach and as you've read in the beginning, always prefer constructor injection whenever possible.

## Service's correct place for injection
---
By now you know how to create a subcomponent or a component, you can get the context from the service, since you've started that service from an Activity/Fragment or a Context, that can be easily casted to `applicationContext` therefore easily creating a subcomponent of the `ApplicationComponent` or if your use case is different you'll know soon enough where to create your graph unrelated to `ApplicationComponent`.

The correct place for the Service to be injected is inside `onCreate()` 
right before the `super.onCreate()` call, just as our `: Application`

```kotlin
class LongRunningService : Service() {

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        //the correct place to inject
        super.onCreate()
    }
}
```

## Work manager
Let's say we have a Worker class
```kotlin
class ContentRecommendation(private val appContext: Context,
                            params: WorkerParameters,
                            private val defaultPrefs: SharedPreferences,
                            private val contentRecommendationAPI: ContentRecommendationAPI) : CoroutineWorker(appContext, params) 
```

for this to work we need to create a WorkerFactory that creates ListenableWorker instances, the factory is invoked every time a work runs.
```kotlin
class ContentFactory(private val defaultPrefs: SharedPreferences,
                     private val contentRecommendationAPI: ContentRecommendationAPI
) : WorkerFactory() {

    override fun createWorker(appContext: Context, workerClassName: String, workerParameters: WorkerParameters): ListenableWorker? {
        return when (workerClassName) {
            ContentRecommendation::class.java.name ->
                ContentRecommendation(appContext, workerParameters, defaultPrefs, contentRecommendationAPI)
            else ->
                // Return null, so that the base class can delegate to the default WorkerFactory.
                null
        }
    }
}
```
then we also need the DelegatingWorkerFactory (injection factory) which delegates to other factories
```kotlin
@Singleton
class ContentRecommendationFactory @Inject constructor(
        defaultPrefs: SharedPreferences,
        contentRecommendationAPI: ContentRecommendationAPI
) : DelegatingWorkerFactory() {
    init {
        addFactory(ContentFactory(defaultPrefs, contentRecommendationAPI))
    }
}
```

then inside `: Application` you have to implement `Configuration.Provider`

and have something like
```kotlin
class WorkerApplicationDemonstration : Application(), Configuration.Provider {

    @Inject
    lateinit var contentRecommendationFactory: ContentRecommendationFactory

    @Inject
    lateinit var contentConfiguration: Configuration

    override fun getWorkManagerConfiguration() = contentConfiguration

    lateinit var appComponent: AppComponent

    override fun onCreate() {
        super.onCreate()
        appComponent = DaggerAppComponent.factory().create(this).also { it.inject(this) }
        WorkManager.initialize(this, contentConfiguration)
    }

}
```

You think it's over? Nope...

Since we've created a custom factory, we have to disable (remove) the default initialization factory or ours won't run.

Inside your `AndroidManifest.xml`
```xml
 <provider
            android:name="androidx.work.impl.WorkManagerInitializer"
            android:authorities="${applicationId}.workmanager-init"
            android:exported="false"
            tools:node="remove" />
```

## What's next?
---
Probably everyone's wondering, why's there no blog post about **Hilt** since it's easier, yes *Hilt* is easier because it does all of this for you behind the scenes so that you don't have to do it, there's one more blog post about the dreaded but powerful `Multi bindings`, I won’t cover WorkManager’s injection using this example but you can realize how much boilerplate we have to write and after the `Multi bindings` post we’re jumping to the Hilt train, which offers easier integration with `WorkManager`.

<img src="/assets/img/dagger/6/patterns.png" class="center">

