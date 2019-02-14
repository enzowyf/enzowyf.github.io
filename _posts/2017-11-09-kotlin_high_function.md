---
layout: post
title:  "Kotlin之高阶函数"
date:   2017-11-09 15:54:42 +0800
categories: Android开发
cover: './../assets/img/post/kotlin_banner.jpg'
tags: kotlin
---

## 高阶函数的基本概念
高阶函数是将函数用作参数或返回值的函数。类似于数学中的高阶函数```f(g(x))```。看看官方的例子：

```kotlin
fun <T> lock(lock: Lock, body: () -> T): T {
    lock.lock()
    try {
        return body()
    }
    finally {
        lock.unlock()
    }
}
```

上面的代码中：```body``` 拥有函数类型：```() -> T```， 所以它应该是一个不带参数并且返回```T``` 类型值的函数。 它在 try-代码块内部调用、被 lock 保护，其结果由```lock()```函数返回。

高阶函数通常与Lambda表达式一起使用：

```kotlin
val result = lock(lock, { sharedResource.operation() })
```

在 Kotlin 中有一个约定，如果函数的最后一个参数是一个函数，并且你传递一个 lambda 表达式作为相应的参数，你可以在圆括号之外指定它：

```kotlin
val result = lock (lock) {
    sharedResource.operation()
}
```

## 常用高阶函数
这里收集了Kotlin标准库里面常用的高阶函数的定义和用法，以备日后查询：

### 1. apply
```kotlin
/**
 * Calls the specified function [block] with `this` value as its receiver and returns `this` value.
 */
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }
```
调用某对象的```apply```函数，在函数范围内，可以通过```this```调用该对象的任意方法，并返回该对象，```this```可以不显示的写处理。例如：

```kotlin
//生成一个TextView实例，并设置相应的属性后返回给textView
val textView = TextView(this).apply {
      //this.text = "Hello World"
      text = "Hello World"
      setTextColor(Color.BLUE)
}

//等同于
val textView = TextView(this)
textView.text = "Hello World"
textView.setTextColor(Color.BLUE)
```

### 2. let
```kotlin
/**
 * Calls the specified function [block] with `this` value as its argument and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R = block(this)
```

 ```let```函数默认当前这个对象作为闭包的it参数，返回值是函数里面最后一行，或者指定```return```。例如上述的例子：

```kotlin
//let函数默认返回最后一行
val textView = TextView(this)
val text: String = textView.let {
      it.text = "Hello World"
      it.setTextColor(Color.BLUE)
      it.text.toString()
}
print(text)//结果为Hello World
```

### 3.also
```kotlin
/**
 * Calls the specified function [block] with `this` value as its argument and returns `this` value.
 */
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.also(block: (T) -> Unit): T { block(this); return this }
```

```also```可以看作是```apply```和```let```的结合：默认当前这个对象作为闭包的```it```参数，最后返回自己，例如：

```kotlin
//生成一个TextView实例，并设置相应的属性后将it返回给textView
val textView = TextView(this).also {
     it.text = "Hello World"
     it.setTextColor(Color.BLUE)
}
```

### 4.run
```run```有两种，一种是```apply```和```let```的另一种结合体：在函数范围内，可以任意调用该对象的任意方法，返回函数里面最后一行，或者指定```return```

```kotlin
/**
 * Calls the specified function [block] with `this` value as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R = block()
```

例子：


```kotlin
//跟let相比，不需要it作为参数了
val textView = TextView(this)
val text: String = textView.run {
//this.text = "Hello World"
      text = "Hello World"
      setTextColor(Color.BLUE)
      text.toString()
}
print(text)//结果为Hello World
```

另一种```run```是：提供```() -> R ```的转换，并返回最后一行，或者指定```return```

```kotlin
/**
 * Calls the specified function [block] and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <R> run(block: () -> R): R = block()
```


例如：

```kotlin
val textView = TextView(this).apply {
      val text: String = run {
//this.text = "Hello World"
        text = "Hello World"
        setTextColor(Color.BLUE)
        text.toString()
      }
      print(text)//结果为Hello World
}
```

### 5.with
```kotlin
/**
 * Calls the specified function [block] with the given [receiver] as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R = receiver.block()
```
```with```函数接收两个参数，一个是调用者，第二个为一个函数，在函数库内可以通过```this```指代调用者（第一个参数）来调用，返回值为函数块的最后一行或指定```return```。例如：

```kotlin
val textView = TextView(this)
    val text: String = with(textView) {
      //this.text = "Hello World"
      text = "Hello World"
      setTextColor(Color.BLUE)
      text.toString()
}
print(text)//结果为Hello World
```
 

### 6.repeat
```kotlin
/**
 * Executes the given function [action] specified number of [times].
 *
 * A zero-based index of current iteration is passed as a parameter to [action].
 */
@kotlin.internal.InlineOnly
public inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0..times - 1) {
        action(index)
    }
}
```

这个没什么好解释的，就是重复某操作指定次数，例如

```kotlin
repeat(10) {print(text)}//打印10次Hello World
```

### 7.takeIf
```kotlin
/**
 * Returns `this` value if it satisfies the given [predicate] or `null`, if it doesn't.
 */
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.takeIf(predicate: (T) -> Boolean): T? = if (predicate(this)) this else null
```
```takeIf ```接收一个字面值为```predicate```的函数```(T) -> Boolean```，这个函数的参数为```T ```（即```takeIf ```的调用者），返回Boolean。

takeIf的作用就是字面意思，如果（if）```predicate```为真，则采用（take）调用者。例如：

```kotlin
val flag = 1
var result = "Hello World".takeIf {
      flag == 1
}

println(result)//因为flag == 1，所以这里可以打印出Hello World
    
//等同于
var result: String? = null
if (flag == 1) {
  result = "Hello World"
}
```

实际上个人觉得如果是新创建对象还是使用if的写法更妥当，因为通过反编译```takeIf ```可以看到，```"Hello World"```这个对象已经被创建，如果```predicate ```不成立，这个对象没有被赋予```result ```就被浪费了。而对于已经存在的旧对象做判断，则可以使用```takeIf ```。



### 8.takeUnless
```kotlin
/**
 * Returns `this` value if it _does not_ satisfy the given [predicate] or `null`, if it does.
 */
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.takeUnless(predicate: (T) -> Boolean): T? = if (!predicate(this)) this else null
```
跟```takeIf ```相反，```predicate```不成立，才采用调用者，例如：

```kotlin
    val flag = 1
    var result = "Hello World".takeUnless {
      flag == 1
    }

    println(result)//因为flag == 1，所以这里会抛出空指针
```

### 9.lazy
```kotlin
public fun <T> lazy(initializer: () -> T): Lazy<T>
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T>
public fun <T> lazy(lock: Any?, initializer: () -> T): Lazy<T>
```
这是一个提供延时加载能力的代理属性，接受一个 lambda 并返回一个```Lazy <T>```实例的函数，返回的实例可以作为实现延迟属性的委托： 第一次调用 ```get()``` 会执行已传递给```lazy()``` 的 lambda 表达式并记录结果， 后续调用 ```get()```只是返回记录的结果。例子：

```kotlin
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}

//第一次访问lazyValue，会打印"computed!"和"Hello"
println(lazyValue)

//第二次访问只打印结果"Hello"
println(lazyValue)
```
默认情况下，对于 lazy 属性的求值是同步锁的（synchronized）：该值只在一个线程中计算，并且所有线程会看到相同的值。如果初始化委托的同步锁不是必需的，这样多个线程可以同时执行，那么可以将```LazyThreadSafetyMode.PUBLICATION```作为参数传递给```lazy()```函数。 而如果你确定初始化将总是发生在单个线程，那么你可以使用```LazyThreadSafetyMode.NONE```模式， 它不会有任何线程安全的保证和相关的开销。

### 10.use
```kotlin
public inline fun <T : Closeable?, R> T.use(block: (T) -> R): R {
    var closed = false
    try {
        return block(this)
    } catch (e: Exception) {
        closed = true
        try {
            this?.close()
        } catch (closeException: Exception) {
        }
        throw e
    } finally {
        if (!closed) {
            this?.close()
        }
    }
}
```

用来简化```Closeable```的操作，例如```close```、```try/catch```，使用统一的模板:

```kotlin
BufferedReader(FileReader("test.txt")).use {
    var line: String?
    while (true) {
        line = it.readLine() ?: break
        println(line)
    }
}
```

### 11.filter
过滤所有符合给定函数条件的元素。

```kotlin
val list = listOf(1,2,3,4,5,6)
val oddList = list.filter { it % 2 == 1 }
println(oddList)//结果为[1,3,5]

//filter操作等同于
for (i in list) {
    if (i % 2 == 1) {
        oddList.add(i)
    }
}
```

### 12.map
返回一个每一个元素根据给定的函数转换所组成的List。

```kotlin
val list = listOf(1,2,3,4,5,6)
val doubleList = list.map { it * 2 }
println(doubleList)//结果为[2,4,6,8,10,12]

//map操作等同于
for (i in list) {
    doubleList.add(i * 2)
}
```

### 13.forEach
遍历所有元素，并执行给定的操作。

```kotlin
list.forEach { println(it) }
//等同于
for (i in list) {
    println(i)
}
```