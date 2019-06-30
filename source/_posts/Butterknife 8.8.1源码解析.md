---
title: Butterknife 8.8.1源码解析
tags: Android源码解析
categories: Android
abbrlink: 5ce963fb
date: 2017-12-17 18:40:23
---



#### 一、本文需要解决的问题
我研究Butterknife源码的目的是为了解决以下几个我在使用过程中所思考的问题：

1. 在很多文章中都提到Butterknife使用编译时注解技术，什么是编译时注解？
2. 是完全不调用findViewById()等方法了吗？
3. 为什么绑定各种view时不能使用private修饰？
4. 绑定监听事件的时候方法命名有限制吗？

<!--more-->


#### 二、初步分析
基于Butterknife 8.8.1版本。
为了更好地分析代码，我写了一个demo：
MainActivity.java:
``` java
public class MainActivity extends Activity {

    @BindView(R.id.text)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
    }

    @OnClick(R.id.text)
    public void textClick() {
        Toast.makeText(MainActivity.this, "textview clicked", Toast.LENGTH_LONG);
    }
}
```
我们从Butterknife.bind()方法，即方法入口开始分析：
ButterKnife#bind()：
```java
@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
}

private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    // ！！！
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
}

@Nullable @CheckResult @UiThread
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      // ！！！
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
}
```
代码还是比较清晰的，bind()方法的流程：
1. 首先获取当前activity的sourceView，其实就是获取Activity的DecorView，DecorView是整个ViewTree的最顶层View，包含标题view和内容view这两个子元素。我们一直调用的setContentView()方法其实就是往内容view中添加view元素。
2. 然后调用createBinding() --> findBindingConstructorForClass()，重点是
```java
Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
BINDINGS.put(cls, bindingCtor);
```
按照所写的代码，这里会加载一个MainActivity_ViewBinding类，然后获取这个类里面的双参数（Activity， View）构造方法，最后放在BINDINGS里面，它是一个map，主要作用是缓存。在下次使用的时候，就可以从缓存中获取到：
```java
Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
if (bindingCtor != null) {
    if (debug) Log.d(TAG, "HIT: Cached in binding map.");
    return bindingCtor;
}
```

#### 三、关于编译时注解
在上面分析过程中，我们知道最后我们会去加载一个MainActivity_ViewBinding类，而这个类并不是我们自己编写的，而是通过编译时注解（APT - Annotation Processing Tool）的技术生成的。
这一节将会介绍一下这个技术。
###### 1、什么是注解
注解其实很常见，比如说Activity自动生成的onCreate()方法上面就有一个@Override注解
![image.png](http://upload-images.jianshu.io/upload_images/1963233-e412f8720ea36e4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)
- 注解的概念：
能够添加到 Java 源代码的语法元数据。类、方法、变量、参数、包都可以被注解，可用来将信息元数据与程序元素进行关联。
- 注解的分类：
  * 标准注解，如Override， Deprecated，SuppressWarnings等
  * 元注解，如@Retention, @Target, @Inherited, @Documented。当我们要自定义注解时，需要使用它们
  * 自定义注解，表示自己根据需要定义的 Annotation
- 注解的作用：
  * 标记，用于告诉编译器一些信息
  * 编译时动态处理，如动态生成java代码
  * 运行时动态处理，如得到注解信息

###### 2、运行时注解 vs 编译时注解
一般有些人提到注解，普遍就会觉得性能低下。但是真正使用注解的开源框架却很多例如ButterKnife，Retrofit等等。所以注解是好是坏呢？
首先，并不是注解就等于性能差。更确切的说是**运行时注解**这种方式，由于它的原理是java反射机制，所以的确会造成较为严重的性能问题。
但是像Butterknife这个框架，它使用的技术是**编译时注解**，它不会影响app实际运行的性能（影响的应该是编译时的效率）。
一句话总结：
- 运行时注解就是在应用运行的过程中，动态地获取相关类，方法，参数等信息，由于使用java反射机制，性能会有问题；
- 编译时注解由于是在代码编译过程中对注解进行处理，通过注解获取相关类，方法，参数等信息，然后在项目中生成代码，运行时调用，其实和直接运行手写代码没有任何区别，也就没有性能问题了。
这样我们就解决了第一个问题。

###### 3、如何使用编译时注解技术
这里要借助到一个类：AbstractProcessor
```java
public class TestProcessor extends AbstractProcessor  
{    
    @Override  
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv)  
    {  
        // TODO Auto-generated method stub  
        return false;  
    }    
}  
```
重点是process()方法，它相当于每个处理器的主函数main()，可以在这里写相关的扫描和处理注解的代码，他会帮助生成相关的Java文件。后面我们可以具体看一下Butterknife中的使用。

#### 四、进一步分析MainActivity_ViewBinding
我们了解了编译时注解的基本概念之后，我们先看一下MainActivity_ViewBinding类具体实现了什么。
在编写完demo之后，需要先build一下项目，之后可以在build/generated/source/apt/debug/包名/下面找到这个类，如图所示：
![](http://upload-images.jianshu.io/upload_images/1963233-05bfdab2cc7029d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
接上面的分析，到最后会通过反射的方式去调用MainActivity_ViewBinding的构造方法。我们直接看这个类的构造方法：
```java
@UiThread
public MainActivity_ViewBinding(final MainActivity target, View source) {
    this.target = target;

    View view;
    // 1
    view = Utils.findRequiredView(source, R.id.text, "field 'textView' and method 'textClick'");
    // 2
    target.textView = Utils.castView(view, R.id.text, "field 'textView'", TextView.class);
    // 3
    view2131165290 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
        @Override
        public void doClick(View p0) {
            target.textClick();
        }
    });
}
```
###### 1、findRequiredView()
```java
public static View findRequiredView(View source, @IdRes int id, String who) {
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
  }
```
看到这里我们已经解决了第二个问题：**到最后还是会调用findViewById()方法，并没有完全舍弃这个方法**，这里的source代表着在上面代码中传入的MainActivity的DecorView。大家可以尝试一下将Activity转化为Fragment的情况~

###### 2、Util.castView
在这里，我们解决了第三个问题，**绑定各种view时不能使用private修饰，而是需要用public或default去修饰，因为如果采用private修饰的话，将无法通过对象.成员变量方式获取到我们需要绑定的View**。
Util#castView()：
```java
public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
    try {
      return cls.cast(view);
    } catch (ClassCastException e) {
      String name = getResourceEntryName(view, id);
      throw new IllegalStateException("View '"
          + name
          + "' with ID "
          + id
          + " for "
          + who
          + " was of the wrong type. See cause for more info.", e);
    }
}
```
这里直接调用Class.cast强制转换类型，将View转化为我们需要的view（TextView）。

###### 3、
```java
view2131165290 = view;
view.setOnClickListener(new DebouncingOnClickListener() {
    @Override
    public void doClick(View p0) {
        target.textClick();
    }
});
```
这里会生成一个成员变量来保存我们需要绑定的View，重点是下面它会调用setOnClickListener()方法，传入的是DebouncingOnClickListener：
```java
/**
 * A {@linkplain View.OnClickListener click listener} that debounces multiple clicks posted in the
 * same frame. A click on one button disables all buttons for that frame.
 */
public abstract class DebouncingOnClickListener implements View.OnClickListener {
    static boolean enabled = true;

    private static final Runnable ENABLE_AGAIN = new Runnable() {
        @Override public void run() {
            enabled = true;
        }
    };

    @Override 
    public final void onClick(View v) {
        if (enabled) {
            enabled = false;
            v.post(ENABLE_AGAIN);
            doClick(v);
        }
    }

    public abstract void doClick(View v);
}
```
这个DebouncingOnClickListener是View.OnClickListener的一个子类，作用是防止一定时间内对view的多次点击，即防止快速点击控件所带来的一些不可预料的错误。个人认为这个类写的非常巧妙，既完美解决了问题，又写的十分优雅，一点都不臃肿。
这里抽象了doClick()方法，实现代码中是直接调用了target.textClick()，这里解决了第四个问题：**绑定监听事件的时候方法命名是没有限制的，不一定需要严格命名为onClick，也不一定需要传入View参数。**

#### 五、MainActivity_ViewBinding的生成
上文提到，MainActivity_ViewBinding类是通过编译时注解技术生成的，我们找到Butterknife相关的继承于AbstractProcessor的类，ButterKnifeProcessor，我们直接看process()方法：
```java
public final class ButterKnifeProcessor extends AbstractProcessor {
    @Override 
    public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
        // 1
        Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

        for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingSet binding = entry.getValue();

            JavaFile javaFile = binding.brewJava(sdk, debuggable);
            try {
                javaFile.writeTo(filer);
            } catch (IOException e) {
                error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
            }
        }

        return false;
    }
}
```

1、findAndParseTargets()
这个方法的作用是处理所有的@BindXX注解，我们直接看处理@BindView的部分：
```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    // 省略代码
    // Process each @BindView element.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
        // we don't SuperficialValidation.validateElement(element)
        // so that an unresolved View type can be generated by later processing rounds
        try {
            parseBindView(element, builderMap, erasedTargetNames);
        } catch (Exception e) {
            logParsingError(element, BindView.class, e);
        }
    }
    // 省略代码
}

private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
      Set<TypeElement> erasedTargetNames) {
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    // Start by verifying common generated code restrictions.
    boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
        || isBindingInWrongPackage(BindView.class, element);

    // Verify that the target type extends from View.
    TypeMirror elementType = element.asType();
    if (elementType.getKind() == TypeKind.TYPEVAR) {
        TypeVariable typeVariable = (TypeVariable) elementType;
        elementType = typeVariable.getUpperBound();
    }
    Name qualifiedName = enclosingElement.getQualifiedName();
    Name simpleName = element.getSimpleName();
    if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
        if (elementType.getKind() == TypeKind.ERROR) {
            note(element, "@%s field with unresolved type (%s) "
                + "must elsewhere be generated as a View or interface. (%s.%s)",
                BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
        } else {
            error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
                  BindView.class.getSimpleName(), qualifiedName, simpleName);
            hasError = true;
        }
    }

    if (hasError) {
        return;
    }

    // Assemble information on the field.
    int id = element.getAnnotation(BindView.class).value();

    BindingSet.Builder builder = builderMap.get(enclosingElement);
    QualifiedId qualifiedId = elementToQualifiedId(element, id);
    if (builder != null) {
      String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));
      if (existingBindingName != null) {
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }

    String name = simpleName.toString();
    TypeName type = TypeName.get(elementType);
    boolean required = isFieldRequired(element);

    builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required));

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
}
```
代码逻辑是处理获取相关注解的信息，比如绑定的资源id等等，然后通过获取BindingSet.Builder类的实例来创建一一对应的关系，这里有一个判断，如果builderMap存在相应实例则直接取出builder，否则通过getOrCreateBindingBuilder()方法生成一个新的builder，最后调用builder.addField()方法。

后续的话返回到findAndParseTargets()方法的最后一部分：
```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    // bindView()     

    // Associate superclass binders with their subclass binders. This is a queue-based tree walk
    // which starts at the roots (superclasses) and walks to the leafs (subclasses).
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();

      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
      }
    }

    return bindingMap;
}
```
这里会生成一个bindingMap，key为TypeElement，代表注解元素类型，value为BindSet类，通过上述的builder.build()生成，BindingSet类中存储了很多信息，例如绑定view的类型，生成类的className等等，方便我们后续生成java文件。最后回到process方法：
```java
@Override 
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();

      JavaFile javaFile = binding.brewJava(sdk, debuggable);
      try {
        javaFile.writeTo(filer);
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }

    return false;
}
```
最后通过brewJava()方法生成java代码。
这里使用到的是javapoet。javapoet是一个开源库，通过处理相应注解来生成最后的java文件，这里是项目地址[传送门](https://github.com/square/javapoet)，具体技术不再分析。






