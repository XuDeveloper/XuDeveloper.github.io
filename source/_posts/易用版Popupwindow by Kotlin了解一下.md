---
title: 易用版Popupwindow by Kotlin了解一下
tags: Android实践
categories: Android
abbrlink: a0194f59
date: 2018-08-08 21:07:23
---


## 概述

XPopupWindow，对系统的PopupWindow进行进一步封装和加强以便于使用。采用Kotlin语言，提供了许多额外的功能方法例如设置弹窗位置，调整弹窗动画等等。

## 项目地址
[XPopupWindow](https://github.com/XuDeveloper/XPopupWindow)

<!--more-->

## 预览

<img src="https://user-gold-cdn.xitu.io/2018/8/6/1650f0e5d2edcf4f?w=800&h=1422&f=gif&s=1833921" width="380" height="800" alt="XPopupWindow-demo"/>


## 特性

* 简单快速地创建一个自定义弹窗
* 以一种相对便捷的方式设置弹窗位置
* 更加自由地调整你的弹窗动画


## 开始

使用Gradle：

```Groovy
allprojects {
    repositories {
        ...
	maven { url "https://jitpack.io" }
    }
}

dependencies {
    implementation 'com.github.XuDeveloper:XPopupWindow:1.0.1'
}
```

## 使用

以创建一个登录弹窗为例：

#### 界面编写
（略，含有一个账号输入框，一个密码输入框以及登录按钮，github有demo）

#### 创建XPopupWindow

```Kotlin
/**
 * Created by Xu on 2018/6/17.
 * @author Xu
 */

class InputPopupWindow : XPopupWindow {
    private var btnLogin: Button? = null
    private var etPhone: TextInputEditText? = null

    constructor(ctx: Context) : super(ctx)

    constructor(ctx: Context, w: Int, h: Int) : super(ctx, w, h)

    /**
     * 设置popupwindow的layoutId
     */
    override fun getLayoutId(): Int {
        return R.layout.popup_input
    }

    /**
     * 设置layout的parentNodeId
     */
    override fun getLayoutParentNodeId(): Int {
        return R.id.input_parent
    }

    /**
     * 初始化界面
     */
    override fun initViews() {
        btnLogin = findViewById(R.id.btn_login)
        btnLogin?.setOnClickListener { dismiss() }
        etPhone = findViewById(R.id.et_mobile)
    }

    /**
     * 初始化数据
     */
    override fun initData() {
        // 设置弹窗背景透明度
        setShowingBackgroundAlpha(0.4f)
        // 弹窗弹出时自动获取输入框的焦点
        setAutoShowInput(etPhone, true)
    }

    /**
     * 为弹窗设置弹出动画，如果不想设置或是想通过xml方式设置，则设置返回值为-1
     */
    override fun startAnim(view: View): Animator? {
        var animatorX: ObjectAnimator = ObjectAnimator.ofFloat(view, "scaleX", 0f, 1f)
        var animatorY: ObjectAnimator = ObjectAnimator.ofFloat(view, "scaleY", 0f, 1f)
        var set = AnimatorSet()
        set.play(animatorX).with(animatorY)
        set.duration = 500
        return set
    }

    /**
     * 为弹窗设置退出动画，如果不想设置或是想通过xml方式设置，则设置返回值为-1
     */
    override fun exitAnim(view: View): Animator? {
        var animatorX: ObjectAnimator = ObjectAnimator.ofFloat(view, "scaleX", 1f, 0f)
        var animatorY: ObjectAnimator = ObjectAnimator.ofFloat(view, "scaleY", 1f, 0f)
        var set = AnimatorSet()
        set.play(animatorX).with(animatorY)
        set.duration = 700
        return set
    }

    /**
     * 通过xml方式设置动画，xml编写方法与原生popupwindow设置动画方法相同
     */
    override fun animStyle(): Int {
        return -1
    }

}

```

#### 具体使用
```kotlin
private fun showInputPopup() {
    inputPopupWindow = InputPopupWindow(this, 1000, 600)
    // 可设置弹窗退出的监听器，在回调中执行相应操作
    inputPopupWindow?.setXPopupDismissListener(object : XPopupWindowDismissListener {
        override fun xPopupBeforeDismiss() {
        }

        override fun xPopupAfterDismiss() {
            Snackbar.make(findViewById(android.R.id.content), "登录成功！", Snackbar.LENGTH_LONG).show()
        }
    })
    inputPopupWindow?.showPopupFromScreenCenter(R.layout.activity_main)
}
```

* **你可以查看xpopupwindowdemo以获取更多使用方法！**

## 给我买杯柠檬茶呗 :smile:

| 微信 |支付宝 | 
| ---- | ---- | 
| ![](https://user-gold-cdn.xitu.io/2018/8/6/1650f0fb5473091e?w=900&h=1350&f=jpeg&s=145793) |    ![](https://user-gold-cdn.xitu.io/2018/8/6/1650f101e76f3a11?w=900&h=1350&f=jpeg&s=150503)

## 协议
```license
Copyright [2018] XuDeveloper

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```