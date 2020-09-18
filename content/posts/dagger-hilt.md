---
title: "Dagger Hilt"
date: 2020-09-18T09:16:33+09:00
draft: false
---

DIコンテナをKoinからDagger Hiltに変更しました。その際のTipsを書き残しておきます。

### 引数ありのViewModelをFragmentへInjectする

- 注入されるFragmentに`@AndroidEntryPoint`のアノテーションを付与する。<br>これにより、注入されるFragmentを指定することができます。<br>
あとはandroidxのFragment拡張を使用して`by viewModels<>()`で受け取りましょう。
```kotlin
// MainFragment.kt
@AndroidEntryPoint
class MainFragment : Fragment() {
    private val viewModel by viewModels<MainViewModel>()
}
```

- 注入したいViewModelに`@ViewModelInject constructor()`を付与する。<br>
これにより注入したいViewModelをオブジェクトグラフに登録することができます。
```kotlin
// MainViewModel.kt
class MainViewModel @ViewModelInject constructor(
    private val repository: Repository
) : ViewModel {
    ...
}
```
- FragmentやViewModelを継承していない、自作のクラスは通常のDagger通り`@Inject constructor()`の付与でオブジェクトグラフに登録します。
```kotlin
// Repository.kt
class Repository @Inject constructor() {
    ...
}
```

- 最後に忘れがちですが、Fragmentを描画しているActivityにも`@AndroidEntryPoint`を付与してあげてください。
```kotlin
// MainActivity.kt
@AndroidEntryPoint
class MainActivity : Activity() {
    ...
}
```

以上がFragmentに引数ありのViewModelを注入する一連の手順となります。
通常のDaggerと違い、ViewModel内にFactoryを書いたり、Moduleに`@ContributesAndroidInjector`を用いてゴニョゴニョする必要がなくなっており、
簡潔に書くことができると思います。

### RoomDatabaseのオブジェクトグラフへの登録
- RoomDatabaseをオブジェクトグラフに登録するためにModuleを作成します。
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object MyDatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(application: Application): MySongDatabase {
        return Room.databaseBuilder(application, MyDatabase::class.java, "database-name")
            .build()
    }

    @Provides
    @Singleton
    fun provideMyDatabaseDao(database: MyDatabase): MyDatabaseDao {
        return database.myDatabaseDao()
    }
}
```

こんな感じになります。Dagger Hiltではモジュールの宣言時に`@InstallIn`アノテーションをつける必要があります。<br>
`@InstallIn(コンポーネント名)`を指定することで、各モジュールが使用されるまたはインストールされるコンポーネントを指定します。<br>
ex) Fragmentに注入したい場合: `@IntallIn(FragmentComponent::class)`<br>

今回はSingletonなRepositoryに注入したいので`@InstallIn(SingletonComponent::class)`を付与しています。


ぜひ実際の[使用例](https://github.com/Index197511/MySongBank "MySongBank")も見ていただけると幸いです。
