---
title: Android adb命令的一些实际运用
tags:
  - Android实践
  - Python
categories: Android
abbrlink: e774643a
date: 2017-11-04 23:10:52
---

在开发应用的过程中，安卓平台给大家提供了非常多的调试工具，包括Android Studio本身自带的工具，如果不想使用Studio的话，也可以在终端使用adb工具进行调试。

关于adb的用法网上有很多教程，这里推荐一个较为完整的指南https://github.com/mzlogin/awesome-adb。

今天记录一下我在实际情况中对adb的运用。

<!--more-->


#### 1.关于adb shell input text的问题

在使用这个命令的时候，我遇到了一个情况，就是无法输入"&"。我在网上搜了一下，在StackOverFlow里面，解决方案是这样的：
adb shell input text "\&"

这个方案在终端运行是可以的，但是我是用python写脚本运行的，这样做是无法成功的。

```python
cmd = "adb shell input text '\&'"
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE, shell=True)
```

想了很久，最后使用以下解决方法，原因我也不太理解，不知道有没有人来解答一下~

```python
cmd = "adb shell input text '\&'"
cmd = cmd.replace('&', "\"\&\"")
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE, shell=True)
```

#### 2.关于grep

grep命令起源于Linux系统。grep命令全称是Global Regular Expression Print，是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。在使用adb命令的时候，我们也经常需要使用到它。

举一个例子：
```python
adb shell dumpsys window policy
```

这个命令会展示出android当前窗口（window）的所有属性信息：

![image.png](http://upload-images.jianshu.io/upload_images/1963233-d49995806d1cc793.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

如果我们想在其中提取出mShowingLockscreen属性要怎么做？

网上给了这样一种方法：
```python
adb shell dumpsys window policy | grep mShowingLockscreen
```

同样在终端使用以及用python写脚本运行，出现问题：

![系统终端测试.png](http://upload-images.jianshu.io/upload_images/1963233-fa558b324514637c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![git bash终端测试.png](http://upload-images.jianshu.io/upload_images/1963233-f3386a7a25e5c219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

（真是奇怪，同样是终端，差别咋怎么大，无法理解.......）

原因不明，下面给出几种解决方案：

* 使用findstr

findstr相当于Windows下的grep命令。
```python
adb shell dumpsys window policy | findstr mShowingLockscreen
```

运行成功！

![success1.png](http://upload-images.jianshu.io/upload_images/1963233-c3f67fff01acf4d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

* 使用^| grep

个人理解：^| 有点类似于转义的作用
```python
adb shell dumpsys window policy ^| grep mShowingLockscreen
```

运行成功！

![success2.png](http://upload-images.jianshu.io/upload_images/1963233-14c95092114c94af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

#### 3.关于python运行adb命令返回结果的问题

一般情况下，使用python运行adb命令是非常方便的，例如：

```python
import subprocess
cmd = "adb shell input text test"
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
```

但其实它只对一些立即返回结果的命令有用，对于一些需要一定等待时间的命令，它有时就会出现错误，例如：

```python
import subprocess
cmd = "adb shell ping -c 4 www.baidu.com"
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
print "test"
```

这样执行会出现错误，会直接输出"test"。
这是因为subprocess.Popen对象创建后，主程序并不会自动等待子进程完成。我们必须调用对象的wait()方法，父进程才会等待 (也就是阻塞block)：

```python
import subprocess
cmd = "adb shell ping -c 4 www.baidu.com"
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
p.wait()
print "test"
```

大家可以运行对比一下~

#### 4.如何获取一段时间的logcat日志

使用的是adb logcat命令，具体的参数网上有很多，这里就不详细展开。

这里记录一下我是如何获取一段时间的logcat日志的：
```python
cmd = "adb logcat -v time"
p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
time.sleep(10)  # 这里用time.sleep模拟了一段时间，可以把它替换成你需要执行的操作
p.terminate()  # 终止程序，相当于终端用Ctrl + C
result = p.communicate()[0] # 获取执行操作前后的日志
```

如有问题，请大家踊跃提出，谢谢大家！





