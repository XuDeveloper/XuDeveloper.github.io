---
title: Kotlin-Android-Extensions 库使用及源码解析
tags: Android源码解析
categories: Android
abbrlink: d6c4ec97
date: 2020-02-06 21:10:23
---

#### **本文预计阅读时间为 15-20 分钟**

### 一、Kotlin-Android-Extensions 简介

Kotlin 从首次推出到现在，可谓发展的十分迅速，独特的空安全特性吸引了很多 Android 开发者去使用，Google 也正式将 Kotlin 这门语言作为 Android 开发的首选语言。Kotlin 官方也为各位开发者提供了一系列的插件，开发文档以及 IDE 支持，本文介绍的 Kotlin-Android-Extensions 就是一款 Kotlin 的安卓开发扩展插件。

<!--more-->

### 二、Kotlin-Android-Extensions 使用

#### 引入

直接在 build.gradle 中引入该插件：

```
apply plugin: 'kotlin-android-extensions'
```

#### 使用

模拟的业务场景如下：
- 在 activity\_main.xml 中创建一个 id 为 button\_test 的 button
- 在 MainActivity.kt 中为这个 button 设置点击事件

```Kotlin
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import kotlinx.android.synthetic.main.activity_main.*

/**
 * Created by Xu on 2020/02/05.
 *
 * @author Xu
 */
class MainActivity: AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        button_test.setOnClickListener { 
            // todo
        }
    }
}
```

这里可以观察到，并没有熟悉的 findViewById() 方法，而是直接使用了 button_test 这个对象，该对象其实是由插件根据布局 xml 中所设置的控件 id 而自动生成的。

### 三、Kotlin-Android-Extensions 源码分析

为什么不需要使用到 findViewById() 方法呢？之前我在分析 ButterKnife 源码的时候也问过类似的问题（[传送门](https://juejin.im/post/5a36412f6fb9a0451d418e2f)），最后其实是通过 APT（编译时注解）的方式自动生成了 findViewById 方法，猜测这里也是通过类似的自动生成代码方式帮我们补充了。

我们首先试着去反编译 Kotlin ByteCode，具体是通过打开 Android Studio -> Tools -> Kotlin -> Show Kotlin Bytecode，然后选择 build 文件夹下的 MainActivity.class，点击 Decompile 即可。反编译完代码如下：

```Java
public final class MainActivity extends AppCompatActivity {
   private HashMap _$_findViewCache;

   protected void onCreate(@Nullable Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      this.setContentView(layout.activity_main);
      ((Button)this._$_findCachedViewById(id.button_test)).setOnClickListener((OnClickListener)null.INSTANCE);
   }

   public View _$_findCachedViewById(int var1) {
      if (this._$_findViewCache == null) {
         this._$_findViewCache = new HashMap();
      }

      View var2 = (View)this._$_findViewCache.get(var1);
      if (var2 == null) {
         var2 = this.findViewById(var1);
         this._$_findViewCache.put(var1, var2);
      }

      return var2;
   }

   public void _$_clearFindViewByIdCache() {
      if (this._$_findViewCache != null) {
         this._$_findViewCache.clear();
      }

   }
}
```

这里会发现多了一个 \_\$\_findViewCache 的成员变量以及 \_\$\_findCachedViewById 的方法，而这个方法内部其实也是使用到了 findViewById，并且对 view 进行了缓存，避免了该方法的重复调用。

那么这些代码是怎么生成的呢？通过谷歌的搜索，笔者找到了该插件的源代码地址（[传送门](https://github.com/JetBrains/kotlin/tree/master/plugins/android-extensions)），然后观察到这个变量和方法命名是固定的，跟具体的类命名无关，猜测是一个固定的常量值，在代码中进行全局搜索，找到以下这个相关类：

AbstractAndroidExtensionsExpressionCodegenExtension.kt：
```Kotlin
abstract class AbstractAndroidExtensionsExpressionCodegenExtension : ExpressionCodegenExtension {
    companion object {
        val PROPERTY_NAME = "_\$_findViewCache"
        val CACHED_FIND_VIEW_BY_ID_METHOD_NAME = "_\$_findCachedViewById"
        val CLEAR_CACHE_METHOD_NAME = "_\$_clearFindViewByIdCache"
        val ON_DESTROY_METHOD_NAME = "onDestroyView"

        fun shouldCacheResource(resource: PropertyDescriptor) = (resource as? AndroidSyntheticProperty)?.shouldBeCached == true
    }
    // 省略部分代码   
}
```

再进一步的去搜索，找到类中对应的 generateCacheField() 和 generateCachedFindViewByIdFunction() 方法。

先看 generateCacheField()：

```Kotlin
private fun SyntheticPartsGenerateContext.generateCacheField() {
    val cacheImpl = CacheMechanism.getType(containerOptions.getCacheOrDefault(classOrObject))
    classBuilder.newField(JvmDeclarationOrigin.NO_ORIGIN, ACC_PRIVATE, PROPERTY_NAME, cacheImpl.descriptor, null, null)
}
```

这里用到 CacheMechanism 的 getType 方法，然后通过 classBuilder#newField()  生成。

CacheMechanism#getType()：
```Kotlin
fun getType(cacheImpl: CacheImplementation): Type {
    return Type.getObjectType(when (cacheImpl) {
        CacheImplementation.SPARSE_ARRAY -> "android.util.SparseArray"
        CacheImplementation.HASH_MAP -> HashMap::class.java.canonicalName
        CacheImplementation.NO_CACHE -> throw IllegalArgumentException("Container should support cache")
    }.replace('.', '/'))
}
```

这里返回的是 _\$_findViewCache 这个成员变量的类型，默认是 HashMap，也可以在 build.gradle 中指定类型：

```
androidExtensions {
    defaultCacheImplementation = "HASH_MAP" // or SPARSE_ARRAY、NONE
}
```

再看 generateCachedFindViewByIdFunction()：

```Kotlin
private fun SyntheticPartsGenerateContext.generateCachedFindViewByIdFunction() {
    val containerAsmType = state.typeMapper.mapClass(container)

    val viewType = Type.getObjectType("android/view/View")

    val methodVisitor = classBuilder.newMethod(
            JvmDeclarationOrigin.NO_ORIGIN, ACC_PUBLIC, CACHED_FIND_VIEW_BY_ID_METHOD_NAME, "(I)Landroid/view/View;", null, null)
    methodVisitor.visitCode()
    val iv = InstructionAdapter(methodVisitor)

    val cacheImpl = CacheMechanism.get(containerOptions.getCacheOrDefault(classOrObject), iv, containerAsmType)

    fun loadId() = iv.load(1, Type.INT_TYPE)

    // Get cache property
    cacheImpl.loadCache()

    val lCacheNonNull = Label()
    iv.ifnonnull(lCacheNonNull)

    // Init cache if null
    cacheImpl.initCache()

    // Get View from cache
    iv.visitLabel(lCacheNonNull)
    cacheImpl.loadCache()
    loadId()
    cacheImpl.getViewFromCache()
    iv.checkcast(viewType)
    iv.store(2, viewType)

    val lViewNonNull = Label()
    iv.load(2, viewType)
    iv.ifnonnull(lViewNonNull)

    // Resolve View via findViewById if not in cache
    iv.load(0, containerAsmType)

    val containerType = containerOptions.containerType
    when (containerType) {
        AndroidContainerType.ACTIVITY, AndroidContainerType.ANDROIDX_SUPPORT_FRAGMENT_ACTIVITY, AndroidContainerType.SUPPORT_FRAGMENT_ACTIVITY, AndroidContainerType.VIEW, AndroidContainerType.DIALOG -> {
            loadId()
            iv.invokevirtual(containerType.internalClassName, "findViewById", "(I)Landroid/view/View;", false)
        }
        AndroidContainerType.FRAGMENT, AndroidContainerType.ANDROIDX_SUPPORT_FRAGMENT, AndroidContainerType.SUPPORT_FRAGMENT, LAYOUT_CONTAINER -> {
            if (containerType == LAYOUT_CONTAINER) {
                iv.invokeinterface(containerType.internalClassName, "getContainerView", "()Landroid/view/View;")
            } else {
                iv.invokevirtual(containerType.internalClassName, "getView", "()Landroid/view/View;", false)
            }

            iv.dup()
            val lgetViewNotNull = Label()
            iv.ifnonnull(lgetViewNotNull)

            // Return if getView() is null
            iv.pop()
            iv.aconst(null)
            iv.areturn(viewType)

            // Else return getView().findViewById(id)
            iv.visitLabel(lgetViewNotNull)
            loadId()
            iv.invokevirtual("android/view/View", "findViewById", "(I)Landroid/view/View;", false)
        }
        else -> throw IllegalStateException("Can't generate code for $containerType")
    }
    iv.store(2, viewType)

    // Store resolved View in cache
    cacheImpl.loadCache()
    loadId()
    cacheImpl.putViewToCache { iv.load(2, viewType) }

    iv.visitLabel(lViewNonNull)
    iv.load(2, viewType)
    iv.areturn(viewType)

    FunctionCodegen.endVisit(methodVisitor, CACHED_FIND_VIEW_BY_ID_METHOD_NAME, classOrObject)
}
```

这里的代码比较复杂，但可以观察到一个重要的地方，它会去判断当前的类是 Activity 还是 Fragment，再去执行对应的寻找控件方法。例如是 Activity 的话，则执行的是 findViewById 方法；而如果是 Fragment，则先执行 getView 方法获取到对应的 rootView，再执行 findViewById。

还有一个点，最后的实现都会调用到 iv#invokevirtual() 方法，iv 是 InstructionAdapter 类的一个实例。InstructionAdapter 继承于 MethodVisiter，用途是生成方法实现的字节码，这里不再深究实现细节，有兴趣的读者可以再去了解一下。

### 四、Kotlin-Android-Extensions 总结

Kotlin-Android-Extensions 这个插件，通过**自动生成寻找控件代码的字节码，对查找完的控件进行缓存以及 IDE 跳转支持**等方式，使得 Android 的业务开发更加地便捷高效，有效提高研发效率，提升研发体验。