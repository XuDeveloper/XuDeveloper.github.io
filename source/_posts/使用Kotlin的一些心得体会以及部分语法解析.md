---
title: 使用Kotlin的一些心得体会以及部分语法解析
tags:
  - Android实践
  - Kotlin
categories: Android
abbrlink: e8b85950
date: 2018-09-16 15:10:52
---

###### **本文预计阅读时间为10分钟**

笔者最近使用Kotlin语言编写一个强化版的Android popupwindow [传送门](https://github.com/XuDeveloper/XPopupWindow) 
个人认为Kotlin语言非常优雅，与Java相比，增加了很多特性和语法糖，在使用过程中也有了一定的思考，并做了一些简单的记录。

<!--more-->

#### 关于空安全（Kotlin的四个特殊操作符）
Kotlin相比于Java，做出了一个重大的改进，就是提出了一个代码空引用问题（就是俗称的NullPointerException）的解决方案。 
在Java中，编写代码总是很容易忘记对对象的非空判断，而Koltin用了以下几种机制保障：
1. ? 操作符
    在Kotlin里面声明一个引用，可以决定这个引用是可容纳null值 （称为可空引用）还是不可容纳null值（称为非空引用）
    例如：
    ```Kotlin
    var a: String = "xu"
    a = null // 编译错误
    var b: String? = "xu"
    b = null // ok
    ```

2. ?. 操作符
    在上述代码中，我们知道
    ```Kotlin
    val l = a.length // 保证不会导致NullPointerException
    val l = b.length // 变量“b”可能为空
    ```
    如何避免这种情况？
   
    ###### 2.1 条件检查
    类似Java的写法：
    ```Kotlin
    if (b != null && b.length > 0) {
        
    } else {
        
    }
    ```
    
    ###### 2.2 Kotlin安全调用
    使用的是?. 操作符
    ```Kotlin
    val test: String? = null
    print(test?.length) // 如果b非空，就返回b.length，否则返回null
    ```

3. ?: 操作符
    在上述的代码中，如果在b为null的情况下，我们想做一些其他的操作的话，可以使用此操作符
    ```Kotlin
    val l = b?.length ?: -1
    ```

4. !! 操作符
    非空断言运算符 !! ，它将任何值转换为非空类型，若该值为空则抛出异常
    ```Kotlin
    val l = b!!.length // 返回一个非空的b值。如果b为null，则会抛出一个NullPointerException异常
    ```

#### 关于data class数据类
在编程过程中，我们肯定会经常创建一些model模型类。 如果使用Java来写的话，在这些类中一般都需要写一大堆方法，例如
```Java
public class People {

    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "People{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        People people = (People) o;
        return age == people.age &&
                Objects.equals(name, people.name);
    }
    
}
```

在 Kotlin 中，我们如果想创建一个类似的模型类，只需要使用data关键字
```Kotlin
data class People(val name: String, val age: Int)
```

通过反编译class文件，我们知道在使用data关键字之后，Kotlin自动帮我们添加了以下方法：
```Java
public final class Student
{
  @NotNull
  private final String name;
  private final int age;
  
  public boolean equals(Object paramObject)
  {
    if (this != paramObject)
    {
      if ((paramObject instanceof Student))
      {
        Student localStudent = (Student)paramObject;
        if (Intrinsics.areEqual(this.name, localStudent.name)) {
          if ((this.age == localStudent.age ? 1 : 0) == 0) {}
        }
      }
    }
    else {
      return true;
    }
    return false;
  }
  
  /* Error */
  public int hashCode()
  {
    // Byte code:
    //   0: aload_0
    //   1: getfield 11	com/xu/xpopupwindow/config/Student:name	Ljava/lang/String;
    //   4: dup
    //   5: ifnull +9 -> 14
    //   8: invokevirtual 63	java/lang/Object:hashCode	()I
    //   11: goto +5 -> 16
    //   14: pop
    //   15: iconst_0
    //   16: bipush 31
    //   18: imul
    //   19: aload_0
    //   20: getfield 19	com/xu/xpopupwindow/config/Student:age	I
    //   23: iadd
    //   24: ireturn
  }
  
  public String toString()
  {
    return "Student(name=" + this.name + ", age=" + this.age + ")";
  }
  
  @NotNull
  public final Student copy(@NotNull String name, int age)
  {
    Intrinsics.checkParameterIsNotNull(name, "name");
    return new Student(name, age);
  }
  
  public final int component2()
  {
    return this.age;
  }
  
  @NotNull
  public final String component1()
  {
    return this.name;
  }
  
  public Student(@NotNull String name, int age)
  {
    this.name = name;this.age = age;
  }
  
  public final int getAge()
  {
    return this.age;
  }
  
  @NotNull
  public final String getName()
  {
    return this.name;
  }
}
```
Kotlin这里不仅仅帮我们创建了set()/get()/toString()方法，还有两个特殊的方法component1()和component2()
这两个方法的作用是能够保证数据类可以使用解构声明（destructuring declarations）。有多少个变量就有多少个component方法
那么什么是解构声明呢，就是把一个对象解构成很多变量去声明，一个解构声明同时创建多个变量，例如
```Kotlin
val xu = User("Xu", 24)
val (name, age) = xu
println("$name, $age")  // 这样就可以独立使用了
```

#### 关于扩展函数
在Kotlin语言中，如果说我们想在一个类中增加一个自定义的函数，可以对已有的类和里面的属性进行扩展性的操作，例如增加扩展函数。扩展函数本身不会对原有类做任何修改，不影响即有功能。个人认为有点类似于在Java中定义static型的工具类和函数，但是在Java中是需要把调用者作为参数传入的，但Kotlin是不需要的。
以本人项目中的代码为例，View类中有一个方法是获取当前view在当前屏幕的位置: getLocationOnScreen()，参数是一个size为2的int数组，以接收横坐标和纵坐标，我们至少要通过两步才能获取到坐标值，如果使用Kotlin扩展函数的话，那我们可以增加一个这样的函数：
```Kotlin
fun View.getViewLocationArr(): IntArray {
    val viewLoc = intArrayOf(0, 0)
    getLocationOnScreen(viewLoc)
    return viewLoc
}
```
在使用的时候，就可以把这个函数当做是View已有的方法去使用：
```Kotlin
var viewLoc = view.getViewLocationArr()
if (viewLoc[0] <= popupContentView.measuredWidth) {
    return false
}
````

###### 扩展函数原理
同样的，我们通过反编译class类文件，得出以下代码：
```Java
@NotNull
public static final int[] getViewLocationArr(@NotNull View $receiver)
{
  Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
  int[] viewLoc = { 0, 0 };
  $receiver.getLocationOnScreen(viewLoc);
  return viewLoc;
}
```
可以看出，Kotlin扩展函数本质上是使用了装饰器模式，只是Kotlin将它作为了一种语法糖，从而在语言级别帮助开发者更好地编写代码。

#### 关于object关键字
object关键字有两个用途，一个是用于声明匿名内部类，例子：
```Kotlin
animator?.addListener(object : AnimatorListenerAdapter() {
    override fun onAnimationStart(animation: Animator?) {

    }

    override fun onAnimationEnd(animation: Animator?) {

    }
})
```

另一个用途是用来做单例声明，例子：
```Kotlin
object InputMethodUtil {
    fun showInputMethod(view: View?) {
        val imm: InputMethodManager = view?.context?.getSystemService(Context.INPUT_METHOD_SERVICE)
                as InputMethodManager
        imm.showSoftInput(view, InputMethodManager.SHOW_IMPLICIT)
    }
}
```
我们在使用的时候，就可以直接调用InputMethodUtil.showInputMethod()方法。
通过反编译class文件，我们可以知道内部是直接使用单例模式来实现。

#### 总结
Kotlin相比于Java，增加了许多新的特性和用法，它更加简洁，省去了一些琐碎的语法，从而帮助开发者更加快速地实现功能。本文只是作为个人在使用kotlin编写代码过程对一些语法的记录，如果想系统地学习Kotlin，建议可以查看Kotlin的的[官方文档](https://kotlinlang.org/docs/reference/android-overview.html)（PS：吐槽一下，Kotlin的中文文档翻译到有点生涩，有些地方不是很理解），后续本人也会发表一些关于Kotlin的文章，更多从知其所以然的角度（例如反编译代码）去认识它，学习它。




