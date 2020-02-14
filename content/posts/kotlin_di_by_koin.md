---
title: "Koinを用いてDIする"
date: 2020-02-14T20:46:35+09:00
draft: false
---
引数ありのViewModelをDIする際にDaggerを使用すると思った以上に手がかかるのでKoinを使用してみました。  
実現したかったことは次のとおりです。

```kotlin
// MainFragment.kt
class MainFragment() : Fragment() {
    private val mainFragmentViewModel // <- ここをDIしたい
}
```
```kotlin
// MainFragmentViewModel.kt
class MainFragmentViewModel(private val repository: MyRepository) : ViewModel() {
}
```
```kotlin
// MyRepository.kt (singleton)
class MyRepository(context: Context) {
    fun getInstance(context: Context) {
        // ....
    }
}
```

DaggerだとAssistedInjectを使って割とめんどくさいことをしないといけないようですが、Koinだとかなり簡潔に記述することができました。

```kotlin
// Application.kt
class Application : Application() {
    private val module = module {
        // singletonはsingleで宣言する
        single<MyRepository> { MyRepository.getInstance(applicationContext)}

        // ViewModelはviewModelで宣言する
        // 引数の位置にget()を書いておくことで一致するものを渡してくれる
        viewModel { MainFragmentViewModel(get()) }
    }

    override fun onCreate() {
        super.onCreate()
        //startKoinを用いてKoinを開始することができる
        startKoin {
            androidContext(applicationContext)
            modules(module)
        }
    }
}
```

```kotlin
class MainFragment() : Fragment() {
     // by viewModel() を用いることで適するものをDIすることができる
     private val mainFragmentViewModel: MainFragmentViewModel by viewModel()
}
```
MainFragmentViewModelとMyRepositoryは何も変更する必要がありませんでした。