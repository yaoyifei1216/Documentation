---
title: Kotlin快速入门
tags: 
 - kotlin
categories:
 - 笔记
---

# Kotlin快速入门

> 第一行代码——Android（第3版） 

## 变量

kotlin使用var和val关键字用来修饰两种类型的变量

val:声明的变量不可变,类似于java的final

var:声明一个可变变量

区别于java要在变量前声明类型,kotlin因为其类型推导机制,可以实现仅仅用两个关键字就可以实现变量的声明

类型推导机制:如果要把一个整型变量赋值给a,那么a肯定就是整型变量

注意:类型推导机制并不是完美的,如果对一个变量延迟赋值的话,这个时候就需要显式的声明变量类型了

与java不同的是,kotlin中没有基本的数据类型,全部使用了对象数据类型,如使用Int而不是int

> Java中final关键字没有被合理使用的问题:
> 在Java中，除非你主动在变量前声明了final关键字，否则这个变量就是可变的。然而这并不是一件好事，当项目变得越来越复杂，参与开发的人越来越多时，你永远不知道一个可变的变量会在什么时候被谁给修改了，即使它原本不应该被修改，这就经常会导致出现一些很难排查的问题。
> 那么我们应该什么时候使用val，什么时候使用var呢？这里我告诉你一个小诀窍，就是永远优先使用val来声明一个变量，而当val没有办法满足你的需求时再使用var。这样设计出来的程序会更加健壮，也更加符合高质量的编码规范。

## 函数

```kotlin
fun methodName(param1: Int, param2: Int): Int {
    return 0
}
```

fun:关键字,声明函数必须要有的

methodName:声明的函数名称,depends on you,最好做到见名知意

(param1: Int, param2: Int):函数参数,可为空,可多个,参数声明:参数名: 参数类型

Int:返回值类型,如果不写就是没有返回值

大括号里就是方法体了

```kotlin
private fun maxValue(a: Int, b: Int): Int {
    return max(a, b)//返回两个数较大的一个
}
```

一个kotlin语法糖:当一个函数中只有一行代码时，Kotlin允许我们不必编写函数体，可以直接将唯一的一行代码写在函数定义的尾部，中间用等号连接即可。

比如我们刚才编写的maxValue()函数就只有一行代码，于是可以将代码简化成如下形式：

```kotlin
private fun maxValue(a: Int, b: Int): Int = max(a, b)

//由于类型推导机制,这样写也是可以的
private fun maxValue(a: Int, b: Int) = max(a, b)
```

## 程序的逻辑控制

### if条件语句

Kotlin中的if语句和Java中的if语句几乎没有任何区别,下面的例子是max函数的具体实现

```kotlin
private fun maxValue1(a: Int, b: Int): Int {
    var value = 0
    if (a > b) {
        value = a
    } else{
        value = b
    }
    return value
}
```

Kotlin中的if语句相比于Java有一个额外的功能，它是可以有返回值的，返回值就是if语句每一个条件中最后一行代码的返回值。因此，上述代码就可以简化成如下形式：

```kotlin
private fun maxValue2(a: Int, b: Int): Int {
    val value = if (a > b) {
        a
    } else {
        b
    }
    return value
}

//因为上面提到的语法糖,只有一行代码可以不用写函数体,因此该方法还可以进一步简化
private fun maxValue2(a: Int, b: Int) = if (a > b) {
        a
    } else {
        b
    }

//好像不太好看
private fun maxValue2(a: Int, b: Int) = if (a > b) a else b
```

### When选择语句

Kotlin中的when语句有点类似于Java中的switch语句，但它又远比 switch语句强大得多

when语句允许传入一个任意类型的参数，然后可以在when的结构体中定义一系列的条件，格式是：

```
匹配值 -> { 执行逻辑 }
```

我们准备编写一个查询考试成绩的功能，输入一个学生的姓名，返回该学生考试的分数。

如果用上一小节学习的if语句来实现这个功能,代码如下:

```kotlin
fun getScore(name: String) = if (name == "Tom") {
    86
} else if (name == "Jim") {
    77
} else if (name == "Jack") {
    95
} else if (name == "Lily") {
    100
} else {
    0
}
```

实在有点冗余,如果使用when是这样的:

```kotlin
fun getScore(name: String) = when (name) {
    "Tom" -> 86
    "Jim" -> 77
    "Jack" -> 95
    "Lily" -> 100
    else -> 0
}

//when语句还有一种不带参数的用法，虽然这种用法可能不太常用，但有的时候却能发挥很强的扩展性。
fun getScore(name: String) = when {
    name == "Tom" -> 86
    name == "Jim" -> 77
    name == "Jack" -> 95
    name == "Lily" -> 100
    else -> 0
}
```

可以看出when和if一样是可以有返回值的

如果执行逻辑只有一行的话,大括号可以省略,上述的例子可以体现这一点

除了精确匹配之外，when语句还允许进行类型匹配。什么是类型匹配呢？举个例子。定义一个checkNumber()函数，如下所示：

```kotlin
fun checkNumber(num: Number) {
    when (num) {
        is Int -> println("number is Int")
        is Double -> println("number is Double")
        else -> println("number not support")
    }
}
```

Number类型是Kotlin内置的一个抽象类，像Int、Long、Float、Double等与数字相关的类都是它的子类

is关键字类似于instanceof关键字

### while和for循环语句

Kotlin也提供了while循环和for循环，其中while循环不管是在语法还是使用技巧上都和Java中的while循环没有任何区别，重点是for

Kotlin在for循环方面做了很大幅度的修改，Java中最常用的for-i循环在Kotlin中直接被舍弃了，而Java中另一种for-each循环则被Kotlin进行了大幅度的加强，变成了for-in循环

首先需要了解区间的概念:kotlin中使用关键字.. 声明一个**闭区间**,0..10 表示[0,10]

Kotlin中可以使用until关键字来创建一个左闭右开的区间,0 until 10表示[0,10)

```kotlin
//循环打印0到10
fun forin() {
    for (i in 0..10){
        println(i)
    }
}
//循环打印0到9
fun forin() {
    for (i in 0 until 10){
        println(i)
    }
}
```

默认情况下，for-in循环每次执行循环时会在区间范围内递增1，相当于Java for-i循环中i++的效果，而如果你想跳过其中的一些元素，可以使用step关键字：

```kotlin
//循环打印0到10中的偶数 相当与i = i +　2
fun forin() {
    for (i in 0..10 step 2){
        println(i)
    }
}
```

.. 和until关键字都要求区间的左端必须小于等于区间的右端，使用downTo可以创建一个降序的闭区间

```kotlin
//循环打印10到0
fun forin() {
    for (i in 10 downTo 0) {
        println(i)
    }
}
```

for-in循环并没有传统的for-i循环那样灵活，但是却比for-i循环要简单好用得多，而且足够覆盖大部分的使用场景。

如果有一些特殊场景使用for-in循环无法实现的话，我们还可以改用while循环的方式来进行实现。

## 面向对象

### 类与对象

声明一个Person类,结构如下：

```kotlin
class Person {
    var name = ""
    var age = 0

    fun eat() {
        println(name + " is eating. He is " + age + " years old.")
    }

    override fun toString(): String {
        println("Person(name='$name', age=$age)")
        return "Person(name='$name', age=$age)"
    }

}
```

声明一个Person对象的实例,并给其赋值,随后使用其的方法

```kotlin
var p = Person()
    p.age = 23
    p.name = "yaoyifei"
    p.toString()
    p.eat()

输出结果:
Person(name='yaoyifei', age=23)
yaoyifei is eating. He is 23 years old.
```

面向对象编程最基本的用法:

先将事物封装成具体的类，然后将事物所拥有的属性和能力分别定义成类中的字段和函数，接下来对类进行实例化，再根据具体的编程需求调用类中的字段和方法。

### 继承

下面学习一下继承,声明一个Student类,并继承Person类

分两步,第一步需要先使Person可以继承

Java中一个类本身就可以被继承为,但是在Kotlin中任何一个非抽象类默认都是不可以被继承的，相当于Java中给类声明了final关键字。之所以这么设计，其实和val关键字的原因是差不多的，因为类和变量一样，最好都是不可变的，而一个类允许被继承的话，它无法预知子类会如何实现，因此可能就会存在一些未知的风险。

做法很简单,在Person类之前加上open修饰符就好了  **open class Person**

第二步，要让Student类继承Person类。在Java中继承的关键字是extends，而在Kotlin中变成了一个冒号，写法如下：

```kotlin
class Student : Person() {
    var sno = ""
    var grade = 0
}
```

通过这两步就完成了Student类的继承,这里重点说一下Person()后面的这一对括号,涉及到Kotlin的构造函数

### 构造函数

任何一个面向对象的编程语言都会有构造函数的概念，Kotlin中也有，但是Kotlin将构造函数分成了两种：主构造函数和次构造函数。

主构造函数将会是你最常用的构造函数，每个类默认都会有一个不带参数的主构造函数

当然也可以显式地给它指明参数像这样:

```kotlin
class Student(val sno: String, val grade: Int) : Person() {
}
```

这里我们将主构造函数里加入了两个字段,那么在对Student类进行实例化的时候，必须传入构造函数中要求的所有参数。比如：

```kotlin
val student = Student("a123", 5)
```

主构造函数没有函数体，如果想在主构造函数中编写一些逻辑，可以写在Kotlin给我们提供的一个init结构体里面,像这样:

```Kotlin
class Student(val sno: String, val grade: Int) : Person() {
    init {
        println("sno is " + sno)
        println("grade is " + grade)
    }
}
```

Java中继承特性中的一个规定，子类中的构造函数必须调用父类中的构造函数，这个规定在Kotlin中也要遵守。

所以Person()的括号就是表示Student调用Person类的无参构造函数

当然了如果Person类的主构造函数里有参数,这里也要相应的提供参数,举个例子:

```Kotlin
open class Person(val name: String, val age: Int) {
    ...
}

class Student(val sno: String, val grade: Int, name: String, age: Int) : Person(name, age) {
    ...
}
```

> 注意:在Student类的主构造函数中增加name和age这两个字段时，不能再将它们声明成val，因为在主构造函数中声明成val或者var的参数将自动成为该类的字段，这就会导致和父类中同名的name和age字段造成冲突。因此，这里的name和age参数前面我们不用加任何关键字，让它的作用域仅限定在主构造函数当中即可

先小结一下:

1. Kotlin将构造函数分成了两种：主构造函数和次构造函数
2. Kotlin主构造函数没有函数体，如果想在主构造函数中编写一些逻辑，写在init结构体里面
3. Kotlin中子类中的构造函数必须调用父类中的构造函数

综合以上几点就可以理解为什么要加上()了

