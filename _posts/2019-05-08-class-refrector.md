---
layout: post
title: 重构 - Deal with Generalization
categories:
- Refactoring
- Scala
- Martin Fowler
---

> 这一章是关于重构概括关系的总结。概括关系，简而言之就是一种代码复用体系，也就是面向对象的类系统的重构。常见的概括关系重构包括字段和函数的上移和下移、构造函数的上移、使用工厂方法代替构复杂造器。除了处理一个现有的类型系统，还可以对类型系统进行更改，比如将父类拆分为父类和子类，将相关但是不同的类移动到父类后直接删除、合并抽象逻辑为父类、从函数中提取公共模板方法、定义两个类的角色 —— 接口。此外，本文还探讨了概括关系的两种选择：委托和继承，前者适用于 has 关系，后者适用于 is 关系。

# 现有继承体系重构

## Pull Up Field

当两个子类拥有相同的字段，将该字段移至超类。

```scala
package BEFORE {
  class People {
    var name:String = ""
    override def toString: String = s"${getClass.getSimpleName} - $name"
  }
  class Student extends People {
    var age:Int = 0
  }
  class Employee extends People {
    var age:Int = 20
  }
  object Test extends App {
    val a = new People
    a.name = "Corkine"
    val b = new Student
    b.name = "Corkine"
    b.age = 40
    val c = new Employee
    c.name = "Corkine"
    c.age = 30
    println(a,b,c)
  }
}
package AFTER {
  class People {
    var name:String = ""
    var age:Int = 0
    override def toString: String = s"${getClass.getSimpleName} - $name"
  }
  class Student extends People {
    age = 0
  }
  class Employee extends People {
    age = 20
  }
  object Test extends App {
    val a = new People
    a.name = "Corkine"
    val b = new Student
    b.name = "Corkine"
    b.age = 40
    val c = new Employee
    c.name = "Corkine"
    c.age = 30
    println(a,b,c)
  }
}
```

## Pull Up Method

有些函数，在各个子类中产生完全相同的结果，将其移动到超类。

```scala
package Before {
  class People()
  class Student(name:String) extends People {
    def sayHi:String = "Hello From Student"
  }
  class Employee(name:String) extends People {
    def sayHi:String = "Hello From Employee"
  }
  object Test extends App {
    val a = new People
    val b = new Student("Stu")
    val c = new Employee("Emp")
    println(a,b.sayHi,c.sayHi)
  }
}
package After {
  class People(name:String) {
    def sayHi:String = s"Hello From ${getClass.getSimpleName}"
  }
  class Student(name:String) extends People(name)
  class Employee(name:String) extends People(name)
  object Test extends App {
    val a = new People("Corkine")
    val b = new Student("Corkine")
    val c = new Employee("Corkine")
    println(a.sayHi, b.sayHi, c.sayHi)
  }
}
```

### Pull Up Extract Method

在多个子类中的有些函数，其大部分逻辑相同，将这个函数重构，拆分，将共同部分合并到超类。

```scala
package Before {
  class People {
    var name = ""
    var age = 0
  }
  class Student extends People {
    var school:String = ""
    def getInformation: String = {
      s"Student - $name, $age, $school"
    }
  }
  class Employee extends People {
    var company:String = ""
    def getInformation: String = {
      s"Employee - $name, $age, $company"
    }
  }
  object Test extends App {
    val a = new People
    a.name = "Corkine"
    val b = new Student
    b.name = "Corkine"
    b.age = 22
    b.school = "CCNU"
    val c = new Employee
    c.name = "Corkine"
    c.age = 22
    c.company = "CCCP"
    println(a, b.getInformation, c.getInformation)
  }
}
package After {
  class People {
    var name = ""
    var age = 0
    def getBasicInformation:String = getClass.getSimpleName + s" - $name, $age, "
  }
  class Student extends People {
    var school:String = ""
    def getInformation: String = {
      getBasicInformation + s"$school"
    }
  }
  class Employee extends People {
    var company:String = ""
    def getInformation: String = {
      getBasicInformation + s"$company"
    }
  }
  object Test extends App {
    val a = new People
    a.name = "Corkine"
    val b = new Student
    b.name = "Corkine"
    b.age = 22
    b.school = "CCNU"
    val c = new Employee
    c.name = "Corkine"
    c.age = 22
    c.company = "CCCP"
    println(a, b.getInformation, c.getInformation)
  }
}
```

### Pull Up Constructor Body

在每个子类中拥有一些构造函数，可以将其移动到超类，然后在子类调用它。

```scala
package Before {
  class Person(val name:String, val age:Int)
  class Student(name:String, age:Int, school:String) extends Person(name,age) {
    println("Init Main Constructor ...")
  }
  class Employee(name:String, age:Int, val salary:Int) extends Person(name,age) {
    println("Init Main Constructor ...")
  }
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a,b,c)
  }
}
package After {
  class Person(val name:String, val age:Int) {
    println("Init Main Constructor ...")
  }
  class Student(name:String, age:Int, school:String) extends Person(name,age)
  class Employee(name:String, age:Int, val salary:Int) extends Person(name,age)
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a,b,c)
  }
}
```

## Push Down Field

当超类中某个字段只被部分，而非全部子类用到，将这个字段移动到它的那些子类中去。

```scala
package Before {
  class Person(val name:String, val age:Int) {
    val schoolCode: Int = {
      this match {
        case s:Student => s.school.length
        case _ => -1
      }
    }
  }
  class Student(name:String, age:Int, val school:String) extends Person(name,age) 
  class Employee(name:String, age:Int, val salary:Int) extends Person(name,age)
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a.schoolCode,b.schoolCode,c.schoolCode)
  }
}
package After {
  class Person(val name:String, val age:Int)
  class Student(name:String, age:Int, val school:String) 
        extends Person(name,age) {
    val schoolCode: Int = this.school.length
  }
  class Employee(name:String, age:Int, val salary:Int) 
        extends Person(name,age)
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a,b.schoolCode,c)
  }
}
```

## Push Down Method

当超类中某个函数只和部分，而非全部子类有关，将这个函数移动到相关的子类中去。

```scala
package Before {
  class Person(val name:String, val age:Int) {
    def schoolCode: Int = {
      this match {
        case s:Student => s.school.length
        case _ => -1
      }
    }
  }
  class Student(name:String, age:Int, val school:String) extends Person(name,age)
  class Employee(name:String, age:Int, val salary:Int) extends Person(name,age)
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a.schoolCode,b.schoolCode,c.schoolCode)
  }
}
package After {
  class Person(val name:String, val age:Int)
  class Student(name:String, age:Int, val school:String) 
        extends Person(name,age) {
    def schoolCode: Int = this.school.length
  }
  class Employee(name:String, age:Int, val salary:Int) 
        extends Person(name,age)
  object Test extends App {
    val a = new Person("Corkine",22)
    val b = new Student("Student", 22, "CCNU")
    val c = new Employee("Employee",22,1000)
    println(a,b.schoolCode,c)
  }
}
```

# 重构现有继承结构

## 拆分：Extract Subclass

当类中某些特性值被某些，而非全部实例用到，新建一个子类，然后将上面所说的特性移动到子类中去。

同 Pull Up Method，不过这里新建了一个子类，略。

## 拆分：Extract Hierarchy

当一个类做了太多的事情，考虑将其分解，然后将一部分任务交给子类去做。这和 Pull Up Method、Field、Extract Subclass 的思想相同，不过，**Extract Subclass 更多的是将大的类的功能拆分，而这里的 Extract Hierarchy 考虑的更多是，用一个子类代表一种类的特殊情况（当这个原本的类使用大量的条件表达式时，这样的拆分更有效）。**

## 合并：Extract Superclass

当两个类有相似特性，为这两个类建立一个超类，将相同特性移动到超类中。

```scala
package Before {
  class Student(val name:String, val age:Int, val school:String)
  class Employee(val name:String, val age:Int, val company:String)
  object Test extends App {
    val a = new Student("Corkine",22,"CCNU")
    val b = new Employee("Corkine",22,"Wuhan CCCP")
    println(a, b)
  }
}
package After {
  class People(val name:String, val age:Int)
  class Student(name:String, age:Int, val school:String) 
        extends People(name, age)
  class Employee(name:String, age:Int, val company:String) 
        extends People(name, age)
  object Test extends App {
    val a = new Student("Corkine",22,"CCNU")
    val b = new Employee("Corkine",22,"Wuhan CCCP")
    println(a, b)
  }
}
```

## 合并：Collapse Hierarchy

当超类和子类没有太大区别，并且子类只是简单复写了一些字段，此外没有别的子类，将它们合为一体。

```scala
package Before {
  class Student(val name:String, val age:Int, val school:String)
  class Employee(val name:String, val age:Int, val company:String)
  object Test extends App {
    val a = new Student("Corkine",22,"CCNU")
    val b = new Employee("Corkine",22,"Wuhan CCCP")
    println(a, b)
  }
}
package After {
  class People(val name:String, val age:Int) {
    var school:String = _
  }
  object Test extends App {
    val a = new People("Corkine",22)
    a.school = "CCNU"
    println(a)
  }
}
```

## 合并：Extract Interface

若干个类接口的同一子集，或者两个类的接口有相同部分，但是两个类不存在相似的范畴，将相同的子集提炼到一个独立的接口中。

```scala
package Before {
  class Animal {
    def sayHi:String = "mow...."
  }
  class People(val name:String, val age:Int) {
    def sayHi:String = s"I'm $name, $age years old."
  }
  object Test extends App {
    val a = new Animal
    val b = new People("Corkine",22)
    def sayHi(from:Any): String = {
      from match {
        case i: Animal => i.sayHi
        case p :People => p.sayHi
      }
    }
    println(sayHi(a), sayHi(b))
  }
}
package After {
  trait HiSayer {
    def sayHi:String
  }
  class Animal extends HiSayer {
    override def sayHi: String = "mow..."
  }
  class People(val name:String, val age:Int) extends HiSayer {
    override def sayHi: String = s"I'm $name, $age years old."
  }
  object Test extends App {
    val a = new Animal
    val b = new People("Corkine",22)
    def sayHi(from:HiSayer): String = from.sayHi
    println(sayHi(a), sayHi(b))
  }
}
```

很显然，接口约定的优势在于，可以统一为接口类，隐藏内部实现，不论是对入参而言，还是对工厂方法获取的对象类型而言。

# 使用委托重构 has 关系

## Replace Inheritance with Delegation

```scala
package Before {
  class Animal(val name:String, val age:Int)
  class Ball(animalName:String, age:Int,
             val ballName:String) extends Animal(animalName, age) {
    def status:String = s"$animalName is playing $ballName"
  }
  object Test extends App {
    val a = new Animal("Cat",2)
    val b = new Ball("Cat",2,"RedBall")
    print(b.status)
  }
}
package After {
  class Animal(val name:String, val age:Int)
  class Ball(var animal: Animal,
             val ballName:String)  {
    def status:String = s"${animal.name} is playing $ballName"
  }
  object Test extends App {
    val a = new Animal("Cat",2)
    val b = new Ball(a,"RedBall")
    print(b.status)
  }
}
```

## Replace Delegation with Inheritance

```scala
package Before {
  class Father(val name:String, val age:Int) {
    def doWithSkill1():Unit = {
      print("Father is singing...")
    }
    def doWithSkill2():Unit = {
      print("Father is singing2...")
    }
    def doWithSkill3():Unit = {
      print("Father is singing3...")
    }
    def doWithSkill4():Unit = {
      print("Father is singing4...")
    }
  }
  class Son(val name:String, val age:Int, val father: Father) {
    def doWithSkill1():Unit = {
      father.doWithSkill1()
    }
    def doWithSkill2():Unit = {
      father.doWithSkill2()
    }
    def doWithSkill3():Unit = {
      father.doWithSkill3()
    }
    def doWithSkill4():Unit = {
      father.doWithSkill4()
    }
  }
  object Test extends App {
    val a = new Father("CC",44)
    val b = new Son("cc",10, a)
    b.doWithSkill1()
    b.doWithSkill3()
  }
}
package After {
  class Father(val name:String, val age:Int) {
    def doWithSkill1():Unit = {
      print("Father is singing...")
    }
    def doWithSkill2():Unit = {
      print("Father is singing2...")
    }
    def doWithSkill3():Unit = {
      print("Father is singing3...")
    }
    def doWithSkill4():Unit = {
      print("Father is singing4...")
    }
  }
  class Son(val sonName:String, val sonAge:Int,
            fatherName:String, fatherAge:Int) extends Father(fatherName,fatherAge)
  object Test extends App {
    val b = new Son("cc",10,"CC",44)
    b.doWithSkill1()
    b.doWithSkill3()
  }
}
```
## Tease Apart Inheritance

对于面向过程的程序，更改为面向对象之后，往对象中不免会添加越来越多的方法。当一个类越来越大，那么就需要使用 Eh Es 等手法将其拆分为继承体系，但是，当使用一个子类表示一种特殊情况、一种功能增强后，慢慢的子类也会变得更加臃肿。越来越多的子类被代表越来越多的例外，这可不好，容易造成类爆炸。

实际上，一个类应该有明确的行为边界，其子类也应该在这个边界内提供增强和特殊增强。因此，当子类膨胀之后，一个可行的办法就是，将子类重新整理，抽取不符合类边界的部分，如果这部分过大，那么使用新类替代，然后使用组合而非继承去组装。

如下所示：

```scala
package Before {
  class Person(val name:String ,val age:Int)
  class Student(name:String, age:Int, val school:String) extends Person(name,age)
  class Employee(name:String, age:Int, val company:String) extends Person(name,age)
  class WuhanStudent(name:String, age:Int, val school:String) extends Person(name,age) {
    def eatReGanMian(): Unit = { }
  }
  class WuhanEmployee(name:String, age:Int, val employee:String) extends Person(name,age) {
    def playInGuangGu(): Unit = { }
  }
  class HeNanStudent(name:String, age:Int, val school:String) extends Person(name,age) {
    def playInLuoYang():Unit = { }
  }
  class HeNanEmployee(name:String, age:Int, val employee:String) extends Person(name,age) {
    def workMoreHard(): Unit = { }
  }
}
```

当整理过继承体系后，现在的继承看起来清晰多了：

```scala
package After {
  class Person(val name:String ,val age:Int) {
    def play(): Unit = { }
  }
  class Student(name:String, age:Int, val location: Location, val school:String) extends Person(name,age) {
    def eat(): Unit = location match {
      case Location(nam) => print(s"Eat $nam")
      case _ => print("Nothing...")
    }
  }
  class Employee(name:String, age:Int, val location: Location, val company:String) extends Person(name,age) {
    def eat(): Unit = { }
    def work(): Unit = { }
  }
  class Location(val name:String)
  object Location {
    def apply(name: String): Location = new Location(name)
    def unapply(arg: Location): Option[String] = Option(arg.name)
  }
}
```

# 大型重构

## Convert Procedural Design to Objects

参见第一部分的 Hello，World 部分， 将一个超大的函数拆分成了多个函数，然后将这些函数分门别类的移动到了几个不同领域的对象中。

## Separate Domain from Presentation

对于 GUI 而言，按照 Domain 和 Presentation 进行拆分，这个和 Extract Subclass 按照功能拆分有一定的区别，和 Extract hierarchy 将特殊条件拆分为子类也不同。Eh 更接近状态博士，Es 更接近功能模块，而这里的 Dp 则更接近一种广义上的结构划分。

______

2019-05-09 撰写本文。