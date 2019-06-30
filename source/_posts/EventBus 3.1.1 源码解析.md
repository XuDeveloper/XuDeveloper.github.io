---
title: EventBus 3.1.1 源码解析
tags: Android源码解析
categories: Android
abbrlink: 1a3301f8
date: 2018-01-20 17:10:23
---

##### * 本篇文章已授权微信公众号 guolin_blog（郭霖）独家发布

#### 一、本文需要解决的问题
我研究EventBus源码的目的是解决以下几个我在使用过程中所思考的问题：

1. 这个框架涉及到一种设计模式叫做观察者模式，什么是观察者模式？
2. 事件如何进行定义，有没有相关限制？
3. 观察者绑定观察事件的时候，绑定方法的命名有限制吗？
4. 事件发送和接收的原理？


<!--more-->

#### 二、初步使用
为了研究源码的方便，我写了一个简单的demo。

##### 定义事件
TestEvent.java：
```java
public class TestEvent {
    private String msg;

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

##### 主Activity
MainActivity.java：
```java
public class MainActivity extends AppCompatActivity {

    private Button button;
    private TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        button = findViewById(R.id.button);
        textView = findViewById(R.id.text);

        EventBus.getDefault().register(this);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                TestEvent event = new TestEvent();
                event.setMsg("已接收到事件!");
                EventBus.getDefault().post(event);
            }
        });

    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onTestEvent(TestEvent event) {
        textView.setText(event.getMsg());
    }

    @Override
    protected void onDestroy() {
        EventBus.getDefault().unregister(this);
        super.onDestroy();
    }
}
```

##### 运行效果
![demo.gif](http://upload-images.jianshu.io/upload_images/1963233-58fa867fd7239615.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

#### 三、源码分析
##### 关于观察者模式
- 简介：**观察者模式**是设计模式中的一种。它是为了定义对象间的一种一对多的依赖关系，即当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。
- 如何使用：这里[传送门](http://www.runoob.com/design-pattern/observer-pattern.html)有相关的demo，这里不再详述。
- 重点：在这个模式中主要包含两个重要的角色：**发布者**和**订阅者（又称观察者）**。对应EventBus来说，发布者即发送消息的一方（即调用EventBus.getDefault().post(event)的一方），订阅者即接收消息的一方（即调用EventBus.getDefault().register()的一方）。
我们已经解决了第一个问题~

##### 关于事件
这里指的事件其实是一个泛泛的统称，指的是一个概念上的东西（当时我还以为一定要以啥Event命名...），通过查阅官方文档，我知道事件的命名格式并没有任何要求，你可以定义一个对象作为事件，也可以发送基本数据类型如int，String等作为一个事件。后续的源码分析我也会再次证明一下。

##### 具体分析
从函数入口开始分析：
1.EventBus#getDefault()：
```java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```
这里就是采用双重校验并加锁的单例模式生成EventBus实例。
```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```
由于我们传入的为this，即MainActivity的实例，所以第一行代码获取了订阅者的class对象，然后会找出所有订阅的方法。我们看一下第二行的逻辑。
SubscriberMethodFinder#findSubscriberMethods()：
```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```
分析：
- 如果缓存中有对应class的订阅方法列表，则直接返回，这里我们是第一次创建，所以此时subscriberMethods为空；
- 接下来会有一个参数判断，通过查看前面的创建过程，ignoreGeneratedIndex默认为false，进入else代码块，后面生成subscriberMethods成功的话会加入到缓存中，失败的话会throw异常。

2.SubscriberMethodFinder#findUsingInfo()：
```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    // 2.1
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    // 2.2
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod: array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            // 2.3
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    // 2.4
    return getMethodsAndRelease(findState);
}
```

2.1 SubscriberMethodFinder#prepareFindState()：
```java
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

private FindState prepareFindState() {
    synchronized(FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```
这个方法是创建一个新的FindState类，通过两种方法获取，一种是从FIND_STATE_POOL即FindState池中取出可用的FindState，如果没有的话，则通过第二种方式：直接new一个新的FindState对象。
FindState#initForSubscriber()：
```java
static class FindState {
    // 省略代码
    void initForSubscriber(Class<?> subscriberClass) {
        this.subscriberClass = clazz = subscriberClass;
        skipSuperClasses = false;
        subscriberInfo = null;
    }
   // 省略代码
}
```
FindState类是SubscriberMethodFinder的内部类，这个方法主要做一个初始化的工作。

2.2 SubscriberMethodFinder#getSubscriberInfo()：
```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index: subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}
```
这里由于初始化的时候，findState.subscriberInfo和subscriberInfoIndexes为空，所以这里直接返回null，后续我们可以再回到这里看看subscriberInfo有什么作用。

2.3 SubscriberMethodFinder#findUsingReflectionInSingleClass()：
```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method: methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?> [] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // ！！！
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode, subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName + "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName + " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```
这个方法的逻辑是：
通过反射的方式获取订阅者类中的所有声明方法，然后在这些方法里面寻找以@Subscribe作为注解的方法进行处理（！！！部分的代码），先经过一轮检查，看看findState.subscriberMethods是否存在，如果没有的话，将方法名，threadMode，优先级，是否为sticky方法封装为SubscriberMethod对象，添加到subscriberMethods列表中。

##### 什么是sticky event？
sticky event，中文名为粘性事件。普通事件是先注册，然后发送事件才能收到；而粘性事件，在发送事件之后再订阅该事件也能收到。此外，粘性事件会保存在内存中，每次进入都会去内存中查找获取最新的粘性事件，除非你手动解除注册。

在这里我们解决了第二个和第三个问题，**方法的命名并没有任何要求，只是加上@Subscribe注解即可！同时事件的命名也没有任何要求！**

之后这个while循环会继续检查父类，当然遇到系统相关的类时会自动跳过，以提升性能。

2.4 SubscriberMethodFinder#getMethodsAndRelease
```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle();
    synchronized(FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```
这里将subscriberMethods列表直接返回，同时会把findState做相应处理，存储在FindState池中，方便下一次使用，提高性能。

3. EventBus#subscribe()：
返回subscriberMethods之后，register方法的最后会调用subscribe方法：
```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}

private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList <> ();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
        }
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```
分析：
- 首先，根据subscriberMethod.eventType（在Demo里面指的是TestEvent），在subscriptionsByEventType去查找一个CopyOnWriteArrayList<Subscription> ，如果没有则创建一个新的CopyOnWriteArrayList；
- 然后将这个CopyOnWriteArrayList放入subscriptionsByEventType中，这里的subscriptionsByEventType是一个Map，key为eventType，value为CopyOnWriteArrayList<Subscription>，这个Map非常重要，后续还会用到它；
- 接下来，就是添加newSubscription，它属于Subscription类，里面包含着subscriber和subscriberMethod等信息，同时这里有一个优先级的判断，说明它是按照优先级添加的。优先级越高，会插到在当前List靠前面的位置；
- typesBySubscriber这个类也是一个Map，key为subscriber，value为subscribedEvents，即所有的eventType列表，这个类我找了一下，发现在EventBus#isRegister()方法中有用到，应该是用来判断这个Subscriber是否已被注册过。然后将当前的eventType添加到subscribedEvents中；
- 最后，判断是否是sticky。如果是sticky事件的话，到最后会调用checkPostStickyEventToSubscription()方法。

这里其实就是将所有含@Subscribe注解的订阅方法最终保存在subscriptionsByEventType中。

4. EventBus#checkPostStickyEventToSubscription()：
```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
        // --> Strange corner case, which we don't take care of here.
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

接下来，我们重点看post()和postToSubscription()方法。post事件相当于把事件发送出去，我们看看订阅者是如何接收到事件的。

5. EventBus#post()：
```java
/** Posts the given event to the event bus. */
public void post(Object event) {
    // 5.1
    PostingThreadState postingState = currentPostingThreadState.get();
    List <Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    // 5.2
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

5.1 代码段分析
- currentPostingThreadState是一个ThreadLocal类型的，里面存储了PostingThreadState，而PostingThreadState中包含了一个eventQueue和其他一些标志位；
- 然后把传入的event，保存到了当前线程中的一个变量PostingThreadState的eventQueue中。
```java
private final ThreadLocal <PostingThreadState> currentPostingThreadState = new ThreadLocal <PostingThreadState> () {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};

/** For ThreadLocal, much faster to set (and get multiple values). */
final static class PostingThreadState {
    final List <Object> eventQueue = new ArrayList<>();
    boolean isPosting;
    boolean isMainThread;
    Subscription subscription;
    Object event;
    boolean canceled;
}
```

5.2 代码段分析
- 这里涉及到两个标志位，第一个是isMainThread，判断是否为UI线程；第二个是isPosting，作用是防止方法多次调用。
- 最后调用到postSingleEvent()方法

6. EventBus#postSingleEvent()：
```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
            eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}

private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class <?> eventClass) {
    CopyOnWriteArrayList <Subscription> subscriptions;
    synchronized(this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription: subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```
- 这里会首先取出Event的class类型，然后有一个标志位eventInheritance判断，默认为true，作用在相关代码注释有说，如果设为true的话，它会拿到Event父类的class类型，设为false，可以在一定程度上提高性能；
- 接下来是lookupAllEventTypes()方法，就是取出Event及其父类和接口的class列表，当然重复取的话会影响性能，所以它也有做一个eventTypesCache的缓存，这样不用都重复调用getClass()方法。
- 然后是postSingleEventForEventType()方法，这里就很清晰了，就是直接根据Event类型从subscriptionsByEventType中取出对应的subscriptions，与之前的代码对应，最后调用postToSubscription()方法。

7. EventBus#postToSubscription()：
```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```
这里会根据threadMode来判断应该在哪个线程中去执行方法：
- POSTING：执行invokeSubscriber()方法，就是直接反射调用；
- MAIN：首先去判断当前是否在UI线程，如果是的话则直接反射调用，否则调用mainThreadPoster#enqueue()，即把当前的方法加入到队列之中，然后通过handler去发送一个消息，在handler的handleMessage中去执行方法。具体逻辑在HandlerPoster.java中；
- MAIN_ORDERED：与上面逻辑类似，顺序执行我们的方法；
- BACKGROUND：判断当前是否在UI线程，如果不是的话直接反射调用，是的话通过backgroundPoster.enqueue()将方法加入到后台的一个队列，最后通过线程池去执行；
- ASYNC：与BACKGROUND的逻辑类似，将任务加入到后台的一个队列，最终由Eventbus中的一个线程池去调用，这里的线程池与BACKGROUND逻辑中的线程池用的是同一个。

补充：BACKGROUND和ASYNC有什么区别呢？
BACKGROUND中的任务是一个接着一个的去调用，而ASYNC则会即时异步运行，具体的可以对比AsyncPoster.java和BackgroundPoster.java两者代码实现的区别。

到这里，我们就解决了第四个问题，**事件的发送和接收，主要是通过subscriptionsByEventType这个非常重要的列表，我们将订阅即接收事件的方法存储在这个列表，发布事件的时候在列表中查询出相对应的方法并执行~**