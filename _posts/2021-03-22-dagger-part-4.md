---
title: Dagger2 is hard, but it can be easy, part 4
date: 2021-03-22 22:15:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

In the [previous](/posts/dagger-part-3/) post we've encountered method injection and learnt about the order of injection.

Today we'll explore @**Singleton**, remember that @**Singleton** is a scope operator.

From the docs we have
```kotlin
/**
 * Identifies a type that the injector only instantiates once. Not inherited.
 *
 * @see javax.inject.Scope @Scope
 */
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

@**Singleton** makes sure you have only one instance of your class in the Dagger graph and it always returns the same instance, it's an annotation @Singleton which is misleading,

@**Singleton** != Singleton pattern

Scoping is a mechanism defined in *JSR-330*.
What does this mean is that, if you scope annotate something, the injector (Dagger) will retain (cache) the instance for possible reusability whenever injection happens.

*JSR-330* has one predefined scope annotation — @Singleton

It gives you the impression that you're getting a singleton but you're not, let's see this case.

```kotlin
@Singleton
class Logger @Inject constructor(
    private val messenger: Messenger
)  {

    init {
        debug("LOGGER INITIALIZED") //testing to see if it's really a singleton
    }

    private lateinit var onMessageReceived: onMessageReceived

    fun debug(text: String) {
        messenger.debugMessage(text)
        //onMessageReceived.forMessage(text) comment out the line for demonstration now
    }

    @Inject
    fun addTextProcessor(textProcessor: TextToStorage){
       onMessageReceived = textProcessor.addMessageListener()
    }

}
```

We've annotated the class with @**Singleton** scope and also our component 
```kotlin
@Component
@Singleton
interface OurFirstComponent {
    fun giveGraphModulesToMainActivity(mainActivity: MainActivity)
}
```

now you think okay, we do have a singleton yaay
<img src="/assets/img/dagger/4/not.gif" class="center">

Let's create another activity for demonstration

```kotlin
class SecondActivity : AppCompatActivity() {

    @Inject
    internal lateinit var logger: Logger

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        DaggerOurFirstComponent.create().giveGraphModulesToSecondActivity(this)

        logger.debug(this::class.java.simpleName)
    }
}
```
go to our component

```kotlin
@Component
@Singleton
interface OurFirstComponent {
    fun giveGraphModulesToMainActivity(mainActivity: MainActivity)
    fun giveGraphModulesToSecondActivity(secondActivity: SecondActivity)
}
```

and inside the *MainActivity*
```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    internal lateinit var logger: Logger

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        DaggerOurFirstComponent.create().giveGraphModulesToMainActivity(this)

        logger.debug(this::class.java.simpleName)

        Handler(Looper.getMainLooper()).postDelayed({
            startActivity(Intent(this, SecondActivity::class.java))
        }, 1000)
    }
}
```

**don't forget** to add the activity inside the *manifest* and run the app.

<img src="/assets/img/dagger/4/1.png" class="center">

Uhm, the Logger was initialized two times, confusing right?

Let's demonstrate a real singleton

```kotlin
object TestSingleton {
    init {
        Log.d("Real singleton", "INITIALIZED")
    }

    fun log(text: String) {
        Log.d("Real singleton ${this::class.java}", text)
    }
}
```

inside your ***MainActivity*** and your ***SecondActivity*** add 
```kotlin
TestSingleton.log(this::class.java.simpleName)
```
and run the app

<img src="/assets/img/dagger/4/2.png" class="center">

as you can clearly see the object is initialized only once, so who's to blame here?

**OurFirstComponent** is the culprit, why?
Because we called
```kotlin
DaggerOurFirstComponent.create()
```
in our activities, but they're created every time you launch a new one right?

So... how do we fix this?

In Android we have `Application` level inheritance which is called only once when the app is launched and every time process death restoration happens.

You might think, okay so, in our first component I need to add a function that creates the object only when `Application` creation happens and you go ahead and make the following

```kotlin
@Component
@Singleton
interface OurFirstComponent {
    fun createSingletons(testApplication: TestApplication)
}
```

and inside the activities

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        (application as TestApplication).logger.debug(this::class.java.simpleName)

        Handler(Looper.getMainLooper()).postDelayed({
            startActivity(Intent(this, SecondActivity::class.java))
        }, 1000)
    }
}
```

```kotlin
class SecondActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        (application as TestApplication).logger.debug(this::class.java.simpleName)
    }
}
```
<img src="/assets/img/dagger/4/3.png" class="center">

We've achieved some form of *singleton*, but not really, once process death restoration happens, the objects annotated with @**Singleton** are re-created again.

In an Android world, **Singleton** doesn't really exist, it's a lie.

P.S: 
When Dagger creates the *implementation* behind the scenes it double checks for the component's initialization once you annotate something with a scope, in our case @**Singleton** or any other custom scope (we'll see how to create that later on) you'll use in the future, meanig it caches the value that you scope (as we've mentioned in the beginning), as for homework you can see how they've [implemented](https://github.com/google/dagger/blob/master/java/dagger/internal/DoubleCheck.java) it.

Thanks for the wholeharted attention.

<img src="/assets/img/dagger/4/great_success.gif" class="center">

