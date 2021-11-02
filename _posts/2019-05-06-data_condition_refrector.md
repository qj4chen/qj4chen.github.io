---
layout: post
title: 重构 - Reorganize Data and Control Structure
categories:
- Refactoring
- Scala
- Martin Fowler
---

> 这是重构的第四篇文章，主要讲解对于数据结构的重建问题。着重介绍了子封装方法代替字段、数据对象替代数据值、数据对象值和数据对象引用的转换、数组数据的拆分、数组数据对象的封装，也介绍了对于单向和双向数据绑定的处理、字段取代子类（组合而非继承）等惯用的数据组织习惯。在第二部分，讲解了对于控制结构的重构，包括表达式清晰化、控制结构优化、卫语句，最后介绍了空对象和多态对于减少控制结构样板代码的作用。


# 数据的封装

## Self Encapsulate Field

略，Scala 具有统一对象访问，不允许访问字段，访问的都是自动编译的同名函数。

## Replace Data Value with Object

下面介绍了数据值和对象值的区别，以及你为什么应该使用对象值而不是数据值。对于字段 a 而言，它是一个字符串，但是承担了过多的责任。方法 getSchoolAndCity 使用了复杂的逻辑来从中抽取合理的信息 —— 使用了脆弱的协议，基于一个带有空格的下划线表示分割，这意味着，当前后字段中出现此下划线时，拆分将错误，如果没有下划线时，拆分将错误。

修改方法就是使用一个数据对象，在这里，很自然的将其表示成为两个字段。当这个对象执行越来越多的操作，使用 Move Method 和 Move Field 手法将很多函数迁移到其对象内部时，这时候面向对象编程的优点才会淋漓尽致的表现出来。比如这里的 sayHello。对于命令式，想象一下如果要调用 1000 个入参基本类型是如何的痛苦吧。

### 一点忠告

> 新手，尤其是 OOP 的新手，很愿意使用数据值和脆弱的约定，这是一种难以扩展的、易碎的错误的选择。从很大程度上来说，这是一个意识的问题。他们可能从 C 过来，按照计算机的思维思考，所有的封装最后总是转换成命令执行 —— 但这不意味着开发者要忍受这种痛苦。实际上，人尽管可以以非常大量、快速的方式接受信息，但是得益于注意系统，我们可以维持对于极少部分内容的深度关注。知识是无穷尽的，领域有专攻，当一个对象具有更加明确的责任边界，事情处理起来会方便很多，这意味着你可以不用在自己代码中扮演上帝，当然，如果你没有承受过过程式的上万行代码之苦的话。


```scala
object ReplaceDataValueWithObject extends App {
  val a = "Central China Normal University | Wuhan"
  def getSchoolAndCity(in:String):(String,String) = {
    val array = in.split("\\|").map(_.trim)
    val school = array.headOption match {
      case Some(i) => if (i.isEmpty) "NoSchool" else i
      case None => "NoSchool"
    }
    val city = if (array.length < 2) "NoCity" else {
      val city = array.reverse.head
      if (city.isEmpty) "NoCity" else city
    }
    (school,city)
  }
  def sayHello(school:String, city:String): Unit = {
    println(s"I came from $school, it is in $city")
  }
  println(getSchoolAndCity(a))
  sayHello(getSchoolAndCity(a)._1,getSchoolAndCity(a)._2)

  val b = Place("Central China Normal University", "Wuhan")
  println(b)
  b.sayHello()
}
```

```scala
case class Place(school:String, city:String) {
  def sayHello(): Unit = {
    println(s"I came from $school, it is in $city")
  }
  override def toString: String = s"($school, $city)"
  override def hashCode(): Int = super.hashCode()
  //对于值对象，需要重写 equals 和 hashCode 进行比较
  override def equals(obj: Any): Boolean = obj match {
    case o: Place =>
      if (this.school == o.school && this.city == o.city) true
      else false
    case _ => false
  }
}
```

### 一个教训

我曾经在写一个心理学 GUI 程序的时候 —— 使用的是自己的框架，因为偷懒，所以没有涉及数据收集步骤。所有的数据收集都是通过遍历日志一行一行进行的。开始很轻松，只用对每行进行分析，如果满足条件，那么用正则表达式提取即可，但是慢慢的，问题出现了，因为当初定义打印到日志中的格式改变了，逐渐项目变得非常复杂，解析日志的程序经常崩溃报错，或者收集不到数据。

在同一个程序中，我还将结果使用 csv 输出，并且在用户为 csv 添加了一列后，对新的 csv 进行重新处理。这是噩梦的源泉，读取一个文档可能遇见各种的奇怪的错误，尤其是一个被用户编辑的文档，此外 Excel 也总是自作聪明的篡改 CSV 的数据呈现格式。更有甚者，在 csv 中间插入一列这个需求都难以实现 —— 因为 csv 是 log 文件直接解析的，为 csv 输出添加功能就不得不修改 log 解析程序，一行一行找对应的列，在合适的位置打印输出。

这一切的噩梦都来自于使用了该死的文本数据。在后来的程序中，替换成了 Java 的二进制序列对象，短短几行代码，基本上再也没有出过问题。


## Change Value to Reference
## Change Reference to Value

注意一个基本值转换成为对象后，其还存在一个引用和不可变的问题。对于简单的、不怎么修改的对象，使用不可变的对象值更有优点，尤其是对于多线程并发调用来说。这个时候，唯一需要做的操作就是重写 equals 和 hashCode 方法来提供对于两个不可变对象的比较的方法。

显然，硬币是两面的。在一些时候，可能我们需要可变的对象引用，而不是不可变的对象值，其差别一般表现在是否含有较多的非 final 字段（对于 Scala 而言则是非 val 的 var）。这时候，如果是从之前的基本值重构过来的话，可能我们需要通过工厂来达到引用的目的。


```scala
object PlaceFactory {
  val buffer: mutable.Buffer[Place] = mutable.Buffer[Place]()
  def getPlace(school:String, city:String): Place = {
    buffer.find(p => p.school == school && p.city == city) match {
      case None =>
        val nPlace = Place(school, city)
        buffer.append(nPlace)
        nPlace
      case Some(p) => p
    }
  }
}

val a1 = Place("a","b")
val a2 = Place("a","b")
println(a1 == a2)
println(a1.hashCode(), a2.hashCode())

val c = PlaceFactory.getPlace("CCAU","Wuhan")
val d = PlaceFactory.getPlace("CCAU","Wuhan")
println(c.hashCode(), d.hashCode())
```

注意上文的 a1 和 a2，它们相等，这是因为 equal 判断我们重写了，但是 hashCode 不同，这是因为其属于不同的方法。而 c 和 d 也相等，并且 hashCode 相同，这是因为它们代表了相同的引用。

# 集合的封装

## Replace Array with Object
## Encapsulate Collection

对于集合而言，如果是那种 MATLAB 风格浓郁的每个矩阵的每列代表不同的字段，如果这种风格迁移到了 Java 上，这会是一个让人头痛的问题。典型的处理方式是，使用 ArrayList 替代 Object[]，此外，将这个数组对象包装成一个字段，控制外部对它的访问，一个更好的主意是，当别人对其访问的时候，转换成只读的数据结构，不允许对其进行操作，比如下面的 languages 字段在 get 的时候被转成了 Array，并且对此 Array 的操作不影响 languages 本身。

```scala
class People(val name:String, var age:Int) {
  private val languages: mutable.Buffer[String] = mutable.Buffer("Scala")
  def getLanguages:Array[String] = languages.toArray
}
```

# 封装的可获得性

## Unidirectional Association and Bidirectional Association

单向和双向绑定在某些场景下，都是有道理的。当程序因为单向绑定而难以操作 —— 必须费尽心机获取其另一半引用的时候，就应该重构为双向绑定，反之亦然。

在处理双向绑定的时候，要注意值为空的问题，这是造成 Java 指针为空错误的最大的罪魁祸首。Scala 可以使用 Map 的 API，或者 Option 包装函数返回值，使用 match 解析结果保证没有空对象。在现代的语言中，通常使用 .? 的方式获取对象的一个方法，如果存在的话。

很显然，如果你不原因这么做，可以在 getXXX 方法中使用一个 Fake Object，来避免这种错误。

此外，获得的可能是一个集合。单向和双向数据绑定是 DDD 的一个重要课题，在 Hibernate(JPA) 等 ORM 框架中经常会处理到。JPA 一个臭名昭著的问题是，当对一个集合字段声明为 @Column，如果不给与初始值，那么调用将会直接出错。因此，对于集合字段，一般给一个空的初始值更好，或者在 getXXX 中进行这种处理，提高性能的同时避免错误。

# 封装替代子类化

## Replace Subclass with Fields

```scala
class People(val name:String, var age:Int) {
  private val languages: mutable.Buffer[String] = mutable.Buffer("Scala")
  def getLanguages:Array[String] = languages.toArray
}
class Man(name:String, age:Int) extends People(name,age) {
  def getCode:String = "M"
}
class Woman(name:String, age:Int) extends People(name,age) {
  def getCode:String = "F"
}
```

可以重构为：

```scala
class SuperPeople(name:String, age:Int,
                  private val code:String) extends People(name,age) {
  def getCode:String = code
}

object PeopleFactory {
  def getPeople(name:String, age:Int, code:String): People = {
    new SuperPeople(name,age,code)
  }
}
```

注意，这本质上是委托和继承之间的选择。总而言之，委托总是更方便的，而继承在原本类不可修改的时候会有优势。从语义上来说，委托适合 has 关系的情况，但是继承只适合于 is 关系的情况。从实际来看，委托的使用情景多一些。

对于上述的这种，对于委托和继承都合适，是一种 is 关系，但是，使用参数而非子类，结构很清晰易懂，同时避免了过多的不必要的类嵌套 —— 如果能够控制类构造器的构造，通过工厂方法来返回对象实例就最好了。

# 控制结构的重构

## 分支判断优化 - 分解条件表达式

> 其目的是为了提升语意清晰度。

条件表达式很容易变得逻辑不清晰，因此，总是考虑使用函数查询或者中间变量来提升条件结构的可读性。
这在很大程度上替代了注释。

需要注意的是，你可以使用 final 变量或者一个函数来代表一个表达式，但是对于变量而言，注意，其被赋值过之后，行为表现和函数查询不同，因此，总是考虑使用函数查询来替代条件表达式，因为控制结构几乎总是和状态判断相关的，而变量可能不会很好代表当前状态，并且其相比较函数而言，更加难以重构，除非它就放在条件表达式上面。

```scala
object AAA {
  val isSummer = true
  val price = 20.0
  val number = 4000
  def getResult: Double = {
    if (isSummer || number > 3000) price * number * 0.9
    else price * number * 1.0
  }

  def needDiscount: Boolean = {
    isSummer || number > 3000
  }

  def getResult2: Double = {
    if (needDiscount) price * number * 0.9
    else price * number * 1.0
  }
}
```

## 分支结构缩减 - 合并条件表达式

> 多个返回值相同的控制结构合并或、单层嵌套控制结构合并与。

多个控制结构返回相同的值，是一种不良的习惯。它们很容易被合并成为一个控制结构，以及一个大的表达式。
除非是你觉得这些检查互相独立，那么不能合并成一个可解释的内容，则不合并。

```
object BBB {
  val isSummer = true
  val price = 20.0
  val number = 4000
  def getResult: Double = {
    if (isSummer) return price * number * 0.9
    if (number > 3000) return price * number * 0.9
    price * number * 0.9
  }

  def needDiscount: Boolean = {
    isSummer || number > 3000
  }

  def getResult2: Double = {
    if (needDiscount) return price * number * 0.9
    price * number * 0.9
  }

  def getResult3: Double = {
    if (isSummer) {
      if (number > 3000) {
        if (price > 10) {
          return price * number * 0.9
        }
      }
    }
    price * number
  }

  def isExpensive: Boolean = price > 10

  //注意，这种单个或者少数的逻辑符号和多个函数查询的控制结构，
  //就不用继续拆分了
  def getResult4: Double = {
    if (needDiscount && isExpensive) price * number * 0.9
    else price * number
  }
}
```


## 分支结构增加 - 使用卫语句

卫语句可以替代嵌套的条件表达式，或者抽取公共逻辑，简而言之，其可以减少分支

```scala
object DDD {
  case class School(name:String, address:Address)
  case class Address(name:String, city:String)
  val a = School("CCNU",Address("Wuhan","China"))
  def printSchoolInfo(): Unit = {
    if (a != null) {
      if (a.address != null) {
        if (!a.address.name.isEmpty) {
          println(a.address.name)
        }
      }
    }
  }

  def printSchoolInfo2(): Unit = {
    if (a == null) return
    if (a.name.isEmpty) return
    if (a.address != null && a.address.name == null) return
    println(a.address.name)
    //其实这个例子不太好，因为这三个分支显然有关系，因此可以使用合并条件表达式，较少分支的方法
  }

  def isSchoolHaveNoAddressName(school: School):Boolean = {
    school != null && school.address != null &&
      school.name != null && !school.address.name.isEmpty
  }

  def printSchoolInfo3(): Unit = {
    if (isSchoolHaveNoAddressName(a)) println(a.address.name)
  }
}
```

## 区块代码优化 - 抽取公共语句

> 合并重复的条件片段 【多个逻辑中的公共语句应该抽取】

和多个返回值相同的控制结构类似，不过这里的限制是，当所有分支都执行同一操作，才可以提取，否则，应该

1、合并控制结构的分支，其更适合平等关系的分支。
2、使用卫语句，当合并分支没有意义时，使用卫语句，先排除不太可能的分支。卫语句更适合那种并非平等关系的分支逻辑。
 
```scala
object CCC {
  def methodA(): Unit = {
    if (1 > 1) {
      println("Hello1")
      println("Hello, World")
    } else if (2 > 2) {
      println("Hello2")
      println("Hello, World")
    }
  }

  def methodA2(): Unit = {
    if (1 > 1) {
      println("Hello1")
    } else if (2 > 2) {
      println("Hello2")
    }
    println("Hello, World")
  }
}
```

## 区块代码优化 - 循环和多返回

> 移除循环停止标记，使用 continue 和 break;

>允许多个分支直接返回，使用 return（在 Scala 中不适用，请绝对不要尽可能的在 Scala 中使用 return 提前返回）

## 高级技术 - 多态替代分支判断

```scala
abstract class Employee(val salary:Int) {
  def getSalary:Int = salary
}
class Manager(salary:Int, val bound:Int) extends Employee(salary) {
  override def getSalary: Int = super.getSalary + bound
}
class Engineer(salary:Int) extends Employee(salary)
class Others(salary:Int, val others:Int) extends Employee(salary) {
  override def getSalary: Int = super.getSalary + (others * 0.7).toInt
}

object Department {

  def printSalary(employee: Employee): Unit = {
    println("\n========== Salary Info ==========")
    println(employee.getSalary)
    println("========== End ==========\n")
  }

  def printSalary2(employee: Employee): Unit = {
    println("\n========== Salary Info ==========")
    employee match {
      case manager: Manager => println(manager.bound + manager.salary)
      case engineer: Engineer => println(engineer.salary)
      case others: Others => println(others.salary + others.others * 0.7)
    }
    println("========== End ==========\n")
  }
}
```

## 高级技术 - Null 对象

```scala
trait NullObject

package NullTest {
  case class School(name:String, address:Address = AddressFactory.fakeAddress)
  case class Address(name:String, city:String)
  object AddressFactory {
    val fakeAddress: Address = new Address("Nothing","Nothing") with NullObject
  }
  object Test {
    def main(args: Array[String]): Unit = {
      val cc = School("CC")
      val address = cc.address
      println(address.isInstanceOf[NullObject]) //true
      val city = cc.address.city
      println(city)
    }
  }
}
```


--------

2019-05-06 撰写本文。