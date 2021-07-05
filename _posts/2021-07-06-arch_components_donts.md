---
title: Common mistakes when using Architecture Components
date: 2021-07-05 21:20:00 +0100
categories: [Jetpack, Android, Architecture Components]
tags: [Jetpack, Android, Architecture Components]
---

<img src="/assets/img/arch_components_mistakes/1.jpeg" class="center">

Android, ehh, Android.... development is what is always described as one of the painful things, but we still love it tho, of course there's a reason why Android's messy, but have you seen front-end development?

Everything was peaceful untill `Fragment`s were introduced with their own lifecycle and to make things messier, Google decided to make things even more messier, `Fragment`'s view will have it's own lifecycle.

<img src="/assets/img/arch_components_mistakes/burn.gif" class="center">


Nevertheless `Fragment`'s nowadays are matured and more stable than the ones 2 years ago.

And Google also released architecture components to ease our lives, or did they?

There are several mistakes you can make when using architecture components, even if you don't make them, you should be aware.

## Leaking LiveData observers in Fragments
---

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewModel.liveData.observe(this){ updateUI(it) } // this means the Fragment as a LifecycleOwner

}
```
Google's lint forces you to use `getViewLifecycleOwner()`, because if you're using the Fragment as a `LifecycleOwner` you're always adding observers and never clearing them because the `LifecycleOwner` didn't reach the `DESTROY`ed state.

🚫 don't use Fragment as a `LifecycleOwner` when observing Live Data

✅ Use the getter for `viewLifecycleOwner`, since your observer has a `DESTROY`ed state when it reaches `onDestroyView`

## Observing flows in launchWhenX
---

Coroutines were pushed as a replacement for `LiveData` but there wasn't something that'll clear up the references.

You decided to get rid of LiveData and you started using `viewLifeCycleOwner.lifecycleScope.launchWhenResumed` or `viewLifeCycleOwner.lifecycleScope.launchWhenStarted` to ensure that a given coroutine runs only in that specific lifecycle state, there's a [deep dive article worth reading](https://medium.com/swlh/deep-dive-into-lifecycle-coroutines-e7192312faf#7d09) why that's the thing you shouldn't use.

TL:DR; flows can be dropped when config changes, every time you call this function `LifecycleController` and `PausingDispatcher` are created and also `launchWhenX` functions also use `Dispatchers.Main.immediate`

🚫 don't use `launchWhenX`

✅ Use the new `repeatOnLifecycle` or just use `flow.asLiveData`

*Thanks to Manuel Vivo who provided a [great explanation](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333)*.


## Reloading data after every rotation and creation of the view in the Fragment
---

How many PRs I have rejected because of this thing I lost count.

It comes in cancerous form like 

```kotlin
@HiltViewModel
class DetailedMovieViewModel @Inject constructor(
    private val moviesRepository: MoviesRepo,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val args = DetailedMovieFragmentArgs.fromSavedStateHandle(savedStateHandle)
    private val movieID get() = args.movieId

    private val movieData = retrofitStateInitialLoading<DetailedMovieModel>()
    val movie = movieData.asStateFlow()

    fun fetchMovieData() {
        viewModelScope.launch {
            movieData.value = withContext(Dispatchers.IO){
                moviesRepository.getDetailedMovie(movieID)
            }
        }
    }

}
```

then we head over where my head hurts the most

```kotlin
@AndroidEntryPoint
class DetailedMovieFragment : Fragment(R.layout.fragment_detailed_movie){

    private val viewModel by viewModels<DetailedMovieViewModel>

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewModel.fetchMovieData() // <----- this line
    }
}
```

How do we fix this?

Simple.

```kotlin
@HiltViewModel
class DetailedMovieViewModel @Inject constructor(
    private val moviesRepository: MoviesRepo,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val args = DetailedMovieFragmentArgs.fromSavedStateHandle(savedStateHandle)
    private val movieID get() = args.movieId

    private val movieData = retrofitStateInitialLoading<DetailedMovieModel>()
    val movie = movieData.asStateFlow()

    init {
        fetchMovieData()
    }

    private fun fetchMovieData() {
        viewModelScope.launch {
            movieData.value = withContext(Dispatchers.IO){
                moviesRepository.getDetailedMovie(movieID)
            }
        }
    }

}
```
```kotlin
@AndroidEntryPoint
class DetailedMovieFragment : Fragment(R.layout.fragment_detailed_movie){

    private val viewModel by viewModels<DetailedMovieViewModel>

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.movie.collect { movie ->
                    updateUI(movie)
                }
            }
        }
        
    }
}
```

🚫 don't call the function to fetch the data in  `onViewCreated` nor in `onCreateView` nor anywhere inside the `Fragment` unless it's an action that's triggered by the user input

✅ Use the `init` function inside the `ViewModel`, that's called only once when the `ViewModel` is created.


<img src="/assets/img/arch_components_mistakes/2.jpg" class="center">


## Misusing data holders
---

`MutableLiveData` and `LiveData` are data holders, there are several mistakes you can make.

### Data holders as variables

```kotlin
var data : MutableLiveData<DetailedMovieModel> = MutableLiveData()
```

```kotlin
var data : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
```

🚫 Don't use `var` as a way to create a data holder

✅ Use `val`

You're creating unnecessary getter for the data holder, if `MutableLiveData` was a variable you could've just assigned the value to it directly without using `.value` or `postValue()`.

### Exposing mutable data holder instead of immutable

You had your data holder fixed but 

```kotlin
val data : MutableLiveData<DetailedMovieModel> = MutableLiveData()
```

```kotlin
val data : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
```

```kotlin

class MovieFragment : Fragment(){

    private val viewModel by viewModels<DetailedMovieViewModel>

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewModel.data.observe....
    }

    private fun doAMistake(){
        viewModel.data.value = someMistake
    }
}

```

I've seen code like this and it makes my yesterday's meal turn over.

```kotlin
private val movieData : MutableLiveData<DetailedMovieModel> = MutableLiveData()
val movie : LiveData<DetailedMovieModel> = movieData
```

```kotlin
private val movieData : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
val movie = movieData.asStateFlow()
```

```kotlin

class MovieFragment : Fragment(){

    private val viewModel by viewModels<DetailedMovieViewModel>

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewModel.movie.observe....
    }

    private fun doAMistake(){
        viewModel.movie.value = someMistake //forbidden because it's immutable and you can't do anything
    }
}

```

🚫 Don't expose `Mutable` data holder to the `Fragment`, if you're using two-way binding with data-binding, then stop using data-binding and use view-binding, this is a harsh statement but just don't use `data-binding`

✅ Expose an immutable data holder

### Using underscore for naming the mutable state holder

```kotlin
private val _movie : MutableLiveData<DetailedMovieModel> = MutableLiveData()
val movie : LiveData<DetailedMovieModel> = _movie
```

```kotlin
private val _movie : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
val movie = _movie.asStateFlow()
```

If you opened a code base like this and you were confused sometimes because someone exposed the immutable thing to the `Fragment` sometimes the mutable and they didn't even think to refactor, then it's your job to slap them for me.

Use more descriptive name, something like `data` or etc... get creative

```kotlin
private val movieData : MutableLiveData<DetailedMovieModel> = MutableLiveData()
val movie : LiveData<DetailedMovieModel> = movieData
```

```kotlin
private val movieData : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
val movie = movieData.asStateFlow()
```

🚫 Don't use `_name` for the love of everyone on your team

✅ Use more descriptive name, something like `data` or get creative.


### Using getters for the immutable live data holder

```kotlin
private val movieData : MutableLiveData<DetailedMovieModel> = MutableLiveData()
val movie : LiveData<DetailedMovieModel> get() = movieData
```

```kotlin
private val movieData : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
val movie get() = movieData.asStateFlow()
```

Kotlin docs just point it out for you
```text
If you define a custom getter, it will be called every time you access the property (this way you can implement a computed property)
```


🚫 Don't use a getter for the immutable data holder

✅ Initialize it immediately after the mutable data holder

```kotlin
private val movieData : MutableLiveData<DetailedMovieModel> = MutableLiveData()
val movie : LiveData<DetailedMovieModel> = movieData
```

```kotlin
private val movieData : MutableStateFlow<DetailedMovieModel?> = MutableStateFlow(null)
val movie = movieData.asStateFlow()
```

<img src="/assets/img/arch_components_mistakes/2.jpg" class="center">


## Leaking ViewModels
---
It is clearly [highlighted](https://developer.android.com/topic/libraries/architecture/viewmodel#implement) that we shouldn’t be passing `View` references to `ViewModel`, that's clear, otherwise you're just killing the purpose of the `ViewModel`.

### Forgetting to unsubscribe a listener

```kotlin
@Singleton
class LocationRepository @Inject constructor() {

    private var localListener: ((Location) -> Unit)? = null

    fun setOnLocationChangedListener(listener: (Location) -> Unit) {
        localListener = listener
    }

    fun removeOnLocationChangedListener() {
        localListener = null
    }

    private fun onLocationUpdated(location: Location) {
        listener?.invoke(location)
    }
}

@HiltViewModel
class MapViewModel @Inject constructor(
    private val repository : LocationRepository
): ViewModel() {

    private val locationData = MutableStateFlow<LocationRepository.Location?>(null)
    

    init {
        repository.setOnLocationChangedListener {   
            locationData.value = it
        }
    }
  
}
```

In this sample we're not clearing the reference and we can't make the `listener` garbage collectable.

```kotlin
@HiltViewModel
class MapViewModel @Inject constructor(
    private val repository : LocationRepository
): ViewModel() {

    private val locationData = MutableStateFlow<LocationRepository.Location?>(null)
    

    init {
        repository.setOnLocationChangedListener {   
            locationData.value = it
        }
    }

    override onCleared() {                           
        repository.removeOnLocationChangedListener()
    }  
}
```

🚫 Don't just create references that'll be held in the `ViewModel`

✅ Every reference has to be cleared in `onCleared()`


### Observing inside a ViewModel

You know about NOT referencing Views inside a `ViewModel` DO NOT DO OBSERVE ANYTHING INSIDE A `ViewModel` AS WELL!

> ViewModel objects are designed to outlive specific instantiations of views or LifecycleOwners. This design also means you can write tests to cover a ViewModel more easily as it doesn't know about view and Lifecycle objects. ViewModel objects can contain LifecycleObservers, such as LiveData objects. However ViewModel objects must never observe changes to lifecycle-aware observables, such as LiveData objects. If the ViewModel needs the Application context, for example to find a system service, it can extend the AndroidViewModel class and have a constructor that receives the Application in the constructor, since Application class extends Context.

One day I opened the `proton-mail-android` app and glanced over [EditContactDetailsViewModel](https://github.com/ProtonMail/proton-mail-android/blob/release/app/src/main/java/ch/protonmail/android/contacts/details/edit/EditContactDetailsViewModel.kt)

<img src="/assets/img/arch_components_mistakes/3.png" class="center">

🚫 Don't ever do this.

✅ Everything you need to observe should be in a `Fragment` or an `Activity`.


## Observing in the wrong places
---
🚫 Do not observe inside `onResume` 

🚫 Do not observe inside `onCreate` in the `Fragment`

🚫 Do not observe inside a `Service` unless you manage it's lifecycle, same goes for a `BroadcastReceiver`

## Resetting the current observer
---
```kotlin
fun <T> LiveData<T>.resetObserver(owner: LifecycleOwner, observer: Observer<T>) {
    removeObserver(observer)
    observe(owner, observer)
}
```

 An observer is unsubscribed before an identical one is subscribed hopefully in `onViewCreated` by removing and adding the same  exact observer you will reset the state so that `LiveData` will deliver the latest result again when `onStart()` happens.

 🚫 Do not do this.


## The final boss, MutableLiveData for single shot events
---

Sorry `Proton` for bashing your code base, again I glanced over at [MailboxViewModel](https://github.com/ProtonMail/proton-mail-android/blob/release/app/src/main/java/ch/protonmail/android/activities/MailboxViewModel.kt)

```kotlin
    private val _hasSuccessfullyDeletedMessages = MutableLiveData<Boolean>()
    
    val hasSuccessfullyDeletedMessages: LiveData<Boolean>
        get() = _hasSuccessfullyDeletedMessages

```

in the `Activity`
```kotlin
mailboxViewModel.hasSuccessfullyDeletedMessages.observe(
            this,
            { isSuccess ->
                //Timber.v("Delete message status is success $isSuccess")
                if (!isSuccess) {
                    showToast(R.string.message_deleted_error)
                }
            }
        )
```

<img src="/assets/img/arch_components_mistakes/least.gif" class="center">

After some time Google came up with a hack solution to use `Event`

```kotlin
class Event<out T>(private val content: T) {

    private val hasBeenHandled = AtomicBoolean(false)

    /**
     * Returns the content and prevents its use again.
     */
    fun getContentIfNotHandled(): T? {
        return if (!hasBeenHandled.getAndSet(true)) {
            content
        } else {
            null
        }
    }

    /**
     * Returns the content, even if it's already been handled.
     */
    fun peekContent(): T = content
}
```

Which you need to use `getContentIfNotHandled()` first in order to see if the content is consumed, I just deleted that thing and went to use a `Channel`

🚫 do not use `MutableLiveData` or flow to show a `Toast` a `SnackBar` or even worse a `Dialog` or something that'll just spam the user endlessly.

✅ Use a [Channel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/) or the hacky solution Google provided and shoot yourself in the foot with writing boilerplate code.


This post is more like a rant, hope your ego isn't hurt so far and you learnt to make the code a better place.

<img src="/assets/img/arch_components_mistakes/2.jpg" class="center">

Thanks for reading and not getting hurt.