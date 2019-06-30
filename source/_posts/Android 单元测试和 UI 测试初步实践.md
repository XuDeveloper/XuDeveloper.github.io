---
title: Android 单元测试和 UI 测试初步实践
tags: Android实践
categories: Android
abbrlink: 1e6d7596
date: 2019-06-30 16:47:42
---

#### **本文预计阅读时间为15-20分钟**

### Android 测试简介

对于大多数 Android 商业项目，基本都是处于高速迭代的开发阶段，这个阶段不仅仅是对项目的开发效率，也对项目的产品质量提出了更高的要求。

<!--more-->

通常大型项目都是通过黑盒测试等方式来提供质量相关的保障，但同时笔者认为也需要 Android 端的单元测试以及能自动在 Android 平台上运行的 UI 测试，这几种测试有以下几个优势：
- 更早发现代码中存在的 bug 等问题，提前 fix bug；
- 更好地设计：在进行项目重构的时候，保证重构的新代码能正确运行，这样就能在业务不断迭代的同时，更好地保障产品质量。

### Android 测试代码位置

在 Android Studio 中新建新的项目时，它已自动为两种测试类型创建了对应的代码目录：

- 单元测试用例：位于 module-name/src/test/java 目录下，只依赖 JVM 环境而不需要 Android 环境
- InstrumentTest 测试/ UI 测试用例：位于 module-name/src/androidTest/java 目录下，在 Android 环境下才能运行

接下来，笔者将尝试为自己的项目（基于 MVP 架构开发）补充相应的单元测试用例和 UI 测试用例，来初步实践下如何在 Android 平台编写和运行相关的测试用例。

### Android 单元测试实践

#### 创建新用例

如果需要编写一个新的本地单元测试用例，只需打开你想测试的 java 代码文件，然后点击类名 -- ⇧⌘T（Windows：Ctrl+Shift+T）-- 选择要生成的方法 -- 选择 test 文件夹，对应于本地单元测试 -- 完成。

#### 增加依赖库

需要 JUnit 和 Mockito 框架支持，所以在 build.gradle 中增加：
```groovy
testImplementation "junit:junit:4.12"
testImplementation "org.mockito:mockito-core:2.7.1"
```

#### 编写测试代码

一般来说，编写一段测试代码需要三个步骤：
- 环境初始化
- 执行操作
- 验证结果正确性

笔者主要测试的是 MVP 架构中 P 层的代码。在笔者的项目中，P 层是通过 Dagger2 机制，注入一个 DataManager，也就是数据获取源。同时也需要一个 V 层的代理，这样在 P 层通过数据源获取数据之后，就能将数据交给 V 层，由 V 层去展示。

代码调用大致逻辑如下：
```java
mPresenter = new NewsPresenter(mDataManager);
mPresenter.getNews();
mPresenter.attach(mView);
--> mView.showProgress(); // 在数据未加载完前加载进度条
--> mView.showNews(news);
--> mView.hideProgress(); // 在数据加载完后隐藏进度条
```

对应着，实际编写 P 层的单元测试用例的时候，并不需要一个真实的数据源，只需要通过 Mockito 框架，mock 出一个测试用的 DataManager 和 V 层代理。

对应着 Presenter 类，新创建的测试代码如下：

```java
/**
 * Created by Xu on 2019/04/05.
 *
 * @author Xu
 */
public class NewsPresenterTest {
    @ClassRule
    public static final RxImmediateSchedulerRule schedulers = new RxImmediateSchedulerRule();

    @Mock
    private NewsContract.View view;
    @Mock
    protected DataManager mMockDataManager;
    private NewsPresenter newsPresenter;
    
    @Before
    public void setUp() {
        MockitoAnnotations.initMocks(this);
        newsPresenter = new NewsPresenter(mMockDataManager);
        newsPresenter.attach(view);
    }
    
    @Test
    public void getNewsAndLoadIntoView() {
        TencentNewsResultBean resultBean = new TencentNewsResultBean();
        resultBean.setData(new ArrayList<>());
        when(mMockDataManager.getNews()).thenReturn(Flowable.just(resultBean));

        newsPresenter.getNews();

        // 测试model是否有获取数据
        verify(mMockDataManager).getNews();

        // 测试view是否调用相应接口
        verify(view).showProgress();
        verify(view).showNews(anyList());
        verify(view).hideProgress();
    }

    @After
    public void tearDown() {
        newsPresenter.detach();
    }
}
```
在其中：
1. 在代码开头，声明了一个 @ClassRule；

什么是 @ClassRule 呢？它跟 @Rule 注解几乎相同，可以在所有类方法开始前进行一些相关的初始化调用操作。使用这个注解，可以在执行测试用例的时候加入特有的操作，而不影响原有用例代码，有效减少耦合程度。

这里主要是因为项目中使用了 RxJava2，而 RxJava 是需要 Android 环境支持的，如果直接运行 JUnit 测试用例会报错，所以在此处增加了一个 @ClassRule，具体可参考
https://stackoverflow.com/questions/41121778/junit-rule-and-classrule

```java
/**
 * Created by Xu on 2019/04/05.
 *
 * @author Xu
 */
public class RxImmediateSchedulerRule implements TestRule {
    private Scheduler immediate = new Scheduler() {
        @Override
        public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
            // this prevents StackOverflowErrors when scheduling with a delay
            return super.scheduleDirect(run, 0, unit);
        }

        @Override
        public Worker createWorker() {
            return new ExecutorScheduler.ExecutorWorker(Runnable::run);
        }
    };

    @Override
    public Statement apply(final Statement base, Description description) {
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                RxJavaPlugins.setInitIoSchedulerHandler(scheduler -> immediate);
                RxJavaPlugins.setInitComputationSchedulerHandler(scheduler -> immediate);
                RxJavaPlugins.setInitNewThreadSchedulerHandler(scheduler -> immediate);
                RxJavaPlugins.setInitSingleSchedulerHandler(scheduler -> immediate);
                RxAndroidPlugins.setInitMainThreadSchedulerHandler(scheduler -> immediate);

                try {
                    base.evaluate();
                } finally {
                    RxJavaPlugins.reset();
                    RxAndroidPlugins.reset();
                }
            }
        };
    }
}
```

2. 采用 Mockito 框架 mock 一个测试用的 DataManager 和 V 层代理 NewsContract.View。所谓的 mock 就是创建一个类的虚假的对象，在测试环境中，用来替换掉真实的对象，以达到验证对象方法调用情况，或是指定这个对象的某些方法返回特定的值等；
3. @Before 注解的方法会在执行测试用例之前执行，这里做一个初始化的操作，主要是 Mockito 框架的初始化及 presenter 的初始化；@After 注解的方法会在执行测试用例之后执行，这里做一个 presenter 的 detach() 操作，防止出现内存泄露等问题；
4. @Test 注解的方法是实际执行的测试方法。这里根据之前的业务代码逻辑：
- 环境初始化：由于 NewsPresenter 的业务逻辑中是需要 DataManager 返回一个 NewsResultBean 实例才能进行后续的操作，而 mock 的话只能返回一个空对象，所以在代码前两行笔者通过 Mockito 的 when() 方法，在程序调用 DataManager#getNews() 方法时返回一个空的 NewsResultBean 实例。
- 执行操作：执行 P 层的 NewsPresenter#getNews()。在业务逻辑中，执行此方法之后，会先调用 DataManager#getNews()，然后将数据交给 V 层的代理。
- 验证结果正确性：一般来说，我们要验证一个方法执行结果是否正确，最简单的方法的就是看执行完的方法输出是否与预期输出相一致。但在这里，NewsPresenter#getNews() 为一个 void 方法，没有返回值，那么该怎么验证呢？其实这个方法也是有输出的，输出就是：调用了 DataManager#getNews() 方法，获取到数据后调用 NewsContract.View#showNews(news) 显示数据。所以这里主要验证的是 DataManager#getNews() 和 NewsContract.View#showProgress()，NewsContract.View#showNews(news) 和 NewsContract.View#hideProgress() 这三个方法是否有被调用到，这里运用到 Mockito 的 verify() 方法。

至此，一个 Android 的单元测试用例编写完成。通过 Android Studio 直接运行此单元测试用例，结果如下：

![](https://xu-1254434063.cos.ap-guangzhou.myqcloud.com/pic/0630blogimg1.png)

> 需要明白一个点：单元测试它只是测试一个方法单元，它不是测试一整个 APP 的功能流程，即单元测试不会涉及到数据库或网络等复杂的外部环境。比如说这里我们只测试到 NewsPresenter#getNews() 方法，并没有测试 NewsFragment 的整个初始化到显示的过程是否正常，数据是否有误。（这样的测试往往称之为集成测试）


### Android UI 测试实践

#### 创建新用例

如果要编写一个新的本地 UI 测试用例，只需打开你想测试的 java 代码文件，然后点击类名 -- ⇧⌘T（Windows：Ctrl+Shift+T）-- 选择要生成的方法 -- 选择 androidTest 文件夹，对应于本地 UI 测试 -- 完成。

#### 增加依赖库

需要 Espresso 框架支持，所以在 build.gradle 中增加（注意是 androidTestImplementation）：
```groovy
androidTestImplementation "androidx.test:runner:1.1.0"
androidTestImplementation "androidx.test:rules:1.1.0"
androidTestImplementation "androidx.test.espresso:espresso-core:3.0.2"
androidTestImplementation "androidx.test.espresso:espresso-contrib:3.0.2"
androidTestImplementation "androidx.test.espresso:espresso-intents:3.0.2"
androidTestImplementation "androidx.test.espresso.idling:idling-concurrent:3.0.2"
androidTestImplementation "androidx.test.espresso:espresso-idling-resource:3.0.2"
```

#### 编写测试代码

笔者主要测试的代码为 NewsDetailActivity，主要功能是加载 intent 传递过来的新闻标题和新闻原文地址，然后在 Toolbar 中显示新闻标题，在 Webview 中加载此新闻。

对应着，实际编写测试代码的时候，可以构造一个测试用的 intent，在 intent 中加入需要的测试数据，然后启动这个 activity，检查数据是否正确即可。这里我们借助 Espresso 框架，它有三个重要的组成部分：ViewMatchers（根据视图 id 或其他属性匹配指定的 View），ViewActions（执行 View 的某些行为，例如点击事件），ViewAssertions（检查 View 的某些状态，例如指定 View 是否显示在屏幕上）。

新创建的 UI 测试代码如下：

```java
/**
 * Created by Xu on 2019/04/09.
 */
@RunWith(AndroidJUnit4.class)
@LargeTest
public class NewsDetailActivityTest {

    @Rule
    public ActivityTestRule<NewsDetailActivity> newsDetailActivityActivityTestRule =
            new ActivityTestRule<>(NewsDetailActivity.class, true, false);

    @Before
    public void setUp() {
        Intent intent = new Intent(InstrumentationRegistry.getInstrumentation().getTargetContext(), NewsDetailActivity.class);
        intent.putExtra(Constants.NEWS_URL, TestConstants.NEWS_DETAIL_ACTIVITY_TEST_URL);
        intent.putExtra(Constants.NEWS_IMG, TestConstants.NEWS_DETAIL_ACTIVITY_TEST_IMG);
        intent.putExtra(Constants.NEWS_TITLE, TestConstants.NEWS_DETAIL_ACTIVITY_TEST_TITLE);
        newsDetailActivityActivityTestRule.launchActivity(intent);
        IdlingRegistry.getInstance().register(newsDetailActivityActivityTestRule.getActivity().getCountingIdlingResource());
    }

    @Test
    public void showNewsDetail() {
        onView(withId(R.id.toolbar)).check(matches(isDisplayed()));
        onView(withId(R.id.iv_news_detail_pic)).check(matches(isDisplayed()));
        onView(withId(R.id.clp_toolbar)).check(matches(isDisplayed()));
        onView(withId(R.id.clp_toolbar)).check(matches(withCollapsingToolbarLayoutText(is(TestConstants.NEWS_DETAIL_ACTIVITY_TEST_TITLE))));
    }

    @After
    public void tearDown() {
        IdlingRegistry.getInstance().unregister(newsDetailActivityActivityTestRule.getActivity().getCountingIdlingResource());
    }
}
```

在其中：
1. 在类声明的开头，添加了两个注解 @RunWith(AndroidJUnit4.class) 和 @LargeTest；

@RunWith 注解可以改变 JUnit 测试用例的的默认执行类，由于这里是需要 Android 环境且使用到 Espresso 框架，所以 @RunWith 选择 AndroidJUnit4 类。@LargeTest 表示此测试用例会使用到外部文件系统或者网络，并且运行时间大于 1000 ms。

2. 声明了一个变量 newsDetailActivityActivityTestRule 并用 @Rule 注解，newsDetailActivityActivityTestRule 是 ActivityTestRule 的实例化对象。ActivityTestRule 主要用来测试单个 Activity，这个 Activity 将在 @Test 和 @Before 前启动。它其中包含一些基础功能，例如启动 Activity，获取当前 Activity 实例等；
3. 同样的，这里 @Before 注解的方法会在执行测试用例之前执行，这里构造一个测试用 intent，最后通过 newsDetailActivityActivityTestRule#launchActivity(intent) 方法启动待测试 Activity，并做一个 IdlingResource 的绑定；@After 注解的方法会在执行测试用例之后执行，这里做一个 IdlingResource 的解绑操作；

什么是 IdlingResource 呢？

通常来说，大多数 APP 在设计业务功能的过程中，会有很多的异步任务，例如使用 Rxjava 发起网络请求等，但是 Espresso 并不知道你的异步任务什么时候结束，如果单纯使用 Thread.sleep() 等待异步回调的结果又过于“硬核”，所以需要借助于 IdlingResource 这个类。

它需要在业务代码中添加相关的逻辑。例如在 NewsDetailActivity 中，会接收到 intent 传递过来的新闻图片地址，然后使用 Glide 异步加载此图片，大致代码如下：

```java
public class NewsDetailActivity extends AppCompatActivity {

    @BindView(R.id.iv_news_detail_pic)
    private ImageView ivNewsDetailPic;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_news);
        // 省略部分代码逻辑
        
        // 开始发起异步操作，App开始进入忙碌状态
        EspressoIdlingResource.increment();
        
        // 开始加载图片
        Glide.with(context).asBitmap().load(imgUrl).into(new GlideDrawableImageViewTarget(mAvatar) {
            @Override
            public void onResourceReady(@NonNull Bitmap resource, @Nullable Transition<? super Bitmap> transition) {
                super.onResourceReady(resource, transition);
                // 异步操作结束，将App设置成空闲状态
                if (!EspressoIdlingResource.getIdlingResource().isIdleNow()) {
                    EspressoIdlingResource.decrement();
                }
            }
        });
    }
    
    // 省略代码
    
    @VisibleForTesting
    public IdlingResource getCountingIdlingResource() {
        return EspressoIdlingResource.getIdlingResource();
    }
}

public class EspressoIdlingResource {
    private static final String RESOURCE = "GLOBAL";

    // Espresso 提供了一个实现好的 CountingIdlingResource 类
    // 如果没有特别需求的话，直接使用它即可
    private static CountingIdlingResource countingIdlingResource = new CountingIdlingResource(RESOURCE);

    public static void increment() {
        countingIdlingResource.increment();
    }

    public static void decrement() {
        countingIdlingResource.decrement();
    }

    public static IdlingResource getIdlingResource() {
        if (countingIdlingResource == null) {
            countingIdlingResource = new CountingIdlingResource(RESOURCE);
        }
        return countingIdlingResource;
    }

}
```

再加上我们在测试代码中声明的 IdlingRegistry.getInstance().register() 和 IdlingRegistry.getInstance().unregister() 方法，根据 APP 是否处于忙碌状态来判断异步任务是否完成，这样 Espresso 就能做到对异步任务进行相应的测试。

4. @Test 注解的方法是实际执行的测试方法。这里根据之前的业务代码逻辑：
- 环境初始化：模拟了测试的 intent 数据
- 执行操作：加载 intent 传递过来的数据
- 验证结果正确性：检查对应的 UI 样式是否正常显示测试数据，这里主要利用 Espresso 的 几个重要的 API：
  - onView()：获得视图 view，这里通过 withId() 方法搜索，即根据 id 来获取对应的 view 
  - check()：检验视图 view，可以检查视图文本是否匹配或者视图是否显示等，主要依靠 match() 方法返回对应的匹配类，Espresso 也自带很多已封装好的 View Matchers 供使用
  
以链式代码的形式编写验证测试结果的代码，例如 onView(withId(R.id.toolbar)).check(matches(isDisplayed())); 意思就是获取 id 为 R.id.toolbar 的 view，检查这个 view 是否正常显示。

如果 Espresso 自带的 View Matchers 不能满足需求的话，我们也可以自定义一个 matcher，例如 onView(withId(R.id.clp_toolbar)).check(matches(withCollapsingToolbarLayoutText(is(TestConstants.NEWS_DETAIL_ACTIVITY_TEST_TITLE)))); ，我们获取到的 view 是一个 CollapsingToolbarLayout，是一个特殊样式的 Toolbar，我们要检查其中的标题是否与测试数据相匹配，我们可以编写自定义的 Matcher：

```java
public static Matcher<View> withCollapsingToolbarLayoutText(Matcher<String> stringMatcher) {
    return new BoundedMatcher<View, CollapsingToolbarLayout>(CollapsingToolbarLayout.class) {
        @Override
        public void describeTo(Description description) {
            description.appendText("with CollapsingToolbarLayout title: ");
            stringMatcher.describeTo(description);
        }

        @Override
        protected boolean matchesSafely(CollapsingToolbarLayout collapsingToolbarLayout) {
            return stringMatcher.matches(collapsingToolbarLayout.getTitle());
        }
    };
}
```

这里传入一个 String 类型的匹配器（通过 is() 方法返回），返回一个 CollapsingToolbarLayout title 的 Matcher。

至此，一个 Android 的 UI 测试用例编写完成。通过 Android Studio 直接运行此用例，结果如下：

![](https://xu-1254434063.cos.ap-guangzhou.myqcloud.com/pic/0630blogimg2.png)

### 总结

本文主要从测试的两个不同粒度：单元测试和 UI 测试入手，综合参考 Google Sample 项目中的测试代码，做一个初步实践，分析编写并运行相关的测试用例。

笔者认为编写 Android 的测试用例的大致流程如下：
1. 确定需要编写的测试用例粒度；
2. 分析针对需要测试的页面，提取出较为重要且简短的业务代码逻辑；
3. 根据这些逻辑，通过三步走（初始化--执行--验证）方法来设计测试用例，这里的业务逻辑不仅仅是指业务需求，还包括其他需要维护的业务或公共代码逻辑；
4. 在做单元测试时，个人认为测试的业务逻辑不需要跨很多页面，在当前页面执行即可，以免增加单元测试用例的维护成本；
5. 单元测试用例并不能直接提升代码质量，但能够在进行项目重构的时候，保证重构的新代码能正确运行，降低风险。

