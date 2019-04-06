---
layout: post
title:  "KotinPoet简介"
date:   2019-03-31 17:49:42 +0800
categories: Kotlin
tags: kotin poet
---

在`Java`开发中，我们使用注解开发一些框架的时候，都会用到`JavaPoet`来生成`Java`代码。那么`Kotlin`也有自己的代码生成工具：同样来自业界良心的`Square`的`KotlinPoet`！下面我们来看看怎么使用`KotlinPoet`吧。

在工程中添加依赖就可以使用：

```groovy
compile 'com.squareup:kotlinpoet:1.2.0'
```

第一个例子当然就是`Hello World`了：

```kotlin
val greeterClass = ClassName("", "Greeter")
val file = FileSpec.builder("", "HelloWorld") //指定生成的文件名：'HelloWorld.kt'
    //在'HelloWorld.kt'中生成类：'Greeter'
    .addType(TypeSpec.classBuilder("Greeter") 
        //在类Greeter添加一个主构造函数
        .primaryConstructor(FunSpec.constructorBuilder() 
            //主构造函数的参数为一个名为name的String对象
            .addParameter("name", String::class)
            .build())
        //在类Greeter生成成员变量：name: String
        .addProperty(PropertySpec.builder("name", String::class)
            .initializer("name")
            .build())
        //类Greeter生成无参无返回值函数：greet
        .addFunction(FunSpec.builder("greet")
            //greet的函数的函数体
            .addStatement("println(%P)", "Hello, \$name")
            .build())
        .build())
    //在'HelloWorld.kt'中生成无返回值函数：'main'
    .addFunction(FunSpec.builder("main")
        // 'main'函数的参数
        .addParameter("args", String::class, VARARG)
        // 'main'函数的函数体
        .addStatement("%T(args[0]).greet()", greeterClass)
        .build())
    .build()

file.writeTo(System.out)
```

运行后生成的代码如下：

```kotlin
import kotlin.String

class Greeter(val name: String) {
  fun greet() {
    println("""Hello, $name""")
  }
}

fun main(vararg args: String) {
  Greeter(args[0]).greet()
}
```
> `KotlinPoet`会自动添加`import`

`KotlinPoet`提供了`Builder`模式、链式调用等友好的`API`。支持`Kotlin`的文件（`.kt`）、类
、接口、`Object class`、类型别名（`Type Aliases`）、变量、函数/构造函数、参数以及注解的生成。

## 生成kt文件
`FileSpec.builder()`用来生成`.kt`文件：

```kotlin
val file = FileSpec.builder("com.ezstudio.demo", "HelloWorld")
    .build()

file.writeTo(File("src"))
```

运行后在`com.ezstudio.demo`包下生成一个`HelloWorld.kt`

## 生成类
### 普通类
`TypeSpec.classBuilder()`可以生成一个普通的类：

```kotlin
val helloWorld = TypeSpec.classBuilder("HelloWorld").build()
println(helloWorld)
```

生成：

```kotlin
class HelloWorld
```

### 抽象类
`addModifiers()`方法指定修饰符`KModifier.ABSTRACT`即可：

```kotlin
val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(KModifier.ABSTRACT)
    .build()
```

生成：

```kotlin
abstract class HelloWorld
```
    
### Object类
`TypeSpec.objectBuilder()`可以生成一个`Object`类：

```kotlin
val helloWorld = TypeSpec.objectBuilder("HelloWorld")
    .build()
```

生成：

```kotlin
object HelloWorld
```

### Companion Object
`TypeSpec.companionObjectBuilder()`可以生成一个`Companion Object`：

```kotlin
val companion = TypeSpec.companionObjectBuilder()
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addType(companion)
    .build()
```

生成：

```kotlin
class HelloWorld {
    companion object
}
```

### 匿名内部类
`TypeSpec.anonymousClassBuilder()`用来生成匿名内部类：

```kotlin
val comparator = TypeSpec.anonymousClassBuilder()
    .addSuperinterface(Comparator::class.parameterizedBy(String::class))
    .addFunction(FunSpec.builder("compare")
        .addModifiers(KModifier.OVERRIDE)
        .addParameter("a", String::class)
        .addParameter("b", String::class)
        .returns(Int::class)
        .addStatement("return %N.length - %N.length", "a", "b")
        .build())
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addFunction(FunSpec.builder("sortByLength")
        .addParameter("strings", List::class.parameterizedBy(String::class))
        .addStatement("%N.sortedWith(%L)", "strings", comparator)
        .build())
    .build()
```
生成：

```kotlin
class HelloWorld {
    fun sortByLength(strings: List<String>) {
        strings.sortedWith(object : Comparator<String> {
            override fun compare(a: String, b: String): Int = a.length - b.length
        })
    }
}
```

## 生成变量

### 变量和初始化
变量可以单独使用`PropertySpec.builder()`生成，也可以通过`TypeSpec.BuilderaddProperty()`生成；
`PropertySpec.Builder.initializer()`用于接口初始化变量：

```kotlin
val android = PropertySpec.builder("android", String::class)
    .addModifiers(KModifier.PRIVATE)
    .initializer(""""Oreo v.8.1"""")
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addProperty(android)
    .addProperty("robot", String::class, KModifier.PRIVATE)
    .build()
```

生成：

```kotlin
class HelloWorld {
    private val android: String = "Oreo v.8.1"

    private val robot: String
}
```

### 可变变量和getter/setter
`KoltinPoet`生成的变量默认为不可变`val`。如果需要可变变量`var`，只需要调用`mutable()`接口即可；

```kotlin
val android = PropertySpec.builder("android", String::class)
    .mutable()
    .getter(FunSpec.getterBuilder()
        .addModifiers(KModifier.INLINE)
        .addStatement("return %S", "foo")
        .build())
    .setter(FunSpec.setterBuilder()
        .addParameter("value", String::class)
        .build())
    .build()
```
生成：

```kotlin
var android: kotlin.String
    inline get() = "foo"
    set(value) {
    }
```
上面的例子只给`getter`设置了`inline`，而`setter`则是非`inline`。那么如果给`setter`也加上`inline`，则生成的代码：

```kotlin
inline var android: kotlin.String
    get() = "foo"
    set(value) {
    }
```

### 可空变量
通过`TypeName.copy()`创建一个类型的可空副本:

```kotlin
val java = PropertySpec.builder("java", String::class.asTypeName().copy(nullable = true))
    .mutable()
    .initializer("null")
    .build()
```

生成：

```kotlin
var java: String? = null
```

## 生成函数
`FunSpec.builder()`用来生成函数

### 无返回值的普通函数
比如第一个例子中的`greet`函数：

```kotlin
val greet = FunSpec.builder("greet")
    .addStatement("println(%P)", "Hello, world!")
    .build()
```

### 带返回值的普通函数
使用`returns()`指定返回值的类型：

```kotlin
val greet = FunSpec.builder("greet")
    .returns(String:class)
    .addStatement("return %P", "Hello, world!")
    .build()
```

### 抽象函数
使用`KModifier.ABSTRACT`来生成抽象函数：

```kotlin
val flux = FunSpec.builder("flux")
    .addModifiers(KModifier.ABSTRACT, KModifier.PROTECTED)
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(KModifier.ABSTRACT)
    .addFunction(flux)
    .build()
```

生成：

```kotlin
abstract class HelloWorld {
    protected abstract fun flux()
}
```

### 扩展函数
使用`returns()`指定扩展的类型：

```kotlin
val abs = FunSpec.builder("abs")
    .receiver(Int::class)
    .returns(Int::class)
    .addStatement("return if (this < 0) -this else this")
    .build()
```

生成：

```kotlin
fun Int.abs(): Int = if (this < 0) -this else this
```

### 参数和默认值
可以使用`FunSpec`的`addParameter()`接口添加参数，也可以单独使用`ParameterSpce.builder()`。

`ParameterSpce`的`defaultValue()`接口用来给参数添加默认值：

```kotlin
val bPara = ParameterSpec.builder("b", Int::class)
        .defaultValue("0")
        .build()
        
val print = "a + b = "+"$"+"{ a + b }"

val add = FunSpec.builder("add")
        .addParameter("a", Int::class)
        .addParameter(bPara)
        .addStatement("print(%P)", print)
        .build()
```

生成：

```kotlin
fun add(a: Int, b: Int = 0) {
  print("""a + b = ${ a + b }""")
}
```

## 生成构造函数

### 普通构造函数
`FunSpec`同样可以用来生成构造函数：

```kotlin
val flux = FunSpec.constructorBuilder()
    .addParameter("greeting", String::class)
    .addStatement("this.%N = %N", "greeting", "greeting")
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addProperty("greeting", String::class, KModifier.PRIVATE)
    .addFunction(flux)
    .build()
```

生成：

```kotlin
class HelloWorld {
    private val greeting: String

    constructor(greeting: String) {
        this.greeting = greeting
    }
}
```

### 主构造函数

```kotlin
val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .primaryConstructor(flux)
    .addProperty("greeting", String::class, KModifier.PRIVATE)
    .build()
```

生成：

```kotlin
class HelloWorld(greeting: String) {
    private val greeting: String
    init {
        this.greeting = greeting
    }
}
```

`KotlinPoet`不会自动的整合主构造函数的参数和类中的同名成员变量。必须显式的告诉`KotlinPoet`：

```kotlin
val flux = FunSpec.constructorBuilder()
    .addParameter("greeting", String::class)
    .build()

val helloWorld = TypeSpec.classBuilder("HelloWorld")
    .primaryConstructor(flux)
    .addProperty(PropertySpec.builder("greeting", String::class)
        .initializer("greeting")
        .addModifiers(KModifier.PRIVATE)
        .build())
    .build()
```

生成:

```kotlin
class HelloWorld(private val greeting: String)
```

## 生成Interface
`TypeSpec.interfaceBuilder()`用来生成接口，由于接口中的方法必须是抽象的，因此必须添加`KModifier.ABSTRACT`：

```kotlin
val helloWorld = TypeSpec.interfaceBuilder("HelloWorld")
    .addProperty("buzz", String::class)
    .addFunction(FunSpec.builder("beep")
        .addModifiers(KModifier.ABSTRACT)
        .build())
    .build()
```

生成：

```kotlin
interface HelloWorld {
    val buzz: String

    fun beep()
}
```

## 生成Enums
`TypeSpec.enumBuilder()`用来生成枚举：

```kotlin
val roshambo = TypeSpec.enumBuilder("Roshambo")
    .addEnumConstant("ROCK")
    .addEnumConstant("SCISSORS")
    .addEnumConstant("PAPER")
    .build()
```

生成：

```kotlin
enum class Roshambo {
    ROCK,

    SCISSORS,

    PAPER
}
```

## 生成注解
`TypeSpec.annotationBuilder()`用于注解：

```kotlin
val greeter = TypeSpec.annotationBuilder("Greeter")
        .build()
```

生成：

```kotlin
annotation class Greeter
```

## 使用注解

使用`TypeSpec.Builder.addAnnotation()`来为类添加注解；

使用`FunSpec.Builder.addAnnotation()`来为方法添加注解：

```kotlin
val greeterClass = ClassName("cn.ezstudio.demo", "Greeter")

val helloWorld = TypeSpec.classBuilder("HelloWorld")
        .primaryConstructor(flux)
        .addAnnotation(AnnotationSpec.builder(greeterClass)
                .build())
        .build()
```

生成：

```kotlin
@Greeter
class HelloWorld(greeting: String)
```
     
## 生成别名
`FileSpec.Builder.addTypeAlias()`，用于生成别名：

```kotlin
val fileTable = Map::class.asClassName()
    .parameterizedBy(TypeVariableName("K"), Set::class.parameterizedBy(File::class))
val predicate = LambdaTypeName.get(parameters = *arrayOf(TypeVariableName("T")),
    returnType = Boolean::class.asClassName())
val helloWorld = FileSpec.builder("com.example", "HelloWorld")
    .addTypeAlias(TypeAliasSpec.builder("Word", String::class).build())
    .addTypeAlias(TypeAliasSpec.builder("FileTable<K>", fileTable).build())
    .addTypeAlias(TypeAliasSpec.builder("Predicate<T>", predicate).build())
    .build()
```    

生成：

```kotlin
package com.example

import java.io.File
import kotlin.Boolean
import kotlin.String
import kotlin.collections.Map
import kotlin.collections.Set

typealias Word = String

typealias FileTable<K> = Map<K, Set<File>>

typealias Predicate<T> = (T) -> Boolean
```

## 生成代码块和控制流

`KotlinPoet`可以很方便的生成多行的代码块，比如要生成一段代码：

```kotlin
fun main() {
    var total = 0
    for (i in 0 until 10) {
        total += i
    }
}
```
可以纯文本书写：

```kotlin
val main = FunSpec.builder("main")
    .addCode("""
        |var total = 0
        |for (i in 0 until 10) {
        |    total += i
        |}
        |""".trimMargin())
    .build()
```

也可以借助`addStatement()`来添加普通语句，并通过`beginControlFlow()`和`endControlFlow()`来添加控制语句：

```kotlin
val main = FunSpec.builder("main")
    .addStatement("var total = 0")
    .beginControlFlow("for (i in 0 until 10)")
    .addStatement("total += i")
    .endControlFlow()
    .build()
```

上面例子生成的代码是写死的，不够灵活。如果需要控制`for`循环的启止范围，可以这样做：

```kotlin
private fun computeRange(name: String, from: Int, to: Int, op: String): FunSpec {
  return FunSpec.builder(name)
      .returns(Int::class)
      .addStatement("var result = 1")
      .beginControlFlow("for (i in $from until $to)")
      .addStatement("result = result $op i")
      .endControlFlow()
      .addStatement("return result")
      .build()
}
```

当调用：`computeRange("multiply10to20", 10, 20, "*")`，就会得到：

```kotlin'
fun multiply10to20(): kotlin.Int {
    var result = 1
    for (i in 10 until 20) {
        result = result * i
    }
    return result
}
```

## 换行
通常来说`KotlinPoet`会自动寻找代码中的空格符来判断是否应该换行，来保证生成的代码的可读性。但很多时候，可能并不能得到你理想的格式。比如：

```kotlin
val funSpec = FunSpec.builder("foo")
    .addStatement("return (100..10000).map { number -> number * number }.map { number -> number.toString() }.also { string -> println(string) }")
    .build()
```

生成的代码看起来换行是不对的：

```kotlin
fun foo() = (100..10000).map { number -> number * number }.map { number -> number.toString() }.also 
{ string -> println(string) }
```

如果不想`KotlinPoet`将某些空格符识别为换行，则需要使用`·`符号来替换空格符：

```kotlin
val funSpec = FunSpec.builder("foo")
    .addStatement("return (100..10000).map·{ number -> number * number }.map·{ number -> number.toString() }.also·{ string -> println(string) }")
    .build()
```

生成：

```kotlin
fun foo() = (100..10000).map { number -> number * number }.map { number -> number.toString()
}.also { string -> println(string) }
```

## 占位符
`KotlinPoet`还可以通过占位符来生成代码，支持的占位符包括：

### 1. %S - Strings

```kotlin
val enzo = "enzo"
FunSpec.builder(enzo)
      .returns(String::class)
      .addStatement("return %S", enzo)
      .build()
```

生成：

```kotlin
fun enzo(): String = "enzo"
```

### 2. %P - String模版
如果生成字符串模版，不能使用占位符`%S`，而是使用`%P`：

```kotlin
val stringWithADollar = "Your total is " + "$" + "amount"
val funSpec = FunSpec.builder("printTotal")
    .returns(String::class)
    .addStatement("return %P", stringWithADollar)
    .build()
```

生成:

```kotlin
fun printTotal(): String = "Your total is $amount"
```

如果换成`%S`则成为：

```kotlin
fun printTotal(): String = "Your total is ${'$'}amount"
```

### 3. %T - 类型

```kotlin
FunSpec.builder("today")
    .returns(Date::class)
    .addStatement("return %T()", Date::class)
    .build()
```

生成：

```kotlin
import java.util.Date
fun today(): Date = Date()
```

### 4.%M - 函数/变量

```kotlin
val createTaco = MemberName("com.squareup.tacos", "createTaco")
val isVegan = MemberName("com.squareup.tacos", "isVegan")
val file = FileSpec.builder("com.squareup.example", "TacoTest")
    .addFunction(FunSpec.builder("main")
        .addStatement("val taco = %M()", createTaco)
        .addStatement("println(taco.%M)", isVegan)
        .build())
    .build()
println(file)
```

生成：

```kotlin
package com.squareup.example

import com.squareup.tacos.createTaco
import com.squareup.tacos.isVegan

fun main() {
    val taco = createTaco()
    println(taco.isVegan)
}
```

### 5.%N - 函数名
有时候，需要引用一个还没生成的函数，可以通过传递名称来解决：

```kotlin
val hexDigit = FunSpec.builder("hexDigit")
    .addParameter("i", Int::class)
    .returns(Char::class)
    .addStatement("return (if (i < 10) i + '0'.toInt() else i - 10 + 'a'.toInt()).toChar()")
    .build()

val byteToHex = FunSpec.builder("byteToHex")
    .addParameter("b", Int::class)
    .returns(String::class)
    .addStatement("val result = CharArray(2)")
    .addStatement("result[0] = %N((b ushr 4) and 0xf)", hexDigit)
    .addStatement("result[1] = %N(b and 0xf)", hexDigit)
    .addStatement("return String(result)")
    .build()
```

生成：

```kotlin
fun byteToHex(b: Int): String {
  val result = CharArray(2)
  result[0] = hexDigit((b ushr 4) and 0xf)
  result[1] = hexDigit(b and 0xf)
  return String(result)
}

fun hexDigit(i: Int): Char {
  return (if (i < 10) i + '0'.toInt() else i - 10 + 'a'.toInt()).toChar()
}
```
### 6.%L - 字面量

前面的例子：

```kotlin
private fun computeRange(name: String, from: Int, to: Int, op: String): FunSpec {
  return FunSpec.builder(name)
      .returns(Int::class)
      .addStatement("var result = 1")
      .beginControlFlow("for (i in $from until $to)")
      .addStatement("result = result $op i")
      .endControlFlow()
      .addStatement("return result")
      .build()
}
```

可以换成`%L`来实现：

```kotlin
private fun computeRange(name: String, from: Int, to: Int, op: String): FunSpec {
  return FunSpec.builder(name)
      .returns(Int::class)
      .addStatement("var result = 0")
      .beginControlFlow("for (i in %L until %L)", from, to)
      .addStatement("result = result %L i", op)
      .endControlFlow()
      .addStatement("return result")
      .build()
}
```

`%L`还有更多用法：

#### 相对形式

`CodeBlock.builder().add("I ate %L %L", 3, "tacos")`

生成：`I ate 3 tacos`

#### 指定位置

`CodeBlock.builder().add("I ate %2L %1L", "tacos", 3)`

还是生成：`I ate 3 tacos`

#### 命名形式

```kotlin
val map = LinkedHashMap<String, Any>()
map += "food" to "tacos"
map += "count" to 3
CodeBlock.builder().addNamed("I ate %count:L %food:L", map)
```

还是生成：`I ate 3 tacos`

