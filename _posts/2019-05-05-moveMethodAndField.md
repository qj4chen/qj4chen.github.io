---
layout: post
title: 重构 - Redistribution Responsibility
categories:
- Refactoring
- Scala
- Martin Fowler
---

> 本文介绍了重构的第二大法宝：重新为类分配责任，简而言之，其包括：字段和函数的迁移、类的拆分和合并、委托隐藏和中间人移除、对于不可写类的功能扩展。在上一节着重介绍的函数拆分，包括各种临时变量的处理，对包括脚本和一般、严肃类型语言均适用。而本文介绍的方法，更多侧重 OOP 语言，最好是统一对象访问的语言，比如 Strongtalk（一种 Smalltalk 的方言）、Scala（一种 Java 的方言）、Dart（一种试图替代 TypeScript 的可选类型语言）。

在上一节中，我们将一个超大的函数拆分成了多个不同的部分，合理安置了临时变量，这在很大程度上已经很好的提升了程序的清晰性。但是还不够，当程序达到上万行的规模，一个超大的类包含超多的函数和一个超大的函数没有什么两样。因此，本节着重介绍对这种问题的处理，即根据类型来划分函数的归属的方法。

主要内容分为四个方面：

第一方面【**类的组成**】，对于多个类型，将其中不合适的方法、字段进行迁移，保证这些方法和其类型扮演的角色之间存在强相关，提高程序的清晰性。

第二方面【**类的体量**】，对于超大的单体类型进行拆分，或者对于多个类型进行合并，合理分配类型角色边界。

第三方面【**类的权限**】，对于类型和类型关系的处理，包括更加强力的控制访问字段类型，进行委托全局处理，或者放松限制，删除委托处理，允许客户直接访问对象。

第四方面【**类的扩展**】，对于无法修改的类型，添加功能的问题，解决方案之一是在客户端侧添加外部函数，解决方法之二是对原来的类进行包装、子类化，以保证在角色类似的前提下对功能进行扩展（面向修改关闭，面向扩展开放的设计原则）。

# 类的组成 

## Move Method

当一个类中的函数和别的类/被别的类更密切的使用，那么应该考虑将其移动到那个类中。如果有多个类使用，要么公共化，要么放在使用最多的类中，即 `a.do(B b) 可以改成 b.do(A a)`。这样有助于提高类的功能边界的清晰性。

```scala
class A {
val a = 0
def doA(): Unit = { }
def doBInA(): Unit = {
doA()
println(a) //调用了 A 的字段和方法
//更接近 B 的责任
}
}
class B
```

Move Method 的要点在于**确定此方法的依赖并且适当的进行处理**：对于字段，更改为方法查询（或者不改，使用类.字段进行访问），对于方法，调用类.方法进行查询。

如上代码更改为：

```scala
class A2 {
val a = 0
def doA(): Unit = { }
def doBInA(): Unit = {
//如果不想影响别的依赖，直接委托 B 即可，否则，删除所有依赖
//此函数的方法
B2.doBInB(this)
}
}

object B2 {
def doBInB(a: A2): Unit = {
//对于 A 的调用，使用外部引用
a.doA()
println(a.a)
}
}
```

## Move Field

对于字段而言，如果其面临和方法同样的问题，那么照例也要那样进行处理。

```scala
//Move Field
//要点：自我封装，确定一个统一引用点，然后委托其他类提供此字段
class A3 {
var a = 0
def method1InA(): Double = math.sqrt(a)
def method2InA(): Int = a * 10
}

class B3
```

需要注意的是，在原本类中可能有很多方法引用了字段，那么需要考虑先使用自我包装，将这个字段改变成一个方法查询，然后在方法查询入口提供一个委托入口。

对于 final 的字段而言，自我包装就没有必要了，直接在赋值的位置进行委托替换即可。这也适用于 Scala 的 val 变量。

```scala
class A4 {
//var _a = 0
def getA: Int = {
//_a Self Encapsulation Field 自我封装一般在 Move Field 中很有用
B4.a //当方法多次调用此字段，放在一个地方有利于统一管理
}
def method1InA(): Double = math.sqrt(getA)
def method2InA(): Int = getA * 10
}

object B4 {
var a = 0
}
```

这里需要注意的是，Move Field 和 Move Method 的区别是什么？在 Scala 下，Field 和 Method 是一个东西吗？不是，重构语境下的 Field 是一个值/引用，即便是函数，也是没有调用的函数。而 Method 是一个带有副作用的，依赖于类中其他字段、方法的动作。

因此，**这里的 Field 和 Method，是静态策略和动态方法之间的区别。因此，对于字段，要先升格为方法，然后在此方法中进行委托。**其实，这对于统一访问特性的语言来说是很容易理解的，在 scala 中，不存在对于字段的访问（编译成的 JVM 机器码再翻译为 Java 代码除外）。

这意味着，对于提供一个引用而言，做一个统一访问点，然后链接到其他类，委托实现，对提供一个对象而言，因为其可能依赖各种复杂的逻辑，因此需要提供其原本的对象（this），赋予移动函数后新函数内部访问原本类的一切公共资源，执行一个动作。

# 类的体量

> 因为平常使用 Scala 开发，下意识的，我总是会将函数中重复的部分提取为一个 val 变量，让这个变量表示一个函数，这就是上一节中讲到的基本函数拆分，如果此函数变量含有原本函数中间变量，则形成一个闭包，也就是上一节中讲到的解释性变量。因为这种思想，所以在处理类责任的时候，更多的会发现因为方法调用了很多的类变量，而觉得无法进行更细致的拆分了 —— 站在 FP 的角度，是的，无法将方法进一步拆分为不可变的算法和策略了。

> 但是这仅仅是对于 Move Field 和 Move Method 的不理解而已。Move Method 可以将过度耦合类变量的语句拆分出来，当方法仅访问某些只被其访问的变量的时候，这很容易拆分 —— 将它们塞入新类即可（Move Method），如果变量被原来类其余方法调用，只用更改调用为新类的变量即可（Move Field）。但是，如果一个方法访问类变量，更改类变量，并且调用了类的其他方法呢，这其实也很简单，Move Field 后，添加一个对于原本类的引用参数，让这部分的语句访问此类对象即可。类是类似于函数的一种作用域，仅仅是贴了一个做什么的标签，每个类不一定要是完全独立的，仅提供策略的，也可以是彼此紧密相连的，这对于习惯 FP 的开发者来说很难第一时间想到 —— FP 习惯了不可变的数据结构和类型。

下面介绍的拆分责任和合并责任中的拆分就需要迈过这个坎，如果理解到这个问题，那么对于一个和本类关系不密切，但是调用了大量的本类字段、方法的目标类而言，就很容易将其隔离，并且移动到新的地方：添加一个原本类的引用即可。


**一个类应该表示一种抽象，但是实际开发中，常常为一个类添加过多的功能。这时候就应该提炼类 Extract Class。**

一个很好的评价标准就是，_如果子类化只影响了父类的某些特性，或者说，父类的某些特性需要用一种方式子类化，但是另外的一些特性需要以另一种方式子类化，这意味着需要分解原来的类_。拆分原理和 Move Field、Move Method 类似。

和 Extract Class 相反的是 Inline Class，当一个类不再承担过多责任的时候，就删除这个类，使用方法和 Move Field、Move Method 类似。

> 拆分责任这件事，说着简单，但实际很难有一个客观的评价标准。尤其是对于 GUI 类而言，很容易写的过于庞大，因为中间涉及大量的 GUI 控件、控制方法和状态。因此，对于一个大类而言，要问自己，如果这个类要子类化成多个子类，即提供多个实现，那么要怎么做？这样想之后，如果做不到，那显然这个类扮演了过多的责任，因此考虑将其拆分，直到一个类能够明确责任边界（即你可以很轻松的想到它可以用 n 种不同的方法做）。对于有子类的类，如果各个子类之间存在较大的差异，那么也应该做这种考虑，各个子类应该仅仅是实现上的区别，和额外的功能补足，而不应该有不同的角色。

## 拆分类的一个实践

我试着对于 JavaFX 的一个 GUI 类进行拆分，之前它是合在一起的，即 GUI 组件和其 Controller 放在一起，虽然我也使用了几种重构手法拆分了大函数，并且创建了几个类分担这个 GUI 类的责任，这看起来很不错了。但是，那种拆分是基于功能的，我拆分的一个小的类包含一个清晰的责任边界，但是其中还是混杂了 GUI 组件以及其动态执行的方法。这种设计有典型的缺陷，就是耦合过度紧密，新类的 GUI 部分必须在原本类 GUI 初始化之前调用，但是新类的动态回调必须在原本类初始化之后由用户定义。这看起来还没什么，只是说新类必须在某个合适的地方初始化，在另外一个地方调用执行方法。

但是，当原本类越来越大，我不得不将其拆分为 Viewer 和 Controller，这是一种 MVP 的架构风格，其中 Viewer 和 Model 完全隔绝，不处理复杂逻辑，仅负责展现，并且提供几种可供 Controller 调用以改变外观状态的方法，而 Controller 则负责和 Model 逻辑交互。VC 的拆分就是一种对于类责任的划分，不同于之前的根据责任的划分，这种基于 VC 的划分更接近业界的划分标准，并且从长远的角度来看更有利于测试、集成和修改 —— 比如让 Viewer 重新实现为 Web、Swing 等。

同样的，我也对之前拆分的新类按照这样的方法拆分成了 Viewer 和 Controller，我发现，Viewer 如果要组合到别的组件上时，其 Controller 定义的回调必然没有设置。因为总是先创建 Viewer，因此，新类的 Controller 和原本类的 Controller 都必须延后调用，这是一种不太 FP 的方式。

`原本类 Viewer -> 新类 Viewer -> 原本类 Controller -> 新类 Controller`。

这里还存在一个两个类的 Controller 的责任划分问题。具体而言，要尽可能保持一个清晰的责任边界，并且为未来的更改做出选择，即可以保持一个组件更抽象，而另一个组件完成脏活，这样，在下次重构时，只用处理其中之一即可。

此外，如果一个组件分成 Viewer 和 Controller，那么，对其扩展要如何处理呢？继承 Controller 或者 继承 Viewer，然后继承组件，为 Viewer/Controller 重新赋值是一种办法，其具体取决于实际想要重载的功能/外观。

## 拆分类的两种方式

这告诉我什么道理呢？拆分类，可能有约定俗成的业界标准，比如 MVC，或者其余领域设计模式 —— 抽象工厂等等。

拆分类不是一个简单的过程，因为我们不得不关注拆分后的类之间的关系，希望它们不要足够紧密，但是又可以不影响互相通信，同时满足可以扩展的需求。

一般而言，划分边界后，对于一个类很容易抽象出一个接口边界，这时候两个类交互使用接口即可。

此外，另一种做法就是子类化，这种方式更简单，子类有所有非私有父类字段方法读取权限，但是其实这是有条件的，因为子类通常是用来扩展满足相似功能的，子类和父类应该具有相同的功能范畴，因此，最好不要使用子类完成类的功能拆分 —— 当然，如果你发现类的一部分是善变的，那么将其拆分到子类进行实现，而在父类提供一个基础或者抽象方法(或者 Scala 的自身类型和特质混合)也是一种很好的实践。

这其实是委托和继承的选择，主要注意，在拆分类的时候，如果是功能拆分，那么使用接口和委托，如果是实现拆分，那么使用继承和子类。

# 类的权限

## 隐藏委托、合并责任

这个拆分纯粹是根据业务来的，在开发的不同阶段，可能需要进行不同的控制。这可能和访问对象的类字段对象的安全性、其字段多少、生命周期敏感性有关。

```scala
class Manager {
var player: Player = _
}

class Player {
var friend: Player = _
def doA(): Unit = {}
def doB(): Unit = {}
}

object Stage {
val manager: Manager = {
val t = new Manager
val player = new Player
val player2 = new Player
player.friend = player2
t.player = player; t
}
def main(args: Array[String]): Unit = {
manager.player.doA()
manager.player.friend.doB()
//移除中间人【Remove Middle Man】
//更灵活
manager
.player
.friend
.friend
.friend
.friend
.friend
.friend.doB()
}
}
```

## 拆分责任和移除中间人

当需要更加强力的控制，或者需要隐藏内部实现，更改为如下代码即可：

```scala
class Manager2 {
var player: Player2 = _
}

class Player2(private val friend: Player2) {
def doA(): Unit = {}
def doB(): Unit = {}
def callFriendsDoB(): Unit = {
if (friend != null) friend.doB()
}
def callFriendsFriendsFriendsFriendsFriendsDoB(): Unit = {
try {
friend.friend.friend.friend.friend.doB()
} catch {
case _: Throwable =>
}
}
}

object Stage2 {
val manager: Manager2 = {
val t = new Manager2
val friend = new Player2(null)
val player = new Player2(friend)
t.player = player; t
}
def main(args: Array[String]): Unit = {
manager.player.doA()
//封装后的结果 【Hide Delegate】
//更方便客户端调用
manager.player.callFriendsDoB()
manager.player.callFriendsFriendsFriendsFriendsFriendsDoB()
}
}
```

# 类的扩展

如果想要扩展类的方法，但是又因为各种原因不能修改类的话，可以使用对象技术和设计模式进行处理。

```scala
//当不可修改原本类时的方法扩展
class C(var init: Int = 0) {
def unitConvert(int: Int): Int = {
int * 100
}
def rateCompute(int: Int): Double = {
if (math.sqrt(int) > 200) 0.9 else 1.0
}
}
//假如需要大量计算 tateCompute(unitConvert(init)) * int
//这本应该假如到 C 中的，但是因为 C 无法更改，因此先放在客户端中
object CallC{
val c = new C(100)
val finalResult: C => Double =
c => c.rateCompute(c.unitConvert(c.init)) * c.init
//finalResult 就是外加函数，外加函数不能过多，否则还会造成混淆
//这时候应该使用 引入本地扩展 方法
def main(args: Array[String]): Unit = {
println(finalResult)
}
}
```

第一种方法是子类化，子类化是一种很好的分离代码的方式，在保持类型对象角色的一致性的情况下复用父类大部分代码，同时重写部分逻辑，改变部分行为，为原本的类提供多态能力以及扩展能力：

```scala
class SuperC(from:Int) extends C(from) {
def getFinalResult: Double =
this.rateCompute(this.unitConvert(this.init)) * this.init
}
object CallC2 {
val c = new SuperC(100)
def main(args: Array[String]): Unit = {
println(c.getFinalResult)
}
}
```

第二种方法是使用 Scala 的隐式类，它可以以一种神奇的方式为类提供扩展方法（动态底层包装和代理）：

```scala
object CallC3 {
implicit class CExt(c:C) {
def getFinalResult: Double = c.rateCompute(c.unitConvert(c.init)) * c.init
}
val c = new C(100)
def main(args: Array[String]): Unit = {
println(c.getFinalResult)
}
}
```

————————

2019-05-05 撰写了本文

2019-05-06 添加了拆分类的一个实践和两种方式。
