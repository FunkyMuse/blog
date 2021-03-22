---
title: Dagger2 is hard, but it can be easy, part 1
date: 2021-03-01 15:46:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

One day you decided to use Dagger2 and then you were lost, don't worry, most of us were already there, Dagger2 is easy, you'll see, it's literally one interface, let's start.

A little bit of history, Dagger2 is a continuation by Google from the Dagger 1 made by Square.

It's a compile time DI library and this comes in handy due to the validation, based on the 330 JSR DI standard that's just one interface 

<img src="/assets/img/dagger/1/1.png" class="center">

We won't talk about the dreaded dagger-android here because some mistakes should remain mistakes and we won't be talking about the thermosiphon eithet.

All you need is these two lines to get you started (look up the latest version for Dagger2, as of now when this article was written, it's 2.33)

```groovy
implementation 'com.google.dagger:dagger:2.33'
kapt 'com.google.dagger:dagger-compiler:2.33'
```
and don't forget to add **kapt** as a *plugin*.


Now, what's **Dagger** actually?

It's a graph, a graph where your objects live.

It's named **Dagger** because:

Directed Acyclic Graph + Ger (don't worry the Ger doesn't mean acidic reflux)

meaning that, we need to make sure there are directed dependency graph edges from low level modules to high level modules, without any dependency cycles formed (this is detected at compile time).

Let's start with our best friend the

**@Module**

we have a graph, but that graph doesn't know how to construct our objects, our friend @Module provides instruction how to build these objects.

Let's create our first module

We created a simple class ***Logger***

```kotlin
class Logger {
        fun log(text: String){
            Log.d(this::class.java.simpleName, text)
        }
    }
```

and a module called **OurFirstModule** and annotated it with @**Module**

```kotlin
@Module
class OurFirstModule {

    @Provides
    fun provideLogger(): Logger = Logger()
}
```

then we have a function called *provideLogger*() the naming is probably a convention that you'll have internally in your company or how you see fit, this is to get you started and as you can see that we instantiated the object here, so that means our **Logger** is now a part of the Dagger graph, every time you ask for a Logger instance, Dagger will give you one.

So... now we want to get this **Logger** somehow in our MainActivity and log some message.

But *MainActivity* is a high level module and **Dagger** doesn't know anything about it, in order to build a bridge we need the friendliest of them all @**Component**.

@**Component** makes sure that all those objects you created in your module end up to the place where you tell it to by using @**Inject**.

We go ahead and create an interface called *OurFirstComponent* and annotate it with @**Component** 

```kotlin
@Component
interface OurFirstComponent {
    fun giveGraphModulesToMainActivity(mainActivity: MainActivity)
}
```
inside the annotation it's gonna ask which modules we need to know about so that we can transfer them to *MainActivity* and other high level modules later on.

We provide an array of our first module as a single object, for now.

The function we wrote inside is self explanatory, then once you write that function you click **build(F10)**.

Let's go to our *MainActivity*

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    internal lateinit var logger: Logger

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        logger.debug("Dagger2 is easy!")
    }
}
```

@**Inject** gives what you ask for, in this case we ask for our **Logger** (but you may ask yourself... we can just instantiate Logger since we have control over it, but let's pretend for a moment that it's coming from a library that you included in your app).

Then you click run and 💥 your app crashes.

For a moment there you start hating Dagger2, but the real pain is Android, we'll see later down the road that Dagger2 is even simpler than this, really simple but Android makes things complicated.

Okay, let's see what we need to do, we told Dagger2 about our **Logger** and it knows how to make it, we also told our component to bring it over to *MainActivity* but we didn't build our bridge(call our @**Component**) at the place where we need our instance to cross over from the other side.

The way it goes it's like this, Dagger2 calls our friend @**Module** and module provides instructions on how to build our **Logger**, when @**Component** tries to take our **Logger** from the construction site to *MainActivity* it has to know about it, @**Component** has a function that knows about *MainActivity* 
```kotlin
fun giveGraphModulesToMainActivity(mainActivity: MainActivity)
```
but that function is never called so it doesn't know about MainActivity, yet!

When you clicked **build(F10)**, Dagger2 generated a dependency-injected implementation for your @**Component** with the set of modules you gave it knowledge about (in our case `[MyFirstModule::class]`, the generated implementation is named `DaggerOurFirstComponent`.

It always starts with Dagger plus the name of the component.

That component has a **create**() method that you need to call to create that bridge (our component) and connect it with *MainActivity* with the function from above, all in all

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
when you run the code, you can see that it doesn't crash.

You may ask yourself why the variable can't be private, where is the correct way to build the component (bridge) for our objects we'll leave that for next time, this may take some time in order for you to stomach it, but it's gonna fuel you in the long run, once you wield the power of Dagger, you'll love it.

Remember, Dagger2 is easy 

<img src="/assets/img/dagger/1/2.gif" class="center">

This is just the tip of the iceberg, more is coming.

[Part #2](/posts/dagger-part-2/)
