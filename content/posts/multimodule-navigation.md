---
title: "Multimodule Navigation"
date: 2020-09-23T08:35:00+09:00
draft: false
---

既存のアプリケーションをマルチモジュールにする際にNavigationComponentとFragmentを主体としたNavigationを定義したのでその際のTipsについて書き残しておきます。

### モジュールの構成について
モジュールの構成は次のとおりです
{{< figure src="/img/module.png">}}
- AppModule: DaggerHiltのエントリポイント(@HiltAndroidApp)を担っています。またRouterモジュールを作成し配布しています。
- RouterModule: マルチモジュール間でのNavigationComponentを用いたFragment間繊維を担っています。
- FeatureModule: 各feature毎のモジュールを集めています。主に画面ごとに一つのfeatureを切っています。
- DataModule: ローカル、リモートのリソースへのRepositoryを通したアクセスを公開しています。
- ModelModule: アプリケーション内で使用するモデルを定義しています。

### マルチモジュール間でのFragmentを用いた画面遷移について
マルチモジュール構成において、画面遷移は複雑なものです。
マルチモジュール構成において、自分と同階層または上階層のモジュールへのアクセスは不可能なため、シングルモジュール構成におけるNavigationGraphを用いた画面遷移は不可能となります。
そのため、画面遷移を担うRouterモジュールを別の階層に作成して、そこに画面遷移を担当してもらうこととします。

1: Featureモジュール内の画面遷移をサポートしたいFeature内に画面遷移のためのInterfaceを定義する。
```kotlin
// FeatureARouter.kt in FeatureModule/FeatureA
interface FeatureARouter {
    fun navToFeatureB()
}
```

```kotlin
// FeatureBRouter.kt in FeatureModule/FeatureB
interface FeatureBRouter {
    fun navToFeatureA()
}
```

2: Routerモジュール内に画面遷移のためのNavigationGraphを作成し、Appモジュール内のactivity_main.xmlから呼び出す。
```xml
<!-- nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    app:startDestination="@id/featureAFragment"
    tools:ignore="UnusedNavigation">

    <fragment
        android:id="@+id/featureAFragment"
        android:name="me.example.feature.featureA.FeatureAFragment"
        tools:layout="@layout/featureA_fragment">
        <action
            android:id="@+id/action_featureA_to_featureB"
            app:destination="@id/featureBFragment" />
    </fragment>

    <fragment
        android:id="@+id/featureBFragment"
        android:name="me.example.feature.featureB.FeatureBFragment"
        android:label="featureB_fragment"
        tools:layout="@layout/featureB_fragment">

        <action
            android:id="@+id/action_featureB_to_featureA"
            app:destination="@id/featureAFragment" />
    </fragment>
</navigation>
```

```xml
<!-- activity_main.xml -->
<fragment
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/nav_graph"/>
```

3: Routerモジュール内に1で定義したInterfaceを定義したクラスを作成する。
```kotlin
// Router.kt in RouterModule
class FeatureAImpl @Inject constructor(
    private val controller: NavController
) : FeatureARouter {
    override fun navToFeatureB() {
        controller.navigate(R.id.action_featureA_to_featureB)
    }
}

class FeatureBImpl @Inject constructor(
    private val controller: NavController
) : FeatureBRouter {
    override fun navToFeatureA() {
        controller.navigate(R.id.action_featureB_to_featureA)
    }
}
```

4: Appモジュール内にDIの為のモジュールを作成し、Routerモジュールを配布する。
```kotlin
// RouterModule.kt in AppModule
@Module
@InstallIn(FragmentComponent::class)
class RouterModule {
    @Provides
    fun provideNavController(fragment: Fragment): NavController {
        return fragment.findNavController()
    }

    @Provides
    fun provideFeatureARouter(impl: FeatureARouterImpl): FeatureARouter = impl

    @Provides
    fun provideFeatureBRouter(impl: FeatureBRouterImpl): FeatureBRouter = impl
}
```

5: 使用するFeatureモジュール内の各フラグメントで受け取る。
```kotlin
// FeatureAFragment.kt
@AndroidEntryPoint
class FeatureAFragment : Fragment() {
    @Inject lateinit var router: FeatureARouter
    
    // 画面遷移
    router.navToFeatureB()
}
```
```kotlin
// FeatureBFragment.kt
@AndroidEntryPoint
class FeatureBFragment : Fragment() {
    @Inject lateinit var router: FeatureBRouter
}
```

こんな感じの流れでマルチモジュール間での画面遷移を実装することができます。

### Extra: Safeargsについて
上記の方法だと、ビルド時にsafeArgsのためのクラスがRouterモジュール内にしか自動生成されず、Featureモジュール内での受取のためのクラスが生成されません。
そのため、safeArgsをサポートしたい場合は受取側のFeatureモジュール内にsafeargsを用いたnav_graphを追加で作成する必要があります。
[使用例](https://github.com/Index197511/MySongBank "MySongBank")に実際に使用している例があるので是非参考にしてみてください。