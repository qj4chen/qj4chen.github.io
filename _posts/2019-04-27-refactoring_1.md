---
layout: post
title: 重构 - Why Refactoring
categories:
  - Refactoring
  - Scala
  - Martin Fowler
---

> 这系列博客是我阅读《重构：改善既有代码的设计》一书的笔记。在原书中使用的是 Java 作为示例，而我使用的则是 Scala，本系列博客可为 Scala 重构提供类似的经验。

# 示例

## 版本一：快速开发的恶果

这是第一章示例的 Scala 等价代码，其包含了一个复杂系统中的一个简单功能。这个功能开始有一个糟糕、但是能运行的实现，这很像面向过程的编程风格：

```scala
case class Movie(title:String, var priceCode:Int)
case class Rental(movie: Movie, daysRented: Int)
case class Customer(name:String) {
  val rentals: mutable.Buffer[Rental] = mutable.Buffer[Rental]()
  def addRental(rental: Rental): Unit = rentals.append(rental)
}
object Movie {
  val CHILDREN = 2
  val REGULAR = 0
  val NEW_RELEASE = 1
}
object Store {
  //v1 问题：函数过长，可以抽取功能边界
  def statement(customer: Customer): String = {
    var totalAmount = 0.0
    var frequentRenterPoints = 0
    val elements = customer.rentals
    var result = "Rental Record for " + customer.name + "\n"
    for (i <- elements.indices) {
      var thisAmount = 0.0
      val rental = elements(i)
      rental.movie.priceCode match {
        case Movie.REGULAR =>
          thisAmount += 2
          if (rental.daysRented > 2)
            thisAmount += (rental.daysRented - 2) * 1.5
        case Movie.NEW_RELEASE =>
          thisAmount += rental.daysRented * 3
        case Movie.CHILDREN =>
          thisAmount += 1.5
          if (rental.daysRented > 3)
            thisAmount += (rental.daysRented - 3) * 1.5
      }
      frequentRenterPoints += 1
      if (rental.movie.priceCode == Movie.NEW_RELEASE &&
      rental.daysRented > 1) frequentRenterPoints += 1
      result += "\t" + rental.movie.title + "\t" + thisAmount + "\n"
      totalAmount += thisAmount
    }
    result += "Amount owned is " + totalAmount + "\n"
    result += "You earned " + frequentRenterPoints + " frequent renter points"
    result
  }
}
```

很显然，这个程序可以运行，但是难以修改。这就是所谓重构，在不影响外部接口的情况下改变内部结构，以让程序结构满足预期，便于进一步修改、增加新功能、减少 Bug。

实际而言，最主要的动机是：如上代码打印一个 CD 租贸行的一个客户租贸的 CD 的总价钱和积分。对于不同的 CD，有不同的折扣和积分。这里的问题是，在 statement 函数中，耦合了大量的业务逻辑：在这里遍历客户的 CD，然后根据其类型收不同的加钱，给与不同的积分。

这个程序看上去很好理解。实际也是很容易理解，对于生手，花一段时间就能看懂，问题是，程序并不是实时印刻在脑海中的，也不一定是一个人维护的，一段好的程序应该具有自证性，并且要尽可能的短，以便于在不影响其他内容的情况下进行快速的、少 Bug 的修改。

听起来没什么，不是吗？

重构，不过解决的问题如此之小。吗？

一般而言，重构总是伴随着目的的，当我们想要打印 HTML 而不是 Text 文本，试问，上面的 statement 如何修改？如果我还要打印 PDF、WORD 呢？**当发现自己需要为程序调价一个特性，而代码结构让自己无法很方便的达成目的，那就先重构那个程序，使添加特性比较容易进行，然后再添加特性。**

对于一次性的脚本，比如 Python 或者 R、MATLAB，显然没有重构的必要，而对于需要重复复用代码的需求，这时候，重构就变成一件不得不做的事情了。

**重构的第一步在于建立测试基线环境**，这意味着，你修改内部结构之后，不应该影响到外部可见的行为。而作为奖赏，修改内部结构会让添加功能更加简单、快速、Bug 更少。

如下是 ScalaTest 的测试基线：

```scala
class StoreTest extends FlatSpec with Matchers {

  val demoCustomer: Customer = {
    val c = Customer("Corkine")
    c.rentals.appendAll(mutable.Buffer(
      Rental(Movie("复仇者联盟 4", Movie.NEW_RELEASE), 5),
      Rental(Movie("复仇者联盟 3", Movie.REGULAR), 3),
      Rental(Movie("复仇者联盟 2", Movie.REGULAR), 4),
      Rental(Movie("复仇者联盟 1", Movie.CHILDREN), 2),
      Rental(Movie("复仇者联盟 0", Movie.CHILDREN), 1)
    ))
    c
  }

  val result: String = """Rental Record for Corkine
                 |	复仇者联盟 4	15.0
                 |	复仇者联盟 3	3.5
                 |	复仇者联盟 2	5.0
                 |	复仇者联盟 1	1.5
                 |	复仇者联盟 0	1.5
                 |Amount owned is 26.5
                 |You earned 6 frequent renter points""".stripMargin

  "v1" should "have a fixed Statement" in {
    val str = Store.statement(demoCustomer)
    assertResult(str, "重构前后返回值应该等同") {
      result
    }
  }
}
```

## 版本二：抽取公共函数

```scala
case class Movie(title:String, var priceCode:Int)
case class Rental(movie: Movie, daysRented: Int)
case class Customer(name:String) {
  val rentals: mutable.Buffer[Rental] = mutable.Buffer[Rental]()
  def addRental(rental: Rental): Unit = rentals.append(rental)
}
object Movie {
  val CHILDREN = 2
  val REGULAR = 0
  val NEW_RELEASE = 1
}
object Store2 {
  def amountFor(rental: Rental): Double = {
    //对于函数而言，要确定其依赖的不变的量（入参）和改变的值（返回值）
    //对于函数内部的变量重命名是一个好习惯，比如返回值使用 result 表示
    //代码总应该表现自己的目的
    var result = 0.0
    rental.movie.priceCode match {
      case Movie.REGULAR =>
        result += 2
        if (rental.daysRented > 2)
          result += (rental.daysRented - 2) * 1.5
      case Movie.NEW_RELEASE =>
        result += rental.daysRented * 3
      case Movie.CHILDREN =>
        result += 1.5
        if (rental.daysRented > 3)
          result += (rental.daysRented - 3) * 1.5
    }
    result
  }
  //v2 问题：功能总应该和其主体关联
  //虽然我们抽象了函数，但是，函数也会发生爆炸，难以管理
  //面向对象的优势在这里应该被很好利用， amountFor(rental) 应该写成 rental#anmoutAll
  def statement(customer: Customer): String = {
    var totalAmount = 0.0
    var frequentRenterPoints = 0
    val elements = customer.rentals
    var result = "Rental Record for " + customer.name + "\n"
    for (i <- elements.indices) {
      var thisAmount = 0.0
      val rental = elements(i)
      //划分边界，并且抽取功能后的代码
      //优点：功能分割，将变化和不变分离，易于维护和后期扩充
      thisAmount = amountFor(rental)
      frequentRenterPoints += 1
      if (rental.movie.priceCode == Movie.NEW_RELEASE &&
        rental.daysRented > 1) frequentRenterPoints += 1
      result += "\t" + rental.movie.title + "\t" + thisAmount + "\n"
      totalAmount += thisAmount
    }
    result += "Amount owned is " + totalAmount + "\n"
    result += "You earned " + frequentRenterPoints + " frequent renter points"
    result
  }
}
```

如上所示，这里抽取了整块代码中最容易变化的计算金额的部分，抽象成公共函数。抽象需要注意的点是，要确认函数依赖的局部变量和外部变量，对于外部不可变值作为参数传入，对于函数改变的值，作为返回值返回即可。

重构应该小步进行，应该立刻编写测试:

```scala
"v2" should "have same Statement with v1" in {
val str = Store2.statement(demoCustomer)
assert(str == result)
}
```

很显然，这里通过了测试，重构后的结果和之前完全相等。

但是，问题依旧存在，当程序变得庞大，多个函数放在一起也难以管理，因此，我们需要将其放入其对应的主体，作为其方法。这样，对于需要依赖很多参数的函数，作为方法就可以访问类对象的类变量了。

注意：`amountFor(rental)` 永远不如 `rental#anmountAll` 来的结构清晰（后者提供了函数的一种所属关系），虽然可能方便一点点。


## 版本三：面向对象的演变

```scala
case class Movie(title:String, var priceCode:Int)
case class Rental(movie: Movie, daysRented: Int) {
  def amountAll: Double = {
    var result = 0.0
    this.movie.priceCode match {
      case Movie.REGULAR =>
        result += 2
        if (this.daysRented > 2)
          result += (this.daysRented - 2) * 1.5
      case Movie.NEW_RELEASE =>
        result += this.daysRented * 3
      case Movie.CHILDREN =>
        result += 1.5
        if (this.daysRented > 3)
          result += (this.daysRented - 3) * 1.5
    }
    result
  }
  val getFrequentRenterPoint: Int = {
    if (this.movie.priceCode == Movie.NEW_RELEASE &&
      this.daysRented > 1) 2 else 1
  }
}
case class Customer(name:String) {
  val rentals: mutable.Buffer[Rental] = mutable.Buffer[Rental]()
  def addRental(rental: Rental): Unit = rentals.append(rental)
}
object Movie {
  val CHILDREN = 2
  val REGULAR = 0
  val NEW_RELEASE = 1
}
object Store3 {
  def amountFor(rental: Rental): Double = {
    //对于公共函数，如果不想去除 API，那么使用新的实现即可
    rental.amountAll
  }
  //v3 包含了大量的临时变量，即在一个 for 循环中不断变化的值
  //下一个版本和原文有较大出入，其更好的利用了 Scala 的函数式编程，减少了临时变量
  def statement(customer: Customer): String = {
    var totalAmount = 0.0
    var frequentRenterPoints = 0
    val elements = customer.rentals
    var result = "Rental Record for " + customer.name + "\n"
    for (i <- elements.indices) {
      var thisAmount = 0.0
      val rental = elements(i)
      //OOP 风格代码，函数各归其家
      thisAmount = amountFor(rental) //thisAmount = rental.amountAll
      frequentRenterPoints += rental.getFrequentRenterPoint.toInt
      result += "\t" + rental.movie.title + "\t" + thisAmount + "\n"
      totalAmount += thisAmount
    }
    result += "Amount owned is " + totalAmount + "\n"
    result += "You earned " + frequentRenterPoints + " frequent renter points"
    result
  }
}
```

可以看到，我们将 Store 中的两个公共函数放入了 Rental 类中，这样函数的功能更清晰。但是，这样的话，我们就面临一个问题，对于之前调用 Store 中的代码，要怎么办？方法其一是更改所有依赖其的代码，方法其二是更改其实现，使用类方法的实现。后者一般是常见的选择。

注意观察，现在 statement 就非常简洁了。测试代码如下：

```scala
"v3" should "have same Statement with v1" in {
assert(Store3.statement(demoCustomer) == result)
}
```

## 版本四：函数式编程风格

上述的 statement 还有一些问题，比如 for 循环中还含有很多临时变量。临时变量是有用的，但是如果能避免，则完全可以避免，因为其不断在 for 中变化，状态太多，容易造成管理和理解上的逻辑问题，引起 Bug。使用函数式风格即拒绝使用传统的命令式控制结构，即 for、while 循环能不用就不用，如果用，也不在其中引入临时变量。

这种风格可能引起性能的降低，但是，应该抱着这种思想：重构的目的应该总是服务编程者优先，而不是服务机器。至于性能优化，则是完全相反的路子，不要在做一件事的时候考虑另一件，很容易发生冲突。重构总是要先按照其目的完成代码优化，便于理解程序逻辑、添加和修改新功能。

```scala
case class Movie(title:String, var priceCode:Int)
object Movie {
  val CHILDREN = 2
  val REGULAR = 0
  val NEW_RELEASE = 1
}
case class Rental(movie: Movie, daysRented: Int) {
  def amountAll: Double = {
    var result = 0.0
    this.movie.priceCode match {
      case Movie.REGULAR =>
        result += 2
        if (this.daysRented > 2)
          result += (this.daysRented - 2) * 1.5
      case Movie.NEW_RELEASE =>
        result += this.daysRented * 3
      case Movie.CHILDREN =>
        result += 1.5
        if (this.daysRented > 3)
          result += (this.daysRented - 3) * 1.5
    }
    result
  }

  val getFrequentRenterPoint: Int = {
    if (this.movie.priceCode == Movie.NEW_RELEASE &&
      this.daysRented > 1) 2 else 1
  }
}
case class Customer(name:String) {
  val rentals: mutable.Buffer[Rental] = mutable.Buffer[Rental]()
  def addRental(rental: Rental): Unit = rentals.append(rental)
}
object Store4 {
  val amountFor: Rental => Double = _.amountAll

  val totalAmount: Customer => Double = customer =>
    customer.rentals.foldLeft(0.0)((sum, r) => sum + r.amountAll)

  val frequentRenterPoints: Customer => Int = customer =>
    customer.rentals.foldLeft(0)((sum, r) => sum + r.getFrequentRenterPoint)

  //这一版已经很好的达成目的了，然而，其实还是有一些可以重构的
  //比如多态和调用，这些重构可能会在程序扩展的时候起到重要作用
  //问题出现在 Rental 的 amountAll 方法上
  def statement(customer: Customer): String = {
    var result = "Rental Record for " + customer.name + "\n"
    customer.rentals.foreach(rental => {
      result += "\t" + rental.movie.title + "\t" + rental.amountAll + "\n"
    })
    result += "Amount owned is " + totalAmount(customer) + "\n"
    result += "You earned " + frequentRenterPoints(customer) + " frequent renter points"
    result
  }

  //作为重构的问题和成果，现在我们可以很轻松的添加打印 HTML 的报表
  def statementForHTML(customer: Customer): String = {
    var result = "<h1>Rental Record for " + customer.name + "</h1>\n<ul>"
    customer.rentals.foreach(rental => {
      result += "<li>" + rental.movie.title + ", " + rental.amountAll + "</li>\n"
    })
    result += "</ul><p>Amount owned is " + totalAmount(customer) + "</p>\n"
    result += "<p>You earned " + frequentRenterPoints(customer) + " frequent renter points</p>"
    result
  }
}
```

注意这里，我们使用 Scala 函数编程的优点抽取了循环，减少了临时变量。这时候，添加一个打印 HTML 功能就非常简单了，可以看到，statementForHTML 方法中几乎每句话都和 statement 逻辑相同，但是每句话都不一样，这样就极大限度的复用了代码，好处？当修改其底层依赖时，这两个打印的函数总会同步发生变化。

测试代码如下：

```scala
"v4" should "have same Statement with v1" in {
val res = Store4.statement(demoCustomer)
println(res)
assert(res == result)
}

it should "work well on HTML Statement" in {
println(Store4.statementForHTML(demoCustomer))
}
```

## 版本五：多态与类的层次

如下是版本五的 statement，没怎么变化，我们在这里要处理的是类。

```scala
object Store5 {

  val totalAmount: Customer => Double = customer =>
    customer.rentals.foldLeft(0.0)((sum, r) => sum + r.amountAll)

  val frequentRenterPoints: Customer => Int = customer =>
    customer.rentals.foldLeft(0)((sum, r) => sum + r.getFrequentRenterPoint)

  //这一版已经很好的达成目的了，然而，其实还是有一些可以重构的
  //比如多态和调用，这些重构可能会在程序扩展的时候起到重要作用
  //问题出现在 Rental 的 amountAll 方法上
  def statement(customer: Customer): String = {
    var result = "Rental Record for " + customer.name + "\n"
    customer.rentals.foreach(rental => {
      result += "\t" + rental.movie.title + "\t" + rental.amountAll + "\n"
    })
    result += "Amount owned is " + totalAmount(customer) + "\n"
    result += "You earned " + frequentRenterPoints(customer) + " frequent renter points"
    result
  }

  //作为重构的问题和成果，现在我们可以很轻松的添加打印 HTML 的报表
  def statementForHTML(customer: Customer): String = {
    var result = "<h1>Rental Record for " + customer.name + "</h1>\n<ul>"
    customer.rentals.foreach(rental => {
      result += "<li>" + rental.movie.title + ", " + rental.amountAll + "</li>\n"
    })
    result += "</ul><p>Amount owned is " + totalAmount(customer) + "</p>\n"
    result += "<p>You earned " + frequentRenterPoints(customer) + " frequent renter points</p>"
    result
  }
}
```

问题还是处在 priceCode 的 match 上，这里可以看到，这里的 Rental 操纵了属性 Movie 的 PriceCode 属性，这是一个选择，我们希望将根据 Movie 类型得到 Amount 的代码放在 Movie 中，还是 Rental 中，这取决于我们的程序目的。

在这里，因为 Movie 的分类是一个很大的变量，而 Rental 则长久不变，因此，考虑到变化分离，我们更希望在之后只用维护 Movie，为其添加新的分类，这样的话，将根据分类计算价格的代码放在 Movie 中更好。

如果移动的话，很简单，但是，我们还想玩点新的，提高 Movie 的可复用行，将 Movie 的变化进一步分离出变化和不变的，很显然，我们不能为 Movie 创建几个根据分类不同的子类，因为 Movie 在运行时不允许转型，但是我们业务或许需要不断调整 Movie 分类，因此，考虑为 Movie 提供 Price 类实例作为 priceCode 的中间值，而利用 Price 类提供多态继承能力来动态改变 priceCode API 获取时得到的值。

```scala
case class Movie(title:String, priceIn:Int) {
  private var _price: Price = _

  priceCode_=(priceIn)

  def priceCode_=(in: Int): Unit = {
    _price = in match {
      case Movie.REGULAR =>
        new RegularPrice
      case Movie.NEW_RELEASE =>
        new NewReleasePrice
      case Movie.CHILDREN =>
        new ChildrenPrice
    }
  }
  def priceCode: Int = {
    _price.getPriceCode
  }
  //实现采用多态交给子类完成
  //这种方法虽然在添加 Movie 类型的时候依然需要更改 priceCode_= 的代码
  //但是因为利用了多态，这意味着，如果频繁变动每种不同标签的折扣，换句话说
  //标签的变动相比较标签折扣的变动较小的话，这种拆分 - 多态还是值得的
  val amountAll: Int => Double = _price.amountAll(_)
}

trait Price {
  def getPriceCode: Int
  val amountAll: Int => Double
}

class ChildrenPrice extends Price {
  override def getPriceCode: Int = Movie.CHILDREN
  override val amountAll: Int => Double = delay => {
    var result = 1.5
    if (delay > 3) result += (delay - 3) * 1.5
    result
  }
}

class NewReleasePrice extends Price {
  override def getPriceCode: Int = Movie.NEW_RELEASE
  override val amountAll: Int => Double = _ * 3.0
}

class RegularPrice extends Price {
  override def getPriceCode: Int = Movie.REGULAR
  override val amountAll: Int => Double = delay => {
    var result = 2.0
    if (delay > 2) result += (delay - 2) * 1.5
    result
  }
}

object Movie {
  val CHILDREN = 2
  val REGULAR = 0
  val NEW_RELEASE = 1
}

case class Rental(movie: Movie, daysRented: Int) {
  //老旧的 API 保存，但是使用新的实现
  val amountAll: Double = movie.amountAll(daysRented)
  val getFrequentRenterPoint: Int = {
    if (this.movie.priceCode == Movie.NEW_RELEASE &&
      this.daysRented > 1) 2 else 1
  }
}

case class Customer(name:String) {
  val rentals: mutable.Buffer[Rental] = mutable.Buffer[Rental]()
  def addRental(rental: Rental): Unit = rentals.append(rental)
}
```

可以看到，在上面我们面向对象的演变一节，我们将公共函数抽取到了类中，作为类方法。而现在，我们想要进一步的在类方法中区分可变和不可变，很显然在这里促销是最容易变化的，而价格类型则不怎么容易变化，因此，考虑为价格类型创建类体系，注入 Movie 中，这样，仅仅在需要添加类型的时候，才处理 Movie 类的代码，而当某种类型的促销发生变化，则直接修改对应子类即可，这种设计是一种权衡，极大的利用了面向对象多态的能力进一步区分一个类中代码的可变和不可变，将一个类重构成了类层次。

其测试代码如下：

```scala
"v5" should "have same Stagement with v1" in {
    assert(Store5.statement(demoCustomer) == result)
}
```

# 要点

## 什么是重构

根据马丁·福乐，作为名词的重构指的是对软件内部结构的调整，其目的是在不改变软件可观察行为的前提下，**使程序更易理解，降低其修改成本，提高可扩展性。**。

和重构在前提上相似，但实际对立的是**性能优化**，性能优化在不改变软件可观察行为的前提下，提高程序的运行速度。

和重构完全相反的是**添加功能**，Kent Beck 认为，程序员头上有两顶帽子，其一是重构，其二是添加新功能。在添加功能的时候，不应该修改既有代码，只添加功能并通过测试。而在重构的时候，不应该添加任何功能，依旧需要通过测试。<u>添加功能和重构不应该混在一起，这就像一心二用，很容易导致 Bug。</u>

## 重构的重要性

重构的重要性在于这样一个前提 —— 在开发者视角的代码工程，并非是一件已经完成的完成品，软件并非是一个产品，而是一系列根据基础代码不断更新迭代的产品。因此，重构尤其重要的意义，因为，它就像为明天铺路，为未来投资，虽然当下对现阶段的功能实现没有帮助，但是在未来，可以减少继续维护和更新功能的成本。**在产品视角，程序应该有两方面的价值，其一是它今天可以为你做什么，其二是它明天可以为你做什么。**重构就是关注第二方面的价值，从短期角度看，重构对今天程序提供的功能没有什么帮助，但是从长远角度看，重构花了很少的时间减少了未来维护的可观时间。

## 为什么重构

重构的目的总是使代码易于阅读，减少修改的成本。具体而言，重构主要做了两方面工作，其一是对于程序内部的，**重构总是改进了软件的内部设计，让代码回到其本来应该所在的位置** —— 其一就是利用大量的中间层，彼此衔接调用，这样允许我们很容易的复用同一个函数/方法/类的逻辑，消除重复代码。试着想一下，如果要修改一个逻辑，而这个逻辑重复分布在多个函数/方法/类中，即便轻微的修改，也是非常庞大的任务。其二是让函数回归其所属的类，全局变量有害，全局函数的弊端也不小。

其二是对于程序开发人员而言的，代码总是给机器读的，但是优雅的代码则是给人读的，除非像是只写一次的脚本，否则我们总是会经常面临复用之前代码，修改它，扩展它的机会，如果代码是容易读的，那么修改起来应该很快。重构在上文中拆分了代码，在这个过程中我们可以给中间层命名，代码应该是自解释的，即通过函数的名称就能知道函数做了什么 —— 换句话说，**代码应该更好的表达自己的用途，每个模块（类、方法、函数）都应该准确说出“它”想要的。**

## 何时何处重构

### 坏代码的特征

根据上面的例子，坏代码总是把变化和不变的混在一起。一个坏的代码无非是和重构后的目标相反 —— 难以阅读和理解、难以进行 Bug Free 的修改，难以进行弹性的扩充：

- **难以阅读**（函数名称乱用缩写，长达 100 行的超级函数，全局定义大量的函数）
- **添加新行为时需要修改已有代码**（添加新功能应该尽可能少的更改之前的代码，如果不，那么需要引入设计模式，解耦变与不变）
    - **逻辑重复**（函数内重复逻辑、重复函数、重复实现子类）
    - **带有复杂条件逻辑**（尤其是 switch 语句，三个及以上的 if...else，没有上下文、缺乏自解释性的控制结构）

### 重构对代码的改进

一个优秀的程序总是应该把变化和不变拆开，而重构的实现几乎总是引入更多的间接层。间接层有利有弊，但总体而言，从阅读、修改和扩容而言，其有如下的优点：

- **分开解释意图和实现**（中间层总是产生更多的函数/方法/类，这些结构的名称解释了其意图，这样，代码本身就可以解释其功能）
- **隔离变与不变**（对于函数而言，当需要对一处逻辑进行修改，中间层可以隔离其对于外界的变化，只用关心输入输出即可，即函数是一个黑箱。对于方法而言，类提供了隔绝内部字段的绝好的机会，将代码抽象为多个类，每个类只在其自身作用域下处理其依赖字段，不容易产生外部依赖的问题。对于类继承而言[包括一般重写和抽象重写，本质都是多态机制]，如果一个对象在多处使用，如果需要进行某个行为的修正，那么继承提供了一种奇妙的代码复用机制，只用定义子类重写的方法，用来表示变化，而其余逻辑使用超类即可）
    - **允许代码逻辑共享**（对于函数、方法而言，一个子函数、方法可以在多个不同地点、超类调用，避免了重复代码[次要]，并且一次修改，处处更新[主要]）
    - **封装条件逻辑**（面向过程的逻辑可以被面向对象的消息所替代。多态提供了对于条件逻辑的一种具有弹性的、清晰易懂的、减少代码的复用机制。）

> 注意 允许代码逻辑共享的本质不是减少重复代码，而是为了可以一次性修改所有重复代码，即隔离变与不变。这里和后者区分开的目的在于，前者并非主观区分开变化和不变，而仅仅是为区分变与不变提供了一个松耦合的预先处理。
> 对于脚本语言，尤其是非 OOP 语言，隔离变与不变更多的就是允许代码逻辑共享，因为拆分函数是这类语言能够做的最好的部分了。而对于 OOP、FP 语言，函数提取、类型提取、类型继承的多态机制，包括简单工厂、工厂方法、抽象工厂、策略、代理等等设计模式都可以达到隔离变与不变。
> 对于封装条件逻辑，这里其实也可以归为隔离变与不变，但是因为它太过于常用，即总是会有这种 switch 控制条件判断，并且需要经常修改其中的逻辑，因此单独介绍，一般通过三大工厂可以很好的隔离可变性。

## 重构和模式

可以这样理解，重构是应用设计模式在工程中的动作，设计模式是一种指导重构的思想。

## 重构和接口

一般而言，接口，或者说 public 接口，又称之为发布的接口，这种接口随着重构，一般不能够随意删除，除非自己管理所有对其依赖的调用。这里的经验是，不要随便提供发布接口，属性或者方法尽量采用私有作用域，或者包作用域。当然，对于 Scala 而言，其类默认为公共作用域，这就要求我们尽可能面向接口编程，而不是面向某个实现类，而尽可能的减少接口的数量。

对于 Java 的异常，其算作签名的一部分，因为 Interface 定义的接口方法涉及异常签名，所以可以为一个包单独提供一个 Exception，然后子类继承它即可，这样，更改抛出的异常也不会影响接口方法签名中的异常声明。

## 重构和设计

编程和盖房子不同，对于盖房子，设计和施工分得很开，而编程则因为完全是思想的产品 —— 因此具有思想典型的特点：快速、不准确、充满漏洞。软件可塑性很强，有了设计，可以思考的更快，但是其中充满了漏洞。

介于这种特点，**软件工程不要求一开始就设计出弹性的、可扩容的机制，然后去实现它，而是先大致根据目前的需求设计，之后再通过重构不断改进以适应新功能。**

**接口 -> 测试 -> 实现 -> 测试**，这个顺序中，测试会发现接口的设计问题，然后回过头重新设计接口，直到看起来没有问题，之后进行实现，在实现的过程中，可能又发现问题，然后又修改接口，测试，上线。这里的在实现中修改接口的过程，就是重构的一种 —— 因为虽然先有结构可用，但是不容易理解。当添加新功能的时候，如果需要修改很多地方，或者修改之前的代码，那么也需要重构。

在最开始进行过多的思考，是一种痛苦，因为空想根本不能代表可能的程序功能走向。抓紧应该做的是实现稍微有弹性的、良好设计模式的设计，之后去实现。当后期出现问题的时候，比如添加功能很困难，出现 bug 的时候，再去修改内部结构，按照设计模式和预期功能进行优化。这样步步为营的方式，最符合实际开发。

## 重构和性能

很多时候，在开发的时候，会过多考虑性能的问题。但是，其实这是有问题的。当在开发的时候思考性能的问题，站在局部的角度看待问题，很容易求一个局部最优解，但，这往往不是全局最优。因此，在一个全局角度看待性能问题是很重要的。这就要求优化性能，但是优化和编写是分开的。重构也是，重构就不要优化性能，而是以容易读、容易扩容的角度设计内部结构。

关于性能一个很有意思的现象时，我们常常认为可能出问题的地方并没有出问题，反而在意想不到的地方耗时严重，这要求测试工具检查。并且大多数代码严重耗时往往集中在很少一部分，因此介于这种原因，推荐把优化、重构、添加功能完全分开。

# 附录

Maven pom 依赖的 jar 和 插件 如下：

```xml
<dependencies>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-core</artifactId>
    <version>1.2.3</version>
</dependency>
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>2.12.6</version>
</dependency>
<dependency>
    <groupId>org.scalatest</groupId>
    <artifactId>scalatest_2.12</artifactId>
    <version>3.1.0-RC1</version>
    <scope>test</scope>
</dependency>
</dependencies>

<build>
<plugins>
<plugin>
<groupId>net.alchim31.maven</groupId>
<artifactId>scala-maven-plugin</artifactId>
<version>3.2.2</version>
<executions>
    <execution>
        <goals>
            <goal>compile</goal>
            <goal>testCompile</goal>
        </goals>
        <configuration>
            <args>
                <arg>-dependencyfile</arg>
                <arg>${project.build.directory}/.scala_dependencies</arg>
            </args>
        </configuration>
    </execution>
</executions>
</plugin>
</plugins>
</build>
```


______________

2019-04-29 更新本文，撰写第二章内容
2019-04-28 撰写 Hello World
