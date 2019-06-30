---
title: 分享一下自己做的一个图片加载库XImageLoader
tags: Android实践
categories: Android
abbrlink: e51028a3
date: 2017-10-21 15:10:23
---

这是一个我自己做的一个Android的自定义图片加载库，主要参考了网上一些大神写过的一些图片加载库，再结合自己的一些想法理解去做了一个较为完整的图片加载库。当然代码中也会存在一些不足的情况，例如代码架构方面不是非常的完善。希望大家提出意见，一起去改进！

注意：这是一个用于学习图片加载与缓存的库，不推荐使用在实际项目之中！

Github地址：https://github.com/XuDeveloper/XImageLoader

<!--more-->



如果你想改进这个图片加载库，欢迎在GitHub上fork这个项目然后pull request给我！如果你喜欢它，请给这个项目一个star或者关注我的GitHub！谢谢你们的支持！
 
### 导入

#### Android Studio

``` xml
  allprojects {
		repositories {
			...
			maven { url "https://jitpack.io" }
		}
  }

  dependencies {
	    compile 'com.github.XuDeveloper:XImageLoader:v1.0'
  }

```
#### Eclipse

> 可以复制源码到你的项目中！

### 使用

默认用法：

``` java

	// 异步接口调用
    XImageLoader.build(context).imageview(ImageView).load(imageUrl);
	// 加载本地文件，你需要使用这样的格式："file:///address"
	XImageLoader.build(context).imageview(ImageView).load("file:///address");

```

或者：

```java

	// 同步接口调用(需要运行在一条新线程中)
	Bitmap bitmap = XImageLoader.build(context).imageview(ImageView).getBitmap(imageUrl);

```

你可以选择是否缓存或者自定义（使用XImageLoaderConfig）：


```java

	XImageLoader.build(context).imageview(isMemoryCache, isDiskCache, ImageView).load(imageUrl);

	XImageLoader.build(context).imageview(isDoubleCache, ImageView).load(imageUrl);
	
	// 具体配置
    XImageLoaderConfig config = new XImageLoaderConfig();
    config.setCache(new DoubleCache(context));
    config.setLoader(new OkhttpImageLoader());
    config.setLoadingResId(R.drawable.image_loading);
    config.setFailResId(R.drawable.image_fail);
	XImageLoader.build(context).imageview(config, ImageView).load(imageUrl);

```

你需要AndroidManifest.xml中设置权限:

```xml

	<uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

```

如果你使用的是Android 6.0以上的设备，你需要动态设置权限：

```java

	XImageLoader.verifyStoragePermissions(activity);

```

代码架构设计：
采用流式编程写法，将加载一张图片分为以下几步：
1.初始化一些基本设置（XImageLoaderConfig ），如是否加入缓存，设置加载时显示的图片资源以及加载失败时显示的图片资源还有图片加载器ImageLoader；（如果不设置下面会自动设置）
2.设置需要加载的ImageView；
3.设置需要加载的图片路径，可以是网络上的图片，也可以是手机本地图片，如果前面没设置ImageLoader，在这里就可以根据路径来动态选择加载哪个特定的ImageLoader；
加载图片有两种方法，一种是同步方法获取图片Bitmap，另一种是在线程池中异步加载图片。

加载图片的优化方法：
1.加入缓存机制，包括LruCache以及DiskLruCache；
2.加载时先预读取图片，根据尺寸计算inSampleSize，对图片进行适当压缩，防止OOM；
3.采用线程池加载。

需要进一步优化的地方：
读取需要加载的ImageView尺寸时用到了反射的方法，影响了一定的效率；