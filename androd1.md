ラクマでAndroidエンジニアをしているyoshidaです。
ラクマでは絶賛Androidエンジニアを募集中のため、赤裸々にラクマのAnroidの開発環境として、パッケージ構成や採用しているアーキテクチャについて説明し、ラクマでのUnitテストはどのように実装しているか説明します。

# Androidの開発環境

Androidの開発環境としては、IDEはAndroid Studioを使用しています。
プログラミング言語はKotlin, Javaが混在しており、Kotlinが7割、Java3割といったところです。
非同期処理はRxJavaを使用おり、DIのライブラリは使用していません。
モジュールはAppモジュールとnetworkモジュールに分かれている構成になっています。
APIにアクセスしてレスポンスを返す処理をnetworkモジュールにまとめてあり、それ以外はAppモジュールにあります。featureごとにモジュールをわける予定はありますが、現状そうなってはいません。
また、ラクマで開発に利用しているサービスは以下のとおりです（2021年6月現在)
|サービス||
|--|--|
|CI| Bitrise|
|CD| Deploygate|
|コード管理| Github (private)|

slackからコマンドを送り、BitriseのWorkflowを実行できるようになっており、
コマンドひとつでリリースビルドを実行し、PlayStoreへの配備までを行う環境が整っています。
コードをpushするごとにUnitTestが走るようになっており、リファクタリングで実装ミスに気づけるようになっています。

# アーキテクチャ

ラクマではGoogleがMVVMを推奨アーキテクチャとしていることもあり、MVVMアーキテクチャを採用しています。
https://developer.android.com/jetpack/guide#recommended-app-arch

プレゼンテーション層とデータ層の２層構造にしています。
ざっとそれぞれの層の役割を説明します。

![スクリーンショット 2021-06-30 1.35.35.png](https://fablic.qiita.com/files/f42a43f2-13ec-5da8-5f5c-f886a7e1c7f4.png)

## データ層

ラクマではデータベースを用いていません。全てサーバからのデータを取得する構成になっています。
リポジトリの役割としてはサーバから取得したデータをViewModelへ返すことです。
その際、convertorを用いてプレゼンテーション用のmodelへの変換といったことはしていません。

## プレゼンテーション層

データ層から受け取ったデータをViewModelでobserveしてActivity/Fragmentに表示します。

# パッケージ構成

ラクマのパッケージ構成は以下のようになっています。
各クラスには基本的にサフィックスをつけるルールになっています。
uiパッケージには、サブパッケージに機能のパッケージが置かれ、そのパッケージにはActivity, Fragment, ViewModel, Adapterのクラスを入れるようにしています。

```
├── camerax
├── ds
├── events
├── exception
├── extension
├── fcm
├── listener
├── model
├── network
├── provider
├── receiver
├── repository
├── service
├── tracking
├── ui
├── utils
└── view
```

今回ラクマの開発環境の説明として、ブランド名をサーバから取得し、取得している間はプログレスを表示し、取得が完了したらプログレスを消した後にRecyclerViewに表示する機能があったとして、ViewModelの実装や、そのViewModelのUnitTestをどう書いているのかサンプルコードを用いて説明したいと思います。
サンプルコードのパッケージ構成は以下のようになります。

```
├── java
│   └── com
│       └── example
│           └── samplerakuma
│               ├── MainActivity.kt
│               ├── repository
│               │   └── BrandRepository.kt
│               └── ui
│                   └── brand
│                       ├── BrandAdapter.kt
│                       ├── BrandFragment.kt
│                       └── BrandViewModel.kt
└── java
    └── com
        └── example
            └── samplerakuma
                └── network
                    ├── response
                    │   └── Brands.kt
                    └── service
                        └── RakumaService.kt

```

# ViewMoel

```kotlin

class BrandViewModel(
    application: Application,
    private val repository: BrandRepository  // コンストラクタインジェクションさせます
) : AndroidViewModel(application) {

  　private val disposables = CompositeDisposable()
    val isLoading = ObservableField(false)
    private var brandsAdapterItems = ObservableArrayList<Brands>()

    fun fetchBrands() {
        disposables.add(repository.fetchBrands()
            .observeOn(AndroidSchedulers.mainThread())
            .doOnSubscribe {
                isLoading.set(true)  // ローディング開始
            }
            .doFinally {
                isLoading.set(false) // ローディング終了
            }.subscribe({ brands ->
                brandsAdapterItems.addAll(brands)　// ObservableArrayListにDataをセットする
            }, {
                Log.e("error", it.message!!)
            })
        )
    }

    fun diposables() {
        disposables.clear()
    }

    class Factory(private val application: Application) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T {
            return BrandViewModel(
                application,
                BrandRepository.getInstance()
            ) as T
        }
    }
}
```

# UnitTest

ラクマではViewModelのUnitTestを書くようにしています。
UnitTestで使用しているツールとしてMockライブラリにMocitoを使用しており、拡張ツールとして mockito-kotlinを使用しています。
ライブラリの使用方法などは、ここでは説明しないため、公式サイトを確認してください。
例えば、ブランド名を取得して表示する場合のラクマで実装されるUnitTestは以下のようになります。
（サンプルのため、テストケースを絞っています)
- ローディングを表示しているか
- ブランド名を取得しているか
- ブランド名を取得した後、ローディングを非表示にしているか
以下にUnitTestのサンプルコードを記します。

``` kotlin
@RunWith(AndroidJUnit4::class)
@LooperMode(LooperMode.Mode.LEGACY)
class BrandViewModelTest {

    @Mock private lateinit var brandRepository: BrandRepository

    @Before
    fun setup() {
        MockitoAnnotations.initMocks(this)
    }

    @Test
    fun testFetchBrandsSuccess() {

        val data = listOf<Brands>(Brands("test"))
        val scheduler = TestScheduler()
        whenever(brandRepository.fetchBrands()).thenReturn(Single.just(data).subscribeOn(scheduler))

        val viewModel = createViewModel()

        // ①BrandName取得処理を起動します
        viewModel.fetchBrands()

        // ②loadingのフラグが立っていることを確認します
        assertThat(viewModel.isLoading.get())
            .`as`("ローディング中")
            .isEqualTo(true)

        scheduler.triggerActions()

        // ③loadingのフラグが落ちていることを確認します
        assertThat(viewModel.isLoading.get())
            .`as`("ローディング終了")
            .isEqualTo(false)

        // ④repositoryから取得したデータがViewModelで取得できていることを確認します
        val adapterItems = viewModel.brandsAdapterItems.stream().collect(Collectors.toList())
        val brands = adapterItems.map { Brands(it.name) }
        assertThat(brands)
            .`as`("取得したブランド名")
            .isEqualTo(data)
    }


    private fun createViewModel(): BrandViewModel {
        return BrandViewModel(
            ApplicationProvider.getApplicationContext(),
            brandRepository
        )
    }

}
```

testFetchBrandsSuccessのテストではこのような形でローディングを行った後にデータを取得し、最後にローディング処理を終了させることを確認できました。

# リファクタリング

上記のような開発環境でラクマを開発しており、今後のリファクタリングの方向について最後に書きたいと思います。

## 課題

現状ではkotlin化は7割程度になっており、まだJavaのコードが残っています。
また、古いコードが残っており、MVVM化も完全には済んでいない状況です。
そのため、kotlin化とMVVM化をリファクタリングしていくことになると思います。
その他にはcoroutineへの置き換えや、マルチモジュール化、Jetpack Composeへの置き換えをリファクタリングしていくことになると思います。
ラクマでは、User Firstをコア・バリューの一つに掲げ、一緒にアプリ開発をしてくれるメンバーを募集しています。
