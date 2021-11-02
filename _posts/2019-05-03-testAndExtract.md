---
layout: post
title: 重构 - Unit Test and Extract Method
categories:
- Refactoring
- Scala
- Martin Fowler
---

> 本文介绍了重构的基础：单元测试，单元测试有助于确定重构前的软件行为，用以和重构之后进行对比验证，重构应该不引起任何的测试程序失败发生。本文也讲解了重构的重头戏：函数拆分，拆分有很多方案，包括各种临时变量的处理方法，要根据情景合理选择合适的方案。

# 单元测试

## 单元测试的价值

单元测试是繁琐的，这是肯定的，要写大量的代码。那么，这些代码是值得的吗？对于小规模开发，或者习惯了脚本交互编程，这显然难以理解 —— 为什么需要单元测试，我直接可以肉眼观察软件行为。

单元测试尤其重要的价值，这种价值只有在某些环境下，或者在从事某项编程的开发者才会感到迫切需要。对于需要定义接口（对于接口和实现分离的语言）、快速处理 Bug、需要重构和复用代码的开发者，会需要单元测试 —— 即，如果使用一门静态高级语言开发 9000 行以上的程序，你就会逐步体会到单元测试的价值所在。

### 确定接口设计

作为单元测试最重要的目的，撰写测试代码的最有用时机应该是开始写程序之前。编程是一种思想的产物，因此，在没有开始写的时候，在大脑中进行的设计必然是充满各种小的错误的，这种设计不可靠，不能作为一个标准接口。

单元测试在这里可以发挥重要的作用，因为它可以让开发者集中精力在接口而非实现：想想吧，你定义了一个接口，然后在 test 包下写实现，这些实现必然是不能用的、随意的，你自然不会关心它们。而在实现接口的过程中，必然会发现之前的问题，比如设计不合理，这时候修改接口，然后去实现，循环往复，就可以设计出可用的健全的接口。这导致了我们会更加关心接口，而非实现。

而如果我们在正常的包目录下写实现呢？我们会更多关心实现，这导致接口不好用，为了避免修改接口 —— 既然实现都写了好多了，那么就很容易导致出现巨无霸函数 —— 重构面临的最重要的问题。

**在测试的过程中，你会发现添加一个功能需要做些什么，然后根据此设计接口。**

### 为实现提供信心、回顾生疏知识

单元测试的第二个目的是，你可以用它来回顾一些生疏的知识，或者对什么框架的使用做一个测试之类，这些代码，放在正常包下不合适，放在 test 下吧，这里的代码都是临时的，用于某项专门目的的，这里再好不过了。

当你在这里测试了一个方法，你在实现的时候，就会很有信心，现在你确定了一个合适的接口，然后测试了实现中的生疏部分，那其余的就很简单了。

### 以防万一... 以打造健全、稳定的程序为目标

很多人对测试有误解，他们过分的关注 IDE 给出的测试覆盖率报告，并且因此不敢写测试 —— 因为完全覆盖所有代码的测试是庞大的工程。但其实不是这样。

要正确认识测试，尤其是单元测试的目的。我们并非要测试各种情况，作为实现者，我们应该更关心边界条件，并且花费一定的时间去处理它们，这会导致我们在编程的时候注意这些细节，并且能更早的发现 Bug，以打造健全稳定的程序。

比如一个读入文件流的方法，要确定读入为空、读入不存在程序的行为，这些边界情况是测试最重要的价值所在，测试永远不是（起码单元测试）为了完美覆盖每行代码用的，而是为了更快的暴露问题，打造健全的、稳定的程序而设计的。**编写不完美但是包含边界条件的测试并且实际运行，好过对于完美测试的无尽期待。**

还有一种迷思，当我写了单元测试，并且根据测试测量了边界条件，发现了 Bug 并且进行了修正，我会更加不喜欢测试，因为我总是会想着去抓住所有的 Bug，这种迷思导致了对于测试的抗拒。实际上，花合理时间抓住大部分 Bug 比穷尽一生抓住所有 Bug 要好的多，实际上，白盒测试通常能够抓住大部分 Bug。

### 定位和复现 Bug

当出现 Bug 的时候，通过测试定位到 Bug，会很方便的了解到底发生了什么，这比 IDE 的 Debug 更好用。

### 重构的基线标准

当然，对于重构，单元测试也是少不了的，很简单，因为单元测试不能改变外部可见的软件行为变化，因此，作为基准，重构不应该引起测试结果的改变。这也是衡量重构后软件可用性的一个重要办法。

# 组织函数

## 组织函数的原则

函数拆分需要以一个代码单元做什么事情为依据，如果有这种清晰的目标，其单元内代码再长或者再短也无所谓，甚至函数名字比内部代码还长，其有两个限度：

- 只要想为函数内某块代码写注释、想要更好的显示代码意图，那么就可以拆分
- 只要想不好有意义的名字，那么就不要拆分

## 拆分函数示例

函数拆分的最大问题就是临时变量问题，从最简单的不需要外部参数，也不需要返回值的打印，到需要处理返回值、入参的一般情况，到函数需要对局部变量再赋值的情况，如下所示 [Extract Method]：

```scala
case class User(name:String, age:Int, school:String,
usedLanguages: mutable.Buffer[Language] = mutable.Buffer[Language]())
case class Language(name:String, useYear:Int)

def doSomething(user: User): Unit = {
//根据语意不同拆分为三个函数
println(s"Printed at ${LocalDateTime.now()}")
println("="*40)
//需要额外小心的处理临时变量
var totalYear = 0
user.usedLanguages.foreach(language => {
println(s"Language: ${language.name} - Year: ${language.useYear}")
totalYear += language.useYear
})
println("="*40)
println(s"All LanguageUseTime: $totalYear")
}

def doSomethingNew(user: User): Unit = {
printHeader()
val resultWithTotal = detail(user)
println(resultWithTotal._1)
println(tail(resultWithTotal._2))
}

//对于不依赖外部变量的部分，直接提取为函数
def printHeader() = {
println(s"Printed at ${LocalDateTime.now()}")
println("="*40)
}

//对于作为结果变量的中间值，重命名，使其语意更清晰
//对于 Java 控制结构，可以拆分多个变量以分担责任：val 初始值、var 变化值
val detail: User => (String, Int) = user => {
val sb = new mutable.StringBuilder()
var result = 0
user.usedLanguages.foreach(language => { sb.append(s"Language: ${language.name} - Year: ${language.useYear}").append("\n")
result += language.useYear
})
(sb.toString(), result)
}

//对于依赖的外部变量：只读的值：作为入参，只写的值：作为返回值
val tail: Int => String = total => {
val sb = new mutable.StringBuilder()
sb.append("="*40).append("\n")
sb.append(s"All LanguageUseTime: $total")
sb.toString()
}
```

## 处理中间变量

对于拆分函数，中间变量的去除是关键。中间变量的产生很有意思，它们是暂时的，并且只能在所属的函数内使用。由于中间变量只在所处的函数内可见，**因此它们会驱使开发者写出更长的函数**。这里有一些用于去除中间变量的方法：

### 只读来自外部

- 作为入参：对于只读，来自外部（类）的变量

见上文，这种方法还可以减少一个函数的外部依赖，减少可能的函数状态，更有利于快速重构。

### 只读多次调用

- 拆分为函数调用：对于只读，且提供算法，且多次调用的变量 [Replace Temp with Query]

```scala
def getFinalResult(count:Int, eachNumberPrice:Int) {
val result =  count * eachNumberPrice
if (result > 2000) result * 0.7
else result * 0.9
}
//将中间变量替换为函数查询
val totalPrice: (Int,Int) => Int = 
(count, eachNumberPrice) => count * eachNumberPrice
def getFinalResult(count:Int, eachNumberPrice:Int) {
if (totalPrice(count, eachNumberPrice) > 2000)
totalPrice(count, eachNumberPrice) * 0.7
else totalPrice(count, eachNumberPrice) * 0.9
}
```

### 只读复杂逻辑

- 拆分为内部函数：对于只读，且提供算法，复杂逻辑下的变量  [Introduce Explaining Variable]

```scala
if (a > b || c < d && e == h) ...
//复杂表达式可以拆分为 final 的中间解释变量
val aIsBiggerThanb = a > b
val cIsSmallerThand = c < d
val eIsEqualWithh = e == h
if （aIsBiggerThanb || cIsSmallerThand && eIsEqualWIthh) ...
```

对于拆分为函数和之前的作为入参拆分函数其实是一模一样的，拆分后的函数如果放在原来作用域外，则需要提供入参和返回值定义，称之为 Extract Method，如果这个函数多次用来查询，为了避免造成多次查询的中间变量，每次查询都调用函数，称这种 Extract Method 为 Replace Temp with Query，如果一个 Extract Method 放在内部，引入中间变量来进行解释，则称之为 Introduce Explaining Variable。

### 读写拆分责任

- 重命名和拆分责任：对于写多次的，作为结果和循环体变量

对于结构体而言，使用增强 for 循环代替更好，for 和 while 这种结构体，在其内部引入了大量的状态，当这些结构体代码越长，其内部赋值，计算的中间变量就越容易出错。考虑使用 Scala 的 FP 容器方法，如果不能，那么尽可能不要给一个变量过多的责任，拆分，并且给与解释性的名称。

```java
List[String] list = ...
int length = list.size;
String temp = "=>";
for (int i = 0; i < lenght; i++) {
//do something...
temp += list[i] + ", ";
}
temp += "\n"
System.out.println(temp)

//可以改写为
final String begin = "=>";
final String end = "\n";
String total = "";
for (int i = 0; i < list.size; i++) {
String temp = list[i] + ", "
total += temp
}
System.out.println(begin + total + end)
```

对应的 Scala:

```scala
val begin = "=>"
val end = "\n"
val total = list.asScala.mkString(", ") + ", "
println(begin + total + end)
```

### 只写直接返回

- 拆分后作为返回值：对于只写一次的变量 [Inline Temp /Inline Methods]

```scala
def getResult() = 1 + 1
def aaa() = {
val result = getResult()
return if (result > 2) true esle false
}
def aaa() = if (getResult() > 2) true else false //inline temp
def aaa() = if (1 + 1 > 2) true else false //inline methods
```

> 注意，上述四种方法，正好对应着从完全重构为函数、部分重构为函数（部分使用函数内中间变量）和尽可能降低中间变量的难以理解性、中间函数的不必要性三个方面。此外，还有一种方式，没有重构函数，但是使用了一种别的方式增加了程序的可读性，这就是将函数变为函数类。

### 圈定新类范畴

- 移动到方法对象：对于大量难以处理的变量 [Replace Method with Method Object]

对于非常长的代码，含有非常多的难以处理的交叉引用的变量，这时候可以考虑使用方法对象。其操作如下：

```scala
class Account {
def doAccountThing(): Unit = {
//Father's thing
}
def doSomething(name:String, total:Int): Unit = {
//lot's of code
doAccountThing()
}
}
```

对于此函数，新建一个类，然后传入此函数原本的参数，以及其本身的类（以便引用父类的字段和方法），然后在方法类中进行原本函数的计算，如果需要父类的方法，使用父类引用即可。

在原本函数中，构建此方法类，传入参数和父类，调用计算函数即可。

这样的设计后，你可以轻松的对现在 compute 方法进行 Extract Method 而不用担心参数传递的问题了 —— 因为所有的局部变量都变成了方法对象的字段，这样的话，你就可以很方便的在方法对象这个作用域内去拆分原本庞大的函数了。

```scala
class Account {
def doAccountThing(): Unit = {
//Father's thing
}
def doSomething(name:String, total:Int): Unit = {
new AccountCompute(this, name, total).compute()
}
class AccountCompute(val account: Account, name:String, total:Int) {
def compute(): Unit = {
//lot's of Account's code here
account.doAccountThing()
}
}
}
```

圈定新类范畴和后面章节中的为类划分责任，拆分类很相似。但是，其作用主体不同，前者是对于大函数而言的，后者是对大类而言的，但是其作用相同，都是为了动态调整各个结构的责任，划分清楚其权利边界。

## 代码风格的调整

需要注意，如果对于一些临时变量无法处理，或者按照上述方式处理了中间变量，在一个方法中只剩下了区区几个变量，但是，有意思的是，它可能还是不够清晰易懂。

典型的比如 GUI，虽然控制组件不太多，但是对于各个控制组件需要多次设置方法来控制其外观展现，并且可能存在交叉的组件协同调用，就会面临这种问题。

这时候，你可以将那些对于同一个组件调用的代码放在一起。对于 Scala 而言，你甚至可以充分利用大括号返回值，这是一种不太优雅的 Builder 模式，但是确实提升了可读性，其在视觉上更加清晰易懂。

```scala
val root: Parent = {

val info: Text = {
val t = new Text()
t.textProperty().bind(new SimpleStringProperty("在英文输入下，记录按键，按键后 ")
.concat(waiting.textProperty()).concat(" 秒内可修改，以防止主试记录的错误。\n" +
"按下 Enter 记录为空，按下数字记录对应数字")); t
}

val keyPressed: Label = {
val t = new Label("")
t.textProperty().bind(key)
t.setTextFill(Color.RED)
t.setFont(Font.font(80)); t
}

val allBox: VBox = {
val t = new VBox()
t.setSpacing(10)
t.setAlignment(Pos.CENTER_LEFT); t
}

val funcBox: HBox = {
val t = new HBox()
t.setAlignment(Pos.CENTER_LEFT)
t.setSpacing(15); t
}

funcBox.getChildren.addAll(new Label("可修改延迟时间(秒)："), waiting, enter, edit, save)
allBox.getChildren.addAll(funcBox, voiceRecorder.viewer)

val pane = new BorderPane()
pane.setTop(info)
pane.setCenter(keyPressed)
pane.setBottom(allBox)

BorderPane.setAlignment(info, Pos.CENTER)
BorderPane.setMargin(info, new Insets(20,20,20,20))
BorderPane.setMargin(allBox, new Insets(20,20,20,20))
pane
}
```

————————————

2019-05-03 撰写本文。

2019-05-06 更新了 “拆分函数重构后的风格调整” 部分。

