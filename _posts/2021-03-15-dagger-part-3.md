---
title: Dagger2 is hard, but it can be easy, part 3
date: 2021-03-15 15:46:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

In the [previous](/posts/dagger-part-2/) post we've seen simpler form of injection with Dagger, but we haven't talked about the types of injection that Dagger2 has to offer.

So far we've seen variable injection but there are construction injection and method injection too.

The easiest of them all is constructor injection and you should strive to use this as much as you can and whenever the circumstances allow you to do so.

For example if we had another class that had a specialty to print these messages, we wanted it to be a part of our **Logger** class, so let's go and create it.

```kotlin
class Messenger @Inject constructor() {

    fun debugMessage(string: String){
        Log.d(this::class.java.simpleName, string)
    }

    fun warningMessage(string: String){
        Log.w(this::class.java.simpleName, string)
    }
}
```

We annotate it with @**Inject** because we want to use it in our **Logger** class, then inside our **Logger** 

```kotlin
class Logger @Inject constructor(
    private val messenger: Messenger
)  {
    fun debug(text: String) {
        messenger.debugMessage(text)
    }
}
```
**remember**, ***we're not using a module*** here because in this sample *we are the creators* of **Logger** and **Messenger**, we'll see later on when *Module* is needed, in our *MainActivity* we're having the same code, untouched

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    internal lateinit var logger: Logger

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        DaggerOurFirstComponent.create().giveGraphModulesToMainActivity(this)

        logger.debug("Dagger2 is easy!")
    }
}
```
run it and it works.

Dagger knows how to construct these objects and it's a breeze working with Dagger on a large project, the compile ***time error detection*** is the best guard you have against *circular* dependencies...

The less common injection, probably one you'll never use is a method injection, the naming shouldn't scare you, you're just annotating a function with @**Inject**, let's see an example.

Let's say we wanted to add a text processor to our **Logger** class that has a functionality to do some heavy duty processing on the text and saves it to storage then send it to server.

For the sake of this demonstration we'll create an interface 

```kotlin
fun interface onMessageReceived{
    fun forMessage(message: String)
}
```
then in our **Logger** class we'll create a variable of that same interface

```kotlin
class Logger @Inject constructor(
    private val messenger: Messenger
)  {

    private lateinit var onMessageReceived: onMessageReceived

    fun debug(text: String) {
        messenger.debugMessage(text)
        onMessageReceived.forMessage(text)
    }

    @Inject
    private fun addTextProcessor(textProcessor: TextToStorage){
       onMessageReceived = textProcessor.addMessageListener()
    }
}
```
and a *function* that adds the text processor, that's annotated with @**Inject** and as you can see we have an argument that's of a type TextToStorage, also the function has to be public due to *annotation processor's* limitations we've read in the [previous](/posts/dagger-part-2/) blog,

```kotlin
class TextToStorage @Inject constructor() {

    init {
        Log.d(this::class.java.simpleName, "initialized")
    }


    fun addMessageListener() = onMessageReceived {
        saveMessageToStorageAndPushToServer(it)
    }

    /**
     * Function that "does some heavy processing and saves the message to storage and sends to server"
     */
    private fun saveMessageToStorageAndPushToServer(text:String){
        Log.d(this::class.java.simpleName, "SAVING MESSAGE TO STORAGE AND SEND TO SERVER $text")
    }
}
```

since our function is annotated with @**Inject** we have to annotate the constructor with @**Inject** so that Dagger *knows what to do*, our **Logger** only sends the message to the listener and doesn't interact anyhow or knows anything about our **TextToStorage** class, the *addTextProcessor* function does everything so that our messages are processed to storage and server.

This example allowed us to notice the order Dagger2 does the injection:

**1st place** is **always** the *constructor injection*

**2nd place** is *method injection* (since the object is constructed, methods are traversed that have @Inject annotation)

**3rd and last place** is *variable injection* (it's pretty obvious, everything needs to be ready/created to inject the variable otherwise how would we do it)

Run the project and see the log

<img src="/assets/img/dagger/3/1.png" class="center">

when **Logger**'s constructor was called and **Logger** received our **Messenger** object, *Dagger* went on further to see for method *injection annotated functions*, it found our *addTextProcessor* and since it knew about **TextToStorage**'s conscturctor, we went on and **initialized** the listener, lastly the variable injection happened and logger.debug was called in the *MainActivity*.

As you can notice that Logger was constructed how it supposed to be, our **Messenger** did it's job and our **TextToStorage** functionality was called without even the need to manually call for a constructor or for the class to know *about the object's creation*, all in all this demonstrates the power of Dagger2 and how it can scale further.

<img src="/assets/img/dagger/3/2.jpg" class="center">

[Part #4](/posts/dagger-part-4/)
