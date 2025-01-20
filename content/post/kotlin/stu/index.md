---
title: "Kotlin官方文档学习笔记"
description: Kotlin官方文档学习笔记
date: 2025-01-20T09:53:21+08:00
image:
math:
license:
hidden: false
comments: true
draft: true
tags:
  - 笔记
categories:
  - Kotlin 
---

## 习惯用法

### 创建 DTOs（POJOs/POCOs）

```kotlin
data class Customer(val name: String, val email: String)
```

会为 Customer 类提供以下功能：

* 所有属性的 getter （对于 var 定义的还有 setter）
* equals()
* hashCode()
* toString()
* copy()
* 所有属性的 component1()、 component2()……等等（[参见数据类](https://book.kotlincn.net/text/data-classes.html)）

### 函数的默认实参

```kotlin
 fun foo(a: Int = 0, b: String = "") { …… }
```

### 过滤 list

```kotlin
val positives = list.filter { x -> x > 0 }
``` 

或者可以更短:

```kotlin
val positives = list.filter { it > 0 }
```

> 了解 [Java 与 Kotlin](https://book.kotlincn.net/text/java-to-kotlin-collections-guide.html##%E8%BF%87%E6%BB%A4%E5%85%83%E7%B4%A0)
> 过滤的区别。

### 遍历 map/pair型list

```kotlin
for ((k, v) in map) {
    println("$k -> $v")
}
```

### 检测元素是否存在于集合中

```kotlin
if ("john@example.com" in emailsList) { …… }

if ("jane@example.com" !in emailsList) { …… }
}
```

### 字符串内插

```kotlin
println("Name $name")
```

> 了解 [Java 与 Kotlin 字符串连接](https://book.kotlincn.net/text/java-to-kotlin-idioms-strings.html##%E5%AD%97%E7%AC%A6%E4%B8%B2%E8%BF%9E%E6%8E%A5)
> 的区别。

### 类型判断

```kotlin
    when (x) {
        is Foo -> ……
        is Bar -> ……
        else -> ……
    }
```

### 只读 list

```kotlin
val list = listOf("a", "b", "c")
```

### 只读 map

```kotlin
val map = mapOf("a" to 1, "b" to 2, "c" to 3)
```

### 访问 map 条目

```kotlin
println(map["key"])
map["key"] = value
```

### 遍历 map 或者 pair 型 list
```kotlin
for ((k, v) in map) {
    println("$k -> $v")
}
```
k 与 v 可以是任何适宜的名称，例如 name 与 age。

### 区间迭代
```kotlin
for (i in 1..100) { …… }  // 闭区间：包含 100
for (i in 1..< 100) { …… } // 左闭右开区间：不包含 100
for (x in 2..10 step 2) { …… }
for (x in 10 downTo 1) { …… }
(1..10).forEach { …… }
```

### 延迟属性
```kotlin
val p: String by lazy { // 该值仅在首次访问时计算
    // 计算该字符串
}
```
### 扩展函数

```kotlin
fun String.spaceToCamelCase() { …… }
"Convert this to camelcase".spaceToCamelCase()
```

### 创建单例
```kotlin
object Resource {
    val name = "Name"
}
```

### 对类型安全值使用内联值类
```kotlin
@JvmInline
value class EmployeeId(private val id: String)

@JvmInline
value class CustomerId(private val id: String)
```
如果您不小心混淆了 EmployeeId 和 CustomerId，则会触发编译错误。
> 只有 JVM 后端才需要 @JvmInline 注解。

### 实例化一个抽象类
```kotlin
abstract class MyAbstractClass {
    abstract fun doSomething()
    abstract fun sleep()
}

fun main() {
    val myObject = object : MyAbstractClass() {
        override fun doSomething() {
            // ……
        }

        override fun sleep() { // ……
        }
    }
    myObject.doSomething()
}
```
### if-not-null 缩写

```kotlin
val files = File("Test").listFiles()

println(files?.size) // 如果 files 不是 null，那么输出其大小（size）
```

### if-not-null-else 缩写
```kotlin
val files = File("Test").listFiles()

// For simple fallback values:
println(files?.size ?: "empty") // 如果 files 为 null，那么输出“empty”

// 如需在代码块中计算更复杂的备用值，请使用 `run`
val filesSize = files?.size ?: run { 
    val someSize = getSomeSize()
    someSize * 2
}
println(filesSize)
```
### if null 执行一个语句
```kotlin
val values = ……
val email = values["email"] ?: throw IllegalStateException("Email is missing!")
```

### 在可能会空的集合中取第一元素
```kotlin
val emails = …… // 可能会是空集合
val mainEmail = emails.firstOrNull() ?: ""
```
> 了解 [Java 与 Kotlin 获取第一元素](https://book.kotlincn.net/text/java-to-kotlin-collections-guide.html##%E8%8E%B7%E5%8F%96%E5%8F%AF%E8%83%BD%E4%BC%9A%E7%A9%BA%E7%9A%84%E9%9B%86%E5%90%88%E7%9A%84%E9%A6%96%E6%9C%AB%E5%85%83%E7%B4%A0)的差别。

### if not null 执行代码
```kotlin
val value = ……

value?.let {
    …… // 如果非空会执行这个代码块
}
```

### 映射可空值（如果非空的话）
```kotlin
val value = ……

val mapped = value?.let { transformValue(it) } ?: defaultValue 
// 如果该值或其转换结果为空，那么返回 defaultValue。
```

### 返回 when 表达式
```kotlin
fun transform(color: String): Int {
    return when (color) {
        "Red" -> 0
        "Green" -> 1
        "Blue" -> 2
        else -> throw IllegalArgumentException("Invalid color param value")
    }
}
```
> 类似Java中的`return switch`语句

### try-catch 表达式
```kotlin
fun test() {
    val result = try {
        count()
    } catch (e: ArithmeticException) {
        throw IllegalStateException(e)
    }

    // 使用 result
}
```

### if 表达式
```kotlin
val y = if (x == 1) {
    "one"
} else if (x == 2) {
    "two"
} else {
    "other"
}
```
### 返回类型为 Unit 的方法的构建器风格用法
```kotlin
fun arrayOfMinusOnes(size: Int): IntArray {
    return IntArray(size).apply { 
        fill(-1) 
    }
}
```

### 单表达式函数
```kotlin
fun theAnswer() = 42
```
等价于
```kotlin
fun theAnswer(): Int {
return 42
}
```

单表达式函数与其它惯用法一起使用能简化代码，例如和 when 表达式一起使用：
```kotlin
fun transform(color: String): Int = when (color) {
    "Red" -> 0
    "Green" -> 1
    "Blue" -> 2
    else -> throw IllegalArgumentException("Invalid color param value")
}   
```
### 对一个对象实例调用多个方法 （with）
```kotlin
class Turtle {
    fun penDown()
    fun penUp()
    fun turn(degrees: Double)
    fun forward(pixels: Double)
}

val myTurtle = Turtle()
with(myTurtle) { // 画一个 100 像素的正方形
    penDown()
    for (i in 1..4) {
        forward(100.0)
        turn(90.0)
    }
    penUp()
}
```

### 配置对象的属性（apply）
```kotlin
val myRectangle = Rectangle().apply {
    length = 4
    breadth = 5
    color = 0xFAFAFA
}
```
这对于配置未出现在对象构造函数中的属性非常有用。

### Java 7 的 try-with-resources
```kotlin
val stream = Files.newInputStream(Paths.get("/some/file.txt"))
stream.buffered().reader().use { reader ->
    println(reader.readText())
}
```

### 需要泛型信息的泛型函数
```kotlin
//  public final class Gson {
//     ……
//     public <T> T fromJson(JsonElement json, Class<T> classOfT) throws JsonSyntaxException {
//     ……

inline fun <reified T: Any> Gson.fromJson(json: JsonElement): T = this.fromJson(json, T::class.java)
```

### 交换两个变量
```kotlin
var a = 1
var b = 2
a = b.also { b = a }
```
### 将代码标记为不完整（TODO）
Kotlin 的标准库有一个 TODO() 函数，该函数总是抛出一个 NotImplementedError。 其返回类型为 Nothing，因此无论预期类型是什么都可以使用它。 还有一个接受原因参数的重载：
```kotlin
fun calcTaxes(): BigDecimal = TODO("Waiting for feedback from accounting")
```
IntelliJ IDEA 的 kotlin 插件理解 TODO() 的语言，并且会自动在 TODO 工具窗口中添加代码指示。

## Kotlin基础

### 类型

#### 基本类型

##### 数字

Kotlin 的基本数值类型包括
* Byte
* Short
* Int
* Long
* Float
* Double 

不同于 Java 的是，字符不属于数值类型，是一个独立的数据类型。

| 类型    | 大小  | 最小值                         | 最大值                       |
|-------|-----|:----------------------------|---------------------------|
| Byte  | 8   | -128                        | 127                       |
| Short | 16  | -32768                      | 32767                     |
| Int   | 32  | -2,147,483,648              | 2,147,483,647             | 
| Long  | 	64 | 	-9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |

```kotlin
val one = 1 // Int
val threeBillion = 3000000000 // Long
val oneLong = 1L // Long
val oneByte: Byte = 1

val pi = 3.14 // Double
// val one: Double = 1 // 错误：类型不匹配
val oneDouble = 1.0 // Double

val e = 2.7182818284 // Double
val eFloat = 2.7182818284f // Float，实际值为 2.7182817
```
如需将数值转换为不同的类型，请使用[显式转换](https://book.kotlincn.net/text/numbers.html##%E6%98%BE%E5%BC%8F%E6%95%B0%E5%AD%97%E8%BD%AC%E6%8D%A2)。
```kotlin
fun main() {
    fun printDouble(d: Double) { print(d) }

    val i = 1    
    val d = 1.0
    val f = 1.0f 

    printDouble(d)
//    printDouble(i) // 错误：类型不匹配
//    printDouble(f) // 错误：类型不匹配
}
```
###### 数字字面常量
数值常量字面值有以下几种:

* 十进制: 123
* Long 类型用大写 L 标记: 123L
* 十六进制: 0x0F
* 二进制: 0b00001011
* 默认 double：123.5、123.5e10
* Float 用 f 或者 F 标记: 123.5f

```kotlin
val oneMillion = 1_000_000
val creditCardNumber = 1234_5678_9012_3456L
val socialSecurityNumber = 999_99_9999L
val hexBytes = 0xFF_EC_DE_5E
val bytes = 0b11010010_01101001_10010100_10010010
```
> 注意: 不支持八进制

##### 布尔
Boolean

布尔值的内置运算有：

* ||——析取（逻辑或）
* &&——合取（逻辑与）
* !——否定（逻辑非）

##### 字符

Char 

字符字面值用单引号括起来: '1'。

特殊字符可以以转义反斜杠 \ 开始。 支持这几个转义序列：
* \\t ——制表符
* \\b ——退格符
* \\n ——换行（LF）
* \\r ——回车（CR）
* \\' ——单引号
* \\" ——双引号
* \\\ ——反斜杠
* \\$ ——美元符

如果字符变量的值是数字，那么可以使用 digitToInt() 函数将其显式转换为 Int 数字。

> 在JVM 平台，当需要可空引用时字符会是装箱的 Java 类，类似于数字。 装箱操作不保留同一性。

##### 字符串
String 

字符串的元素——字符可以使用索引运算符访问: s[i]。 可以使用 for 循环遍历这些字符：
```kotlin
fun main() {
    val str = "abcd" 
//sampleStart
    for (c in str) {
        println(c)
    }
//sampleEnd
}
```
字符串是不可变的。 一旦初始化了一个字符串，就不能改变它的值或者给它赋新值。 所有转换字符串的操作都以一个新的 String 对象来返回结果，而保持原始字符串不变：
```kotlin
fun main() {
//sampleStart
    val str = "abcd"

    // 创建并输出一个新的 String 对象
    println(str.uppercase())
    // ABCD

    // 原始字符串保持不变
    println(str) 
    // abcd
//sampleEnd
}
```
> 在大多数情况下，优先使用字符串模板或多行字符串而不是字符串连接。

###### 多行字符串
多行字符串可以包含换行以及任意文本。 它使用三个引号（"""）分界符括起来，内部没有转义并且可以包含换行以及任何其他字符：
```kotlin
val text = """
    for (c in "foo")
        print(c)
"""

// 如需删掉多行字符串中的前导空格，请使用 trimMargin() 函数：
val text = """
|Tell me and I forget.
|Teach me and I remember.
|Involve me and I learn.
|(Benjamin Franklin)
    """.trimMargin()
```
默认以竖线 | 作为边界前缀，但你可以选择其他字符并作为参数传入，比如 trimMargin(">")。

##### 字符串模板
```kotlin
fun main() {
//sampleStart
    val i = 10
    println("i = $i") 
    // i = 10

    val letters = listOf("a","b","c","d","e")
    println("Letters: $letters") 
    // Letters: [a, b, c, d, e]

//sampleEnd
```

##### 数组
数组是一种保存固定数量相同类型或其子类型的值的数据结构。 Kotlin 中最常见的数组类型是对象类型数组，由 Array 类表示。

> 如果在对象类型数组中使用原生类型，那么会对性能产生影响，因为原生值都装箱成了对象。 为了避免装箱开销，请改用原生类型数组。
