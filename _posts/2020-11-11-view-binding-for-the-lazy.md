---
title: View binding for the lazy
date: 2020-11-05 11:46:00 +0100
categories: [Android, Kotlin]
tags: [Android, Kotlin]
---

<img src="/assets/img/2/1.png" class="center">

If you ever looked at this image and wondered, why this blog post is written when I as a developer was promised a low amount of code needed for view binding to work, well technically that's true, but also false at the same time.
 
You see... everything in Android is complicated and you should make peace with it, well you might ask yourself how?
Just build a workaround solution that'll ease your life, thank me later.
 
TL:DR; use the [library/helper](https://github.com/FunkyMuse/KAHelpers/tree/master/viewbinding) I published to forget even copy/pasting the code from this post.
 
On a normal day you'll have something like
 

```kotlin
private lateinit var activityMainBinding: ActivityMainBinding
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    activityMainBinding = ActivityMainBinding.inflate(inflater)
    setContentView(activityMainBinding.root)
}
```
which is a lot of code to write, whopping 3 lines.

Okay we can definitely make it better, thanks to Kotlin.
 
```kotlin
private val activityMainBinding by viewBinder(ActivityMainBinding::inflate)
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(activityMainBinding.root)
}
```


That looks way better doesn't it?
 
What's happening here?

>We have an inline delegate + extension function thanks to the lazy (more on that)
We access ActivityMainBinding's function inflate via reference  
Magically inflated

Let's take a look at our viewBinder function

```kotlin
inline fun <T : ViewBinding> Activity.viewBinder(
        crossinline bindingInflater: (LayoutInflater) -> T) =
        lazy(LazyThreadSafetyMode.NONE) {
            bindingInflater.invoke(layoutInflater)
        }
```

 We have an inline function of course to minimize the overhead, then we have another function as a parameter that takes the LayoutInflater as a parameter and returns the generic T which is our ViewBinding.

Our lazy delegate provided by Kotlin is how we get the by in our viewBinder and we use LazyThreadSafeMode.NONE since the view can't be accessed on other threads and thus we don't need the lazy to create even more overhead with the synchronizations.

We marked our bindingInflater function as a crossinline since it's called in another scope of a function (lazy), and then we just invoke it (which means inflating our view binding) with a parameter of layoutInflater which is context.getLayoutInflater in the background.

Now we have to write only 2 lines of code instead of 3.

But that's too much, 2 lines of code, nope, we can do better, we are lazy.

Le few moments later and here we are.
```kotlin
private val activityMainBinding by viewBinding(ActivityMainBinding::inflate)
override fun onPostCreate(savedInstanceState: Bundle?) {
    super.onPostCreate(savedInstanceState)
    
}
```

Same, but different, but still the same, huh?
Let's see.

```kotlin
fun <T : ViewBinding> AppCompatActivity.viewBinding(
        bindingInflater: (LayoutInflater) -> T,
        beforeSetContent: () -> Unit = {}) =
        ActivityViewBindingDelegate(this, bindingInflater, beforeSetContent)
```
Okay then, we have almost the same logic, except this time it ain't an inline function and there's another beforeSetContent function and another class ActivityViewBindingDelegate, let's jump to it.

This class is a really simple class, it leverages Kotlin's delegates

class ActivityViewBindingDelegate<T : ViewBinding>(
        private val activity: AppCompatActivity,
        private val viewBinder: (LayoutInflater) -> T,
        private val beforeSetContent: () -> Unit = {}
) : ReadOnlyProperty<AppCompatActivity, T>, LifecycleObserver

It has a generic of a view binding type and generally identical parameters as the function above and instead of an extension function our AppCompatActivity is a the type of object which owns the property that's delegated, in our case the ViewBinding.
```kotlin
class ActivityViewBindingDelegate<T : ViewBinding>(
        private val activity: AppCompatActivity,
        private val viewBinder: (LayoutInflater) -> T,
        private val beforeSetContent: () -> Unit = {}
) : ReadOnlyProperty<AppCompatActivity, T>, LifecycleObserver {

    private var activityBinding: T? = null

    init {
        activity.lifecycle.addObserver(this)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun createBinding() {
        initialize()
        beforeSetContent()
        activity.setContentView(activityBinding?.root)
        activity.lifecycle.removeObserver(this)
    }

    private fun initialize() {
        if (activityBinding == null) {
            activityBinding = viewBinder(activity.layoutInflater)
        }
    }

    override fun getValue(thisRef: AppCompatActivity, property: KProperty<*>): T {
        ensureMainThread()
        initialize()
        return activityBinding!!
    }

    private fun ensureMainThread() {
        if (Looper.myLooper() != Looper.getMainLooper()) {
            throw IllegalThreadStateException("View can be accessed only on the main thread.")
        }
    }
}
```

Let's do a step by ste...

We have our
```kotlin
private var activityBinding: T? = null
```
Which holds our view binding and once the delegate's constructor gets called we initialize our LifecycleObserver by calling
```kotlin
init {
    activity.lifecycle.addObserver(this)
}
```
so that we can listen to events like Lifecycle.Event.OnSomething

we need to initialize our view binding
 
```kotlin
@OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
fun createBinding() {
    initialize()
    beforeSetContent()
    activity.setContentView(activityBinding?.root)
    activity.lifecycle.removeObserver(this)
}

private fun initialize() {
    if (activityBinding == null) {
        activityBinding = viewBinder(activity.layoutInflater)
    }
}
```

We have nearly the same logic as above but we have 

1. initialize() - self explanatory
2. beforeSetContent() - which takes place before setContentView(), this is useful for all the Dagger component 3.injections and other stuff you need to do before setContentView happens
3. Then we set the content view
4. We remove the lifecycle observer???? yeah we do because setContentView has to happen only once, call it magic, call it true.

We override the
```kotlin
override fun getValue(thisRef: AppCompatActivity, property: KProperty<*>): T {
    ensureMainThread()
    initialize()
    return activityBinding!!
}
```

C'mon man, already... it was supposed to be simple, it isn't..... not yet.

We have to ensure access only on the main thread and then do initialization phase in case it's a null (process death enters the chat).

and now we can do 

private val activityMainBinding by viewBinding(ActivityMainBinding::inflate) {
    //initialize Dagger voodoo or something else before setContentView
}

with just one (almost one if you don't do dagger initialization or something else you can omit the '{}')
 
Why are we using onPostCreate, well the lifecycle observe propagates the lifecycle event as a reflective call to the function where we initialize the view binding, therefore we have to use the view somehow and that's why we do it onPostCreate, this function is a call in-between onCreate and onStart, it's where you do final initialization before usage.

```kotlin
private val activityMainBinding by viewBinding(ActivityMainBinding::inflate) {
    //initialize Dagger voodoo
}
override fun onPostCreate(savedInstanceState: Bundle?) {
    super.onPostCreate(savedInstanceState)
    
}
```

ain't it elegant?


You might ask yourself if we've forgotten the complication of our lives called Fragments, nope, not yet.

```kotlin
class TestFragment : Fragment() {

    private var testFragmentBinding: TestFragmentBinding? = null

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        testFragmentBinding = TestFragmentBinding.inflate(inflater, container, false)
        return testFragmentBinding?.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        testFragmentBinding = null
    }

}
```
We have to create it, inflate it, set it to null in onDestroyView()

<img src="/assets/img/2/2.gif" class="center">


But why set it to null?

Fragments can be detached (when that happens the view is the only one that gets killed) but they'll still live, except if we don't destroy the view, mister memory leak appears.

```kotlin
class TestFragment : Fragment(R.layout.test_fragment) {

    private val testFragmentBinding by viewBinding(TestFragmentBinding::bind)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        //do your stuff
    }
}
```

```kotlin
fun <T : ViewBinding> Fragment.viewBinding(
        viewBindingFactory: (View) -> T, 
        disposeEvents: T.() -> Unit = {}) =
        FragmentViewBindingDelegate(this, viewBindingFactory, disposeEvents)
```
Similar to our activity case, but instead, our parameter here is a view and we have disposeEvents which is invoked on an object and this FragmentViewBindingDelegate inspired by Zhuinden's post

```kotlin
class FragmentViewBindingDelegate<T : ViewBinding>(private val fragment: Fragment,
                                                   private val viewBinder: (View) -> T,
                                                   private val disposeEvents: T.() -> Unit = {}) : ReadOnlyProperty<Fragment, T>, LifecycleObserver {

    init {
        fragment.observeLifecycleOwnerThroughLifecycleCreation {
            lifecycle.addObserver(this@FragmentViewBindingDelegate)
        }
    }
    
    private inline fun Fragment.observeLifecycleOwnerThroughLifecycleCreation(crossinline viewOwner: LifecycleOwner.() -> Unit) {
        lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onCreate(owner: LifecycleOwner) {
                viewLifecycleOwnerLiveData.observe(this@observeLifecycleOwnerThroughLifecycleCreation, Observer { viewLifecycleOwner ->
                    viewLifecycleOwner.viewOwner()
                })
            }
        })
    }

    private var fragmentBinding: T? = null

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun disposeBinding() {
        fragmentBinding?.disposeEvents()
        fragmentBinding = null
    }

    override fun getValue(thisRef: Fragment, property: KProperty<*>): T {
        ensureMainThread()
        val binding = fragmentBinding
        if (binding != null) {
            return binding
        }
        val lifecycle = fragment.viewLifecycleOwner.lifecycle
        if (!lifecycle.currentState.isAtLeast(Lifecycle.State.INITIALIZED)) {
            throw IllegalStateException("Fragment views are destroyed.")
        }
        return viewBinder(thisRef.requireView()).also { fragmentBinding = it }
    }
}
```
It's kinda obnoxious to look at first, but let's take a dive,

We have our Fragment again that's holding the property, our viewBinder which accepts a View as a parameter and disposeEvents function (more on that later).

```kotlin
init {
    fragment.observeLifecycleOwnerThroughLifecycleCreation {
        lifecycle.addObserver(this@FragmentViewBindingDelegate)
    }
}
```
```kotlin
private inline fun Fragment.observeLifecycleOwnerThroughLifecycleCreation(crossinline viewOwner: LifecycleOwner.() -> Unit) {
    lifecycle.addObserver(object : DefaultLifecycleObserver {
        override fun onCreate(owner: LifecycleOwner) {
            viewLifecycleOwnerLiveData.observe(this@observeLifecycleOwnerThroughLifecycleCreation, Observer { viewLifecycleOwner ->
                viewLifecycleOwner.viewOwner()
            })
        }
    })
}
```
We need to listen to the lifecycle somehow ... we need to know when the viewLifecycle is created so that we can do our bindings, thankfully Fragments have

```kotlin
viewLifecycleOwnerLiveData
```

which we can use to observe and provide lifecycle events we have to listen to the lifecycle onCreate event then and only then we know that we can observe viewLifecycleOwnerLiveData and take the observation result to add an observer (listen for events) to this delegate, FRAGMENTS ARE EASY.

```kotlin
override fun getValue(thisRef: Fragment, property: KProperty<*>): T {
    ensureMainThread()
    val binding = fragmentBinding
    if (binding != null) {
        return binding
    }
    val lifecycle = fragment.viewLifecycleOwner.lifecycle
    if (!lifecycle.currentState.isAtLeast(Lifecycle.State.INITIALIZED)) {
        throw IllegalStateException("Fragment views are destroyed.")
    }
    return viewBinder(thisRef.requireView()).also { fragmentBinding = it }
}
```

Ensuring our views are accessed on the main thread,
initializing the binding, checking what the lifecycle of our Fragment is, so that we can't do any accessing when onDestroyView is called or onDestroy

and then where the magic happens is 

```kotlin
return viewBinder(thisRef.requireView()).also { fragmentBinding = it }
```
it takes our viewBinder function from the constructor, it gives it a parameter thisRef.requireView() which points to the Fragment that calls requireView(), which is the same as getView but it throws an exception that we have already solved with the lifecycle check and then .also allows us to set our fragmentBinding to the view that we just initialized and that's T, in our case a type of View Binding that we set to our fragmentBinding variable.

But wait, there's more.

You wanted to do dispose events, clean ups.

If you try onDestroyView() or onDestroy() you'll get IllegalStateException that views are destroyed and you'll think onPause and onStop are the perfect place to do disposes, they are but not quite your use case?

Hey, we got that case covered

```kotlin
@OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
fun disposeBinding() {
    fragmentBinding?.disposeEvents()
    fragmentBinding = null
}
```

before we dispose our Bindings we have the disposeEvents that are tied to the viewLifecycleOwner's onDestroy event, so we can have something like this

```kotlin
private val testFragmentBinding by viewBinding(TestFragmentBinding::bind){
    //do the dispose here
    this.recyclerView.adapter = null
}
```

and we would forget about even calling onDestroyView.


That's it we made view binding easy to use, just go into your app's build gradle and enable it.

```groovy
buildFeatures {
    viewBinding = true
}
```

and/or include the library/helper I published to forget even copy/pasting the code from this post.

That's it folks, View binding is now easy to work with, an elegant solution and replacement for synthetics which are soon to be deprecated and the nasty findViewById, no more deprecations huh, finally a good solution?

<img src="/assets/img/2/3.jpg" class="center">
