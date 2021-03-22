---
title: Dagger2 is hard, but it can be easy, part 2
date: 2021-03-08 15:46:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

We're here again telling ourselves that Dagger2 is easy, as we've seen in the first [part](/posts/dagger-part-1/).

To demystify some myths about Dagger2 we'll see it's simplicity in the smallest blog post so far.

From the previous article we had a @**Module**, @**Component** and @**Inject**.

What if I told you that you can achieve the simplest dependency injection with @**Inject** and @**Component**, without the need of a ***module***...

As I previously mentioned, Dagger2 is really easy, Android makes it hard, enough talk, showing you the code.


We go ahead and comment out **OurFirstModule** class
```kotlin
/*@Module
class OurFirstModule {

    @Provides
    fun provideLogger(): Logger = Logger()
}*/

```

```kotlin
class Logger {
        fun log(text: String){
            Log.d(this::class.java.simpleName, text)
        }
    }
```

and in our MainActivity

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

Our component looks simple enough

```kotlin
@Component
interface OurFirstComponent {
    fun giveGraphModulesToMainActivity(mainActivity: MainActivity)
}
```

and we just commented our Module and removed the `(modules = [OurFirstModule::class])` annotation's parameter.

See?
Simple.

What does @**Inject** do?

It gives you the object you annotated @**Inject** with, in our case the **Logger** we created.

Annotating an object you want to get with @**Inject** to get that same object later on using @**Inject** but as a variable.

Since we mentioned variables, you might ask yourself why the variables can't be private?

Dagger does not support injection into private fields...

Oh okay Dagger, but why?

When the graph has your object it knows about the package and it's name when you annotated it with @**Inject** since this is transformed by an *annotation processor*, that's a limitation of the *annotation processor*, it needs to be visible in the package, so that means you can make the variable internal in this case and it will work, basically Dagger sets the lateininit value to the one it has in the graph but it can't be private due to the the variable not visible out of the class.

<img src="/assets/img/dagger/2/1.gif" class="center">

[Part #3](/posts/dagger-part-3/)

