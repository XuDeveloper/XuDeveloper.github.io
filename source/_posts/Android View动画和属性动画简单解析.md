---
title: Android View动画和属性动画简单解析
tags: Android知识点
categories: Android
abbrlink: 348ce477
date: 2017-10-21 14:58:23
---

#### 一、View动画

简介：View动画通过对场景里的对象不断做图像变换（平移、缩放、旋转、透明度）从而产生动画效果，是一种渐近式动画，并且View动画支持自定义。

<!--more-->

1. View动画主要分为四类：TranslateAnimation，ScaleAnimation，RotateAnimation，AlphaAnimation，可通过XML或者Java代码声明使用，动画XML文件需要放在res/anim/filename.xml中。
例子：
```xml
<set xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    android:interpolator=""
    android:shareInterpolator="["true" | "false"] >
    <alpha
        android:fromAlpha="float"
        android:toAlpha="float" />
    <scale
        android:fromXScale="float"
        android:toXScale="float"
        android:fromYScale="float"
        android:toYScale="float"
        android:pivotX="float"
        android:pivotY="float" />
    <translate 
        android:fromXDelta="float" 
        android:toXDelta="float"
        android:fromYDelta="float"
        android:toYDelta="float" />
    <rotate 
        android:fromDegrees="float"
        android:toDegrees="float"
        android:pivotX="float"
        android:pivotY="float" />
</set>
```

Java代码：
```java
// 使用Java代码加载XML动画
Animation animation = AnimationUtils.loadAnimation(this, R.anim.animation_test);
mButton.startAnimation(animation);
// 使用Java代码创建动画
AlphaAnimation alphaAnimation = new AlphaAnimation(0, 1);
```

2. View动画既可以是单个动画，也可以由一系列动画组成。
3. 几个标签解读：
- set：
表示动画集合，对应AnimationSet类，它可以包含若干个动画，并且它的内部也是可以嵌套其他动画集合的。
- android:interpolator：
表示动画集合所采用的插值器，什么是插值器呢？它影响动画的速度，比如非匀速动画就需要通过插值器来控制动画的播放过程。属性可不指定，默认为@android:anim/accelerate_decelerate_interpolator，即加速减速插值器。
- android:shareInterpolator：
表示集合中的动画是否和集合共享同一个插值器。如果集合不指定插值器，那么子动画就需要单独指定所需的插值器或者使用默认值。
其余的属性网上都能查到，这里就不详细描述了。

#### 二：属性动画

简介：属性动画通过动态地改变相关对象的属性，比如长宽等，从而实现动画效果，属性动画为API 11（Android 3.0）以上的新特性，在低版本无法直接使用属性动画，但仍然可通过兼容库（NineOldAndroids）去使用。

属性动画有ValueAnimator、ObjectAnimator和AnimatorSet等概念。其中ObjectAnimator继承自ValueAnimator、AnimatorSet是动画集合，可以定义一组动画。

（1）使用
举例：改变一个对象（myObject）的translationY属性，让其沿着Y轴上平移一段距离：
```java
ObjectAnimator.ofFloat(myObject, "translationY", -myObject.getHeight()).start();
```
（2）插值器和估值器：
属性动画有两个新概念：
   插值器：根据时间流逝的百分比来计算出属性值改变的百分比，对应的接口是Interpolator；
   估值器：根据属性改变的百分比计算出属性的改变值，对应的接口是TypeEvaluator；
代码设置：

```java
ValueAnimator.setEvaluator(TypeEvaluator evaluator)
ValueAnimator.setInterpolator(TimeInterpolator value)
```

（3）属性动画的监听器
属性动画提供了监听器用于监听动画的播放过程。主要有如下两个接口：AnimatorUpdateListener和AnimatorListener。
```java
public static interface AnimatorListener {
    void onAnimationStart(Animator animation);
    void onAnimationEnd(Animator animation);
    void onAnimationCancel(Animator animation);
    void onAnimationRepeat(Animator animation);
}
```

它可以监听动画的开始、结束、取消以及重复播放。系统提供了AnimatorListener的适配器类AnimatorListenerAdapter。
AnimatorUpdateListener：

```java
public static interface AnimatorUpdateListener { 
    void onAnimationUpdate(ValueAnimator animation);
}
```

AnimatorUpdateListener会监听整个动画过程，动画是由许多帧组成的，每播放一帧，onAnimationUpdate就会被调用一次。

（4）对任意属性做动画
属性动画的原理：属性动画要求动画作用的对象提供该属性的get和set方法，属性动画根据外界传递的该属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样，确切地说是随着时间的推移，所传递的值越来越接近最终值。
总结，对object的属性abc做动画，需满足条件：
（1）object必须提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法，因为系统要去取abc属性的初始值；
（2）object的setAbc对属性abc所做的改变必须能够通过某种方法反映出来，比如会带来UI的改变。

建议：
1. 给对象加上get和set方法；
2. 用一个类来包装原始对象，间接为其提供get和set方法；
3. 用ValueAnimator，监听动画过程，实现属性的变化。