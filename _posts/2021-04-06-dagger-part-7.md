---
title: Dagger2 is hard, but it can be easy, part 7
date: 2021-04-06 22:55:00 +0100
categories: [Dagger]
tags: [Dagger, Kotlin, Android]
---

<img src="/assets/img/dagger/dagger_title.jpeg" class="center">

In the [previous](/posts/dagger-part-6/) post we've encountered when's the right place to inject, named and qualifiers.

In this post we'll learn about multi bindings and their power.

They're your best buddies in the Dagger world.

There's only two of them `Set` and `Map` multi bindings and as you might've guessed
they're indeed collections.

Go ahead and include 
```groovy
implementation "com.squareup.retrofit2:converter-moshi:$retrofit"
implementation "com.squareup.retrofit2:converter-gson:$retrofit"

implementation 'com.squareup.okhttp3:logging-interceptor:4.9.1'

implementation "com.squareup.moshi:moshi-kotlin:$moshi"
kapt "com.squareup.moshi:moshi-kotlin-codegen:$moshi"
```
*Retrofit's version at the moment of writing this blog is "2.9.0" and "moshi" is 1.12.0*

Go ahead and create a Retrofit module and Interceptors module
```kotlin
@Module
object RetrofitModule {
    
}
```

```kotlin
@Module
object InterceptorsModule{

}
```
don't forget to include the modules in the singleton component

```kotlin
@Component(modules = [RetrofitModule::class, InterceptorsModule::class]) //<-this here
@Singleton
interface SingletonComponent
.
.
.
```

Create an interface called `TestApi`
```kotlin
interface TestApi {S


    @GET("posts")
    suspend fun getPostsAdapter(): Response<List<TestModel>>

    companion object {
        const val BASE_URL = "https://jsonplaceholder.typicode.com/"
    }
}
```

and a `TestModel` which will get the response from the url

```kotlin
import com.squareup.moshi.Json
import com.squareup.moshi.JsonClass

@JsonClass(generateAdapter = true)
data class TestModel(
    @Json(name = "body")
    val body: String,
    @Json(name = "id")
    val id: Int,
    @Json(name = "title")
    val title: String,
    @Json(name = "userId")
    val userId: Int
)
```
*we're using moshi codegen for the json transformation*

Let's create some `Intereceptor`s
```kotlin
class ContentTypeInterceptor(private val contentType: String = "application/json") : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response = chain
            .proceed(with(chain.request().newBuilder()) {
                header("Content-Type", contentType).build()
            })
}
```

```kotlin
class EmptyInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response = chain.proceed(chain.request())
}
```

A bootleg retry interceptor
```kotlin
class RetryRequestInterceptor : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {

        val request = chain.request()
        var response: Response = chain.proceed(request)
        var tryCount = 0

        while (!response.isSuccessful && tryCount < 3) {
            tryCount++
            response = chain.proceed(request)
        }

        return response
    }
}
```

and

```kotlin
class UnauthorizedInterceptor(private val customMessage: String? = null) : Authenticator {

    companion object {
        private const val unAuthorized = 401
    }

    override fun authenticate(route: Route?, response: Response): Request {

        if (!response.isSuccessful && response.code == unAuthorized)
            throw UnauthorizedException(customMessage)

        return response.request
    }

    class UnauthorizedException(private val customMessage: String?) : IOException() {
        override val message: String
            get() = customMessage ?: "Un-authorized, please check credentials or re-login/authorize"
    }
}
```


Let's populate our `Retrofit` module
```kotlin
@Module
object RetrofitModule {


    @Singleton
    @Provides
    fun moshiConverterFactory() = MoshiConverterFactory.create()


    @Singleton
    @Provides
    fun okHttpClientConfiguration(
        interceptors: Set<@JvmSuppressWildcards Interceptor>
    ): OkHttpClient {
        val timeout = 10L //even this can be provided
        val timeUnit = TimeUnit.SECONDS //even this too, but for the sake of keeping this short we  aren't
        val client = OkHttpClient().newBuilder()
            .apply {
                connectTimeout(timeout, timeUnit)
                callTimeout(timeout, timeUnit)
                readTimeout(timeout, timeUnit)
                writeTimeout(timeout, timeUnit)
            }
        interceptors.forEach {
            client.addInterceptor(it)
        }
        return client.build()
    }

    @Provides
    @Singleton
    fun retrofitClient(
        moshiConverterFactory: MoshiConverterFactory,
        okHttpClient: OkHttpClient
    ) = Retrofit.Builder()
        .addConverterFactory(moshiConverterFactory)
        .client(okHttpClient)
        .baseUrl(TestApi.BASE_URL)
        .build().create<TestApi>()
}
```

and our `Interceptors` module

```kotlin
@Module
object InterceptorsModule {

    @Provides
    @IntoSet
    @Singleton
    fun contentTypeInterceptor() = ContentTypeInterceptor()

    @Provides
    @IntoSet
    @Singleton
    fun unauthorizedInterceptor() = UnauthorizedInterceptor()

    @Provides
    @IntoSet
    @Singleton
    fun emptyInterceptor() = EmptyInterceptor()

    @Provides
    @IntoSet
    @Singleton
    fun retryInterceptor() = RetryRequestInterceptor()

    @Provides
    @Singleton
    @IntoSet
    //this one we don't create don't be confused
    fun httpLoggingInterceptor() :Interceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY else HttpLoggingInterceptor.Level.NONE
    }
}
```

## Set
--- 

The aforementioned example is just creating a Retrofit Client that's all, the new comer you're seeing here is 
`@IntoSet`

Once you get creating a multi module project, the `Set` multibindings (@Intoset) is really coming in handy, especially when decoupling of the implementation by the type you're gonna use.

It can scale from so called `Initializers`(that hide away the initialization logic),
`Adapters` that bind some data and in our example `Interceptor`s and it doesn't stop here...

What `@IntoSet` does is, it creates a `Set` for you of the type you're providing, in this example you can *also* levarage the set for providing the `MoshiConverterFactory` with some `Adapters` of your own that can lie in a separate module called `:moshi-adapters` or whatever you named them, but for the sake of this example let's keep it short, hopefully by now you've realized the power.

The part that you should know is that if you've provided one thing into the **graph** you can use it as a parameter into a function.

As you see our Client configuration function that provides an `OkHttpClient`
```kotlin
fun okHttpClientConfiguration(
        interceptors: Set<@JvmSuppressWildcards Interceptor>
    )
```
accepts set of interceptors from other Dagger module (`InterceptorsModule`) but they're brought together with 
`@Component(modules = [RetrofitModule::class, InterceptorsModule::class])`, also there's this `@JvmSuppressWildcards`, the [explanation](https://kotlinlang.org/docs/java-to-kotlin-interop.html#variant-generics) of why's this happening can be a bit confusing, let's keep it simple:

1. We're injecting `Set<Interceptor>`, what Kotlin does is it generates Java code
2. The java code looks like `Set<? extends Interceptor>` where as you know `?` is a [wild card](https://docs.oracle.com/javase/tutorial/extra/generics/wildcards.html), Dagger gets confused
3. If we add `@JvmSuppressWildcards` that generated code gets transformed into `Set<Interceptor>` and Dagger isn't confused anymore into the Kotlin world.

Now this approach of Set gets to keep your `Interceptor`s into a separate module and this scales really good because all you have to do is include a parameter of Set in your function that provides a dependency maybe coming from a module that you once wrote and reused at multiple times with other modules that our `IntereceptorsModule`.


## Map
---

`@IntoMap` binds a provided dependency into a Map collection using a key, Dagger comes with:
1. `@StringKey` annotation which binds a dependency to a `String` key
2. `@ClassKey` which binds it to a class
3. `@MapKey` for keys that are of type enums or parameterized classes (your own)

Go ahead and create a `TestViewModel`

```kotlin
class TestViewModel @Inject constructor(private val testApi: TestApi) : ViewModel() {

    private val listData: MutableStateFlow<SimpleResult<List<TestModel>>> = MutableStateFlow(SimpleResult.Loading)
    val list = listData.asStateFlow()

    init {
        viewModelScope.launch(Dispatchers.IO) {
            listData.value = try {
                transformPosts(testApi.getPostsAdapter())
            } catch (throwable: Throwable) {
                SimpleResult.Exception(throwable)
            }
        }
    }

    private fun transformPosts(response: Response<List<TestModel>>): SimpleResult<List<TestModel>> =
        if (response.isSuccessful) {
            SimpleResult.Success(response.body() ?: emptyList())
        } else {
            SimpleResult.ApiError(response.code(), response.errorBody())
        }


    sealed class SimpleResult<out T> {
        data class Exception(val throwable: Throwable) : SimpleResult<Nothing>()
        data class ApiError(val responseCode: Int, val errorBody: ResponseBody?) : SimpleResult<Nothing>()
        data class Success<T>(val data: T) : SimpleResult<T>()
        object Loading : SimpleResult<Nothing>()
    }
}
```

We're including our `testApi : TestApi` using constructor injection and just calling an api to get the result, don't forget to add the `Internet` permission
`<uses-permission android:name="android.permission.INTERNET"/>` inside your manifest.

Before you move onto the next part add 
```groovy
implementation "androidx.lifecycle:lifecycle-runtime-ktx:$lifecycle"
```
where "lifecycle" is '2.4.0-alpha01' or better, to add a way to collect flows safely without leaking resources by using `addRepeatingJob`.

For the sake of this demonstration into your `activity_main.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
    
</androidx.constraintlayout.widget.ConstraintLayout>
```

inside `MainActivity`

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

        if (savedInstanceState == null) {
            supportFragmentManager
                .beginTransaction()
                .add(R.id.fragment_container, HomeFragment(), "HomeFragment")
                .commit()
        }
    }

}
```

Inside your `HomeFragment`
```kotlin
class HomeFragment : Fragment(R.layout.home_fragment) {

    override fun onAttach(context: Context) {
        injector {
            fragmentComponentFactory().create().inject(this@HomeFragment)
        }
        super.onAttach(context)
    }

    private val testViewModel by viewModels<TestViewModel>()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.addRepeatingJob(Lifecycle.State.STARTED){
            testViewModel.list.collect {
                handleAPICall(it)
            }
        }
    }

    private fun handleAPICall(simpleResult: TestViewModel.SimpleResult<List<TestModel>>) {
        when(simpleResult){
            is TestViewModel.SimpleResult.ApiError -> {
                Log.d("SimpleResult.ApiError", "Response code ${simpleResult.responseCode}")
            }
            is TestViewModel.SimpleResult.Exception -> {
                when(simpleResult.throwable){
                   is UnauthorizedInterceptor.UnauthorizedException->{
                       Log.d("Exception", "Log off")
                   }
                }
            }
            TestViewModel.SimpleResult.Loading -> {
                Log.d("SimpleResult.Loading", "SHOW SPINNER")
            }
            is TestViewModel.SimpleResult.Success -> {
                val successData = simpleResult.data
                Log.d("SimpleResult.Success", successData.toString())
            }
        }
    }
}
```

now you're ready to run the application, once you do that 
```java
java.lang.RuntimeException: Cannot create an instance of class com.example.test.TestViewModel
```
You might ask yourself why, what did I do wrong?
Nothing, it's Android again complicating things...

The ViewModel can't be created because Dagger doesn't know how to provide the dependency to that particular ViewModel of ours, as you know that ViewModels are created by a `ViewModelProvider.Factory` we need to tell Dagger about that somehow and also provide a way for it to know how to create our `TestViewModel`, which is where `Map` multi bindings shine.

First, we need a key that will provide us a class that inherits from `ViewModel`
```kotlin
@MustBeDocumented
@Target(
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY_GETTER,
    AnnotationTarget.PROPERTY_SETTER
)
@MapKey
annotation class ViewModelKey(val kotlinClassKey: KClass<out ViewModel>)
```
as you can see as we've mentioned before that class has a `@MapKey` which means it's something specialized for our use case.
The `@Target` is specialized where do we apply that annotation to, in our case means that we need to use `@Provides` on a function in a module, the key is of a `KClass` that has a `out` which is a covariant that can accept every kotlin class name that inherits from `ViewModel` (we're restricting this to ViewModel only, instead we could've just used Any, but we don't need that because we might have another key of type Any and confusion... confusion).

Secondly we need a `ViewModel.Factory`
```kotlin
@Suppress("UNCHECKED_CAST")
@Singleton
class DaggerViewModelFactory @Inject constructor(
    private val viewModelMap: Map<Class<out ViewModel>, @JvmSuppressWildcards Provider<ViewModel>>
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        val creator = viewModelMap[modelClass] ?: viewModelMap.entries.firstOrNull {
            modelClass.isAssignableFrom(it.key)
        }?.value ?: throw IllegalArgumentException("Unable to locate class $modelClass")
        return creator.get() as T
    }
}
```
Thirdly we need a module
```kotlin
@Module
abstract class ViewModelModule {
    @Binds
    @IntoMap
    @ViewModelKey(TestViewModel::class)
    abstract fun bindTestViewModel(testViewModel: TestViewModel) : ViewModel

    @Binds
    abstract fun bindViewModelFactory(factory: DaggerViewModelFactory) : ViewModelProvider.Factory
}
```
You might ask yourself why an abstract class, because the abstract class only accepts an implementation of dependencies which we did already in our case this is `TestViewModel` and we return it as a `: ViewModel` because our `@IntoMap` expects a type of `Provider<ViewModel>>` and of course we tell Dagger that the map it gives to our `DaggerViewModelFactory` has to have `TestViewModel::class` so that it can create our `TestViewModel` later on.

For example if you have a class
```kotlin
class Car () : Vehicle
```
and 
```kotlin
interface Vehicle
```
inside your abstract module you can do
```kotlin
@Module
abstract class CarsModule {
    @Binds
    abstract fun bindCar(car: Car) : Vehicle
}
```
`@Binds` works nearly the same as `@Provides` which can only provide dependencies of abstract functions that you already have an implementation for and the way it creates them is way more effecient than `@Provides`, which means try to use `@Binds` as much as you can, clean architecture picture in 3...2...1...
<img src="/assets/img/dagger/7/clean_code.jpeg" class="center">


You might not be familiar with why `Provider` is used in this case, sometimes you need multiple instances of the same type to be returned instead of just injecting a single value, meaning that you can have that `TestViewModel` inside a `HomeFragment`, `LoginFragment` and `YourNameHereFragment` etc... etc..
As we've talked into [part 1](/posts/dagger-part-1/), Dagger is really one interface and that's `Provider` which has only one function called `get()` which gets the dependency.

let's go step by step:
1. Every view model factory inherits from `ViewModelProvider.Factory` which has only one `create()` method and that method has a parameter of a `modelClass: Class<T>` which is our custom map key
2. Since we have the key from step one, now we need to `@Inject` the map where we have the Key that's of our class for the ViewModel and a provider which'll give us an instance of that ViewModel
3. We access the `viewModelMap[modelClass]` that means we might not have a value for that key, we traverse the map and we check if `modelClass.isAssignableFrom(it.key)`, meaning that they're either the same types and we just access the value using `.value`
4. `creator` is of a type `Provider<ViewModule>` which returned our `ViewModel` and all we need to do is call `creator.get()` as we've discussed that Providers have only one function
5. We cast it as `T` because the `create()` function expects a typed parameter

With this we've created a generic `ViewModelProvider.Factory` that we can use now to create our `TestViewModel`

Now we need to head out for our `SingletonComponent` and include the `ViewModelModule`

```kotlin
@Component(modules = [RetrofitModule::class, InterceptorsModule::class, ViewModelModule::class])
@Singleton
interface SingletonComponent {
    .
    .
    .

}
```

The only thing we need to change now in our `HomeFragment` is 
```kotlin
@Inject
    lateinit var daggerViewModelFactory: ViewModelProvider.Factory

    private val testViewModel by viewModels<TestViewModel>(){
        daggerViewModelFactory
    }
```
which means now dagger knows how to create our ViewModel using the factory we created leveraging the `by viewModels` delegate that expects a creator of a type factory.

Our `HomeFragment` can look like 

```kotlin
class HomeFragment : Fragment(R.layout.home_fragment) {

    override fun onAttach(context: Context) {
        injector {
            fragmentComponentFactory().create().inject(this@HomeFragment)
        }
        super.onAttach(context)
    }

    @Inject
    lateinit var daggerViewModelFactory: ViewModelProvider.Factory

    private val testViewModel by viewModels<TestViewModel>(){
        daggerViewModelFactory
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.addRepeatingJob(Lifecycle.State.STARTED){
            testViewModel.list.collect {
                handleAPICall(it)
            }
        }
    }

    private fun handleAPICall(simpleResult: TestViewModel.SimpleResult<List<TestModel>>) {
        when(simpleResult){
            is TestViewModel.SimpleResult.ApiError -> {
                Log.d("SimpleResult.ApiError", "Response code ${simpleResult.responseCode}")
            }
            is TestViewModel.SimpleResult.Exception -> {
                when(simpleResult.throwable){
                   is UnauthorizedInterceptor.UnauthorizedException->{
                       Log.d("Exception", "Log off")
                   }
                }
            }
            TestViewModel.SimpleResult.Loading -> {
                Log.d("SimpleResult.Loading", "SHOW SPINNER")
            }
            is TestViewModel.SimpleResult.Success -> {
                val successData = simpleResult.data
                Log.d("SimpleResult.Success", successData.toString())
            }
        }
    }
}
```
Run it and it works.

You might think there's nothing wrong with this but there actually is, everything is scoped to `@Singleton`, also getting a `SavedStateHandle` requires to use `AssistedInject` that requires a lot of changing which won't matter once we start using *Hilt* from the next part as this is sufficient for today and this blog post is longer than I anticipated also every bit of this informations needs some time for you to stomach it.

From the next blog post, we're jumping onto the **Hilt** train and see where it goes, also **Hilt** in a nutshell
<img src="/assets/img/dagger/7/hilt.jpeg" class="center">

Thank you again for your wholehearted attention and stay tuned for the Hilt blog series.