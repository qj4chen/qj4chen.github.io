---
layout: post
title: JavaScript学习笔记 - 对象篇
categories:
  - JavaScript
  - 读书笔记
---
> 在本月的前些时候，我学习了JavaScript这门语言。在前两篇文章中较为详细的介绍了JavaScript的语法和客户端API，本文则重点介绍JavaScript极富创意的对象表面量表示法和基于原型的编程范式。这是我在寒假学习JavaScript的最后一篇文章，也是近期最后一篇关于此的文章。学习JavaScript纯属一时冲动，不过却收获满满，10天前的我一头扎进书里，而当10天后的我从书中抬起头来，这世界恍如隔世，光怪陆离，如梦似幻。

# 3 深入理解JavaScript对象

## 3.1 对象的属性

在Javascript中，一个对象可以是一个单独的拥有属性和类型的实体。属性定义了对象的特征，通过点号表示法来访问对象。`objectName.propertyName`。对象的名称会被强制转换成为字符串，即便你设置一个Object当作propertyName，其也会调用toString()转换成为字符串，任何没有赋值的对象的值都为undefined。属性的访问使用 `myCar["model"] = "Mustang";` 或者点号表示法，属性的值可以为bool值、字符串、数字等几乎任何类型，你甚至可以嵌套一个对象到其对象属性上。

## 3.2 基于类和基于原型的语言

JS是基于原型的语言，这意味着其属性可以分为私有属性和属于原型链（继承而来的）属性，在调用对象的属性的时候，默认会首先检查私有属性（OwnProperty），如果没有，通过对象的proto属性来遍历原型链进行继续检查。一个简短的、函数风格的（构造新对象，面向对象而非过程）对象如下：

    var createPet = function(name,age){
        var that = {};
        that.name = name || "Unset";
        that.age = age || "Unknow"; 
        return that;
    }

    var cmCat = createPet("huahua");
    var cmCatSon = Object.create(cmCat);
    
    cmCat =>
        age: "Unknow"
        name: "huahua"
        --proto--: Object { … } // 继承了Object对象的所有方法和属性

    cmCatSon =>
        {}
        --proto-- Object { name: "huahua", age: "Unknow" } // 即cmCat对象
    

对于函数风格编程，所有采用createXXX()函数创建的对象的proto属性，也就是继承均为Object，而通过Object.create(o)创建的对象自动继承自这个通过函数创造的这个对象，比如cmCatSon继承自cmCat，其具有自己的私有属性（虽然现在是空的），以及原型链上的属性（通过proto可查看，可以直接访问和使用）。这种使用方法简单明了，函数不在对象创造的过程中起重要作用，用户随时可以新建一个函数用来进行从Objcet的新的原型链。此外，面向对象编程风格、纯函数编程风格都十分明显。

但是，JavaScript为了讨好基于类的经典语言的习惯，搞出了一个类似于“类”的函数类型，这个函数（称之为伪类风格编程）相比较函数风格编程唯一的优势就是，其可以少些几个字母，使用this来代替变量，不用写ruturn语句，并且通过new声明将自身的prototype属性赋值给实例的--proto--方法，这样就实现了混杂着类的基于原型的编程风格，当然，它还有一个好处，就是可以搭建基于构造函数的伪类继承。不得不承认的一个事实是，使用构造函数可以带来更加有层次的模板关系，对于一个具有复杂结构的编程环境来说一个复杂的模板尤其方便之处，但是所有基于此的继承都要从头开始解析构造函数，这造成了效率的损失。反之，使用字面量定义并且使用Object.create()方法进行继承则可以带来更为简单、直接和易用的对象模型。尤其考虑到JS随处打补丁的编程习惯（又称灵活的设计模式），一个简单易用的对象模型还是有必要的。

> 原型链和继承

不同于基于类的语言，类的继承之间是直接复制的关系，所有的实例都从类中复制一份属性到自己的空间，JS采用的是全局变量和原型链的概念，这意味着你通过任何方式继承的属性其实都存在于一个公共空间内，称之为原型链。对任意对象调用proto属性可以返回其被构造的对象（返回的不是构造器函数而是此函数的prototype属性，即其映射的原型对象）。如果需要对原型链添加属性，调用YourClass.prototype.attr = function(){} 即可，对单一对象进行的属性更改不会影响原型链的属性，比如YourInst.attr = function(){}。

> 属性的枚举

使用 `Object.key(obj_1)` 来返回一个不包括原型链的对象的所有属性，结果格式为数组。

使用 `Object.getOwnPropertyNames(o)` 也可以返回一个数组，其包含对象所有非原型链属性（不论是否可以枚举）。

> 一个比喻

大概基于类的语言和基于原型的语言的区别就相当于SQL和NoSQL一样，前者有所谓的模板和结构，而后者也可以有前者的模板，但是也可以没有，其相当的灵活。我在学习MongoDB的时候，就很喜欢自己搞一个模板给所有传入的值，然后统一写入到数据库，但是后来我发现没有必要，尽管这样做是可行的，但是并没有很好的利用这些程序/数据库/语言所提供的自由特性。一大串的None和null并不好看，而我唯一欣慰的则是查询每一个文档都会返回给我数据的结构。这就好像中央集权和联邦民主，使用者唯一要考虑的问题是，定义一个以后基本不改变的数据结构代价大还是面对今后需要调整数据结构的代价大。

## 3.3 创建新对象

### 3.3.1 使用字面量创建

一般来说，有三种方法可以创建新对象，其一为使用对象初始化器，其实就是Object字面量来“写”一个对象。

    if (cond) { var obj_1 = { name:"pet", age:"32" ,1+1*100:233}};

其中作为属性的名称，其可以为任意东西，反正最后会被转换成为字符串，比如这个obj_1中就有一个101的属性。属性可以嵌套对象。对象的每次调用都会重新调用对象初始化器，因此，可以在其前面添加一个if条件判断，只有满足条件时才创建这个对象。

### 3.3.2 使用构造函数创建**

第二种方法是使用构造函数，如下：

    var Pet = function Pet (name, sex) {
        this.name = name;
        this.sex = sex;
    }

之后使用 var objinst = new Pet("namestring") 进行调用，注意不要少了new这个字段。注意，这样建立的对象包含一个proto属性（返回其原型链的上游），查看可见其构造函数对应的对象为 constructor:Pet()。

在调用new构造函数的时候，JS会自动调用Pet.prototype（构造函数对应的原型对象）并向objinst的proto属性添加这个构造器。直接调用Pet而不使用new不会构造出一个新对象，因为其没有上述过程的实现。介于new是一个很容易忘记的符号，并且这种基于类的编程习惯和JS设计不符合，加之对于多个构造函数进行继承的繁复操作(call、更改prototype、更改prototype.contructor)，因此并不建议任何人以任何方法是用构造函数。

如果任何对于这个实例（基于类的说法）/对象（基于原型的说法）进行属性的重新定义，那么其将不会影响原型链，比如：objinst.color = "red"; 这时这个对象的color属性就变成了红色，但是使用Pet创建的其余对象则不会有这个color属性，如果需要对构造函数进行更改，则需要操作 Pet.prototype.color = "None"; 这样会影响所有基于构造函数的对象，其实这是一种类似于“类（Class）”的写法，其会导致大量编程中的混乱（因为过强的动态性导致了我们不知道在哪里更改过prototype.attr这个东西），这样很糟糕，构造函数的另外一个缺点是容易忘记在使用时写new，这样的话不会有任何错误提醒，但是后果却很严重，因为其默认将原型继承自Object对象，因此，不推荐任何使用构造函数来新建对象的方式。

### 3.3.3 使用create方法创建

第三种方法是使用Object.create(o)来构建新对象，JS基于原型的理念在这里得到了很好的表现：

    var a = { name:"cat", age:"3" }; var b = Object.create(a); b; b.@@proto@@; b.name; // Returns cat
    // @@处应为双下划线，其和MD语法冲突。

我们可以使用对象a新建一个对象b，这样的话，b可以使用基于a的任何属性。在proto属性上可以看到对象a是其原型链的上游。b使用的所有对象和其属性都是基于a的原型链，使用这种方法而不是构造函数，再也不用“为了让基于类的程序员快速上手这门语言”而牺牲掉的语言一致性了。

对象中不仅含有变量，还有方法，方法是一个函数，不过其被包裹在对象中而已，因此称之为方法，这和基于类的语言是没有区别的。

    var createPet = function (name= "Unset",age="Unknow") {

        return {
            name : name,
            age : age,
            detail : this.name + ", " + this.age,
            hi : function () {
                var age_s = "";
                if (typeof(this.age) === "number") 
                { if (this.age < 3) { age_s = "Young"; } else { age_s = "Old" }} else {age_s = "Lalala";}
                return "I am " + this.name + ", " + age_s;
            }
        }    
    };

    var a = createPet("miao",2); a.name; a.hi(); // Returns miao; Returns I am miao, Young

这是一个推荐的用法，其避免了使用构造器的随处打补丁的不一致性，同是具备了纯函数编程的优点，生成了一个新对象，任何时候调用a的时候会调用这个createPet函数来返回一个对象。这样构建的对象和使用字面量构建的对象一样，属于Object直接的原型。而比如使用 var babycat = Object.create(a);则可以继续原型链并且继承所有a对象的属性和方法。

这种方法可以用来制造模块，模块是我们用函数和闭包实现的一种可以提供接口，但是却隐藏状态和实现的函数或者对象，通过模块的使用，我们可以摒弃全局变量的使用。一般而言在定义的函数模块内声明一些变量，然后返回一个对象或者函数，这样的话，这些变量由于函数闭包所以可见，而对于全局变量而言则不可见，外部亦不能更改其内部状态。

JavaScript 有一个特殊的关键字 this，它可以在方法中使用以指代当前对象。这和Python中的“self”很类似，你如上述createPet函数中在hi()方法中我们就调用了this来指代这个函数本身。需要注意的是，this使用条件存在限制，比如：

    var a = function(){
        var that = this;
        setInterval(function(){that.say = "Hello";},1000)
        // this.say 是错误的，因为这是另一个函数，其this指代已经改变
    }

### 3.4 使用函数构造器来实现构造函数的继承细节**

使用函数构造器是我们不推荐的一种方法，因为如果想要构建两个函数构造器，比如Ball和Shape来表示球和更为抽象的形状，在这个例子中需要做如下工作：


    // 首先使用Shape构造器构造父类

    function Shape(x, y, velX, velY, exists) {
        this.x = x;
        this.y = y;
        this.velX = velX;
        this.velY = velY;
        this.exists = exists;
    }

    // 其次构造一个子类，其第一句含义等同于Python的Super(ClassName,self).__init__(parent)调用父类以初始化。

    function Ball(x, y, velX, velY, color, size, exists) {
        Shape.call(this, x, y, velX, velY, exists);
        this.color = color;
        this.size = size;
    }

    // 新声明的函数构造器指向对象为Object，这不对，应该为Shape
    // 调用Shape.prototype这个属性以利用create以创建一个基于Shape Object的子对象。

    Ball.prototype = Object.create(Shape.prototype);
    
    // 上述操作会导致constructor为Shape，需要进一步精细定义以更正这个问题
    // 现在使用Ball构建对象其继承会是Ball了。
    
    Ball.prototype.constructor = Ball;

    // 如果需要在原型链上开个口子以让所有使用Ball构造的对象都有draw函数则可以这样做。
    
    Ball.prototype.draw = function () {
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.arc(this.x,this.y,this.size,0,2*Math.PI);
        ctx.fill()
    };

可以看到，这种方法对于使用基于类的语言的开发者很方便，但是对于JS却是一种困扰，其没有突出JS的基于原型的概念，实际上，任何深入学习JS的人都会被Object.create()以及字面量声明对象、基于构造器的对象搞得一团浆糊。过于“友好”有时候是一种灾难。我在开始学习的时候就很自然的把函数构造器当作类来理解，发现没什么问题，尤其是ES5添加了class的内容，一如既往，这依旧是JS一致性较差的证明，为了网页端开发的快速上手与交互需求，其语法必然要牺牲一致性以提高新手可用性。

此外，JS不存在多重继承，哪怕你觉得它就是在干多重继承的事，实际上，它不会正常工作。

## 3.5 prototype、--proto--和constructor属性、Object.create()方法细节

在查阅手册的时候会发现很多名称为 `Object.prototype.valueOf()` 这样的字段，以及 `Object.keys()` 这样的字段。对于前者，其默认可以通过原型链的传递而被其继承的任何对象调用，而后者则只能被其本身调用。

ECMA标准中没有一个叫做--proto--的属性，但是几乎所有的浏览器引擎都支持它，调用一个对象的这个属性，其返回对象的原型对象，原型包括一个constructor和另外嵌套的一个--proto--属性，点开这个--proto--属性则又是一个constructor和--proto--属性。这就是整个原型链，最终的proto属性将返回Object{}对象，这是原型链的根源。调用每个节点的constructor字段都返回其字段的构造函数，即调用 `objinst.--proto--.constructor` 可以返回这个对象的构造器函数。调用嵌套的--proto--可以返回整个原型链。

prototype属性和--proto--属性很容易混淆，但是其区分很简单，因为只有函数/构造函数（伪类）才有prototype属性。调用prototype属性可以返回通过此伪类生成的对象的原型。调用Object.prototype可以返回所有属于Objcet以下的原型链，这有很多，但是调用一个我们自己构造的构造函数类名的prototype属性则只返回从这里往下的原型链，这就比较少了，这些都是我们在构造函数中定义过的方法。

    function Person(){ this.name = "Unset" }; var mw = new Person(); mw.--proto-- === Person.prototype // Returns true
    
    mw.--proto--和Person.prototype均返回一个对象：
    {
    constructor: function Person()
    --proto--: Object { … }
    }

调用 `ClassName.prototype.attr = function(){}` 可以定义这个伪类以及其所有继承的对象的attr方法。使用 `ClassName.prototype.constructor` 可以返回构造器函数，和其继承对象调用--proto--属性返回的一样。

o2 = Object.create(o1)是如何工作的呢？55其传递进去一个对象，然后根据这个对象生成新的对象，后者继承所有前者的方法和属性。其内部工作原理如下：首先对o2新建一个空对象{}，之后设置其--proto--属性为o1整个对象，这样就完成了继承。如下：

    var o1 = {name:"Marvin"}; var o2 = Object.create(o1);

    // o2 Returns : {--proto--:{name:"Marvin"}

如果是一个ClassName.prototype被传递给create方法，则其传出结果中不仅包含了--proto--属性，还包含constructor属性，这也就是基于伪类的继承中，父子伪类需要重新定义子伪类的prototype以及prototype.constructor属性的原因。[见3.4使用函数构造器来实现构造函数的继承细节**]


## 3.6 Setter、Getter、defineProperty、Apply和Delete

显然，这里的方法是用来给对象打补丁用的，为一个已经创建的方法新建一个属性或者方法。同样，这里是不推荐的，但是，不可否认，在很多情况下棋还是较为好用的。

setter和getter分别可以将一个传入的参数设置为对象的一个属性以及得到对象属性，如下：

    var o = {
        a: 7,
        get b() { 
            return this.a + 1;
        },
        set c(x) {
            this.a = x / 2
        }
    };
    console.log(o.a); // 7
    console.log(o.b); // 8
    o.c = 50;
    console.log(o.a); // 25

在某些情况下，这是比较好用的，比如对于日期，想要为Date()添加一个新方法，可以这样做：

    var d = Date.prototype;
    Object.defineProperty(d,"year",{
        get:function(){return this.getFullYear()},
        set:function(y){this.setFullYear(y)}
    });
    var today = new Date();
    console.log(today.year);
    today.year = 2020;
    console.log(today.year);

虽然一般不建议修改顶层Object的方法。这就是使用get和set的两种方式，要么使用set: funtion(x){}，要么使用set d(x) {}。前者对应匿名函数，后者对应具名函数，在不同情境下使用。

最后，你可以使用delete a.attr 来删除一个并非继承而来的属性。

我们可以使用：Function.prototype.apply() 这个语句来给某个函数/对象的原型链添加一个方法，比如：

    var numbers = [5, 6, 2, 3, 7];

    var max = Math.max.apply(null, numbers); // expected output: 7

    var min = Math.min.apply(null, numbers); // expected output: 2

Apply和call函数是姊妹函数，前面见到过，call类似于Pyhton的super，可以在子类中调用父类的方法。apply类似于Python的setattr方法，其可以将一串参数赋值给一个对象，其第一个参数为当作this来看待的对象，其可以用来作为构造函数的通用调用器。

    function trivialNew(constructor, ...args) {
        var o = {}; // 创建一个对象
        constructor.apply(o, args);
        return o;
    }

    var bill = trivialNew(Person, "William", "Orange");

    var bill = new Person("William", "Orange"); // 这两句话是等效的

## 3.7 对象的比较

这是一个很无聊的话题，因为对象并非有自己的属性，因此新建两个一样的对象并不会在运算符中被判断相等，但是，对于引用的对象，其会被判断相等。

    var fruit = {name: "apple"};
    var fruitbear = {name: "apple"};
    fruit == fruitbear // return false

    var fruit = {name: "apple"};
    var fruitbear = fruit;  
    // 将fruit的对象引用(reference)赋值给 fruitbear
    // 也称为将fruitbear“指向”fruit对象
    // fruit与fruitbear都指向同样的对象
    fruit == fruitbear // return true

## 3.8 减少全局变量的污染

全局变量在JS中起到关键的作用，但是很不幸，这是一个糟粕之处。一个改进方法是声明一个新的对象，然后将所有全局变量均放在此处，然后通过句点表示法来进行访问，但每进行一次访问都需要遍历所有对象属性，这样做的开销较大。

    var MYAPP = {}; MYAPP.flight = { airline : "Oceanic", number : 815 };



<br>
<br>
————————————————————————

> 参考书籍：

《JavaScript语言精粹》，Douglas Crockford, 电子工业出版社

[Mozilla开发者中心-教程](https://developer.mozilla.org/en-US/docs/Learn/JavaScript)

[Mozilla开发者中心-指南](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)

[Mozilla开发者中心-参考手册](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)

> 更新日志：

2018年2月12日 完成入门手册中JS对象部分的学习，通过一个跳跳球小游戏实践了对象在JS中的作用。

2018年2月13日 完成《JavaScript语言精粹》一书的阅读，发现很多自己对于JS理解上的问题、自己架构的不完善性。

2018年2月15日 完成面向其他语言开发者的《JavaScript教程》中数组、字符串、基本语法的查漏补缺，更新了此文。

2018年2月16日 进一步了解“JS对象”，完成本文“进一步了解JavaScript对象”部分，这是新年第一天，我发现这部分内容理解上较为困难，因此花了不少时间。

2018年2月17日 完成“JS对象”的其余部分，详细介绍了prototype和--proto--属性以及伪类、纯函数风格的类调用。

2018年2月20日 又重新读了一遍《JavaScript语言精粹》，完成了一些细节部分的整理。在今天之前，我还打算买一本砖头书-《JavaScript权威指南》来辟邪，但是现在觉得，这本《语言精粹》对我来说就足够了，省下来的时间可以搞搞JQuery以及BS和React。

> 后记:

至此，寒假快要过去，我学习JS的时间也不多了。Python数据清洗的话，我注意到pandas相比较于MongoDB的聚合功能提供的功能更多，加之专业学习要求具备一定的数据分析能力，matplotlib和pandas,以及numpy，因此JS旅途只能在这里告一段落。直到我通过计算机三级网络，完成Socket、TCP/IP的Python API学习，以及CGI和Django的进一步学习之后，这些JS知识才能真正发挥作用。当然，我还希望在下学期能够熟悉一下C++，温习一下Qt，QML和QSS，这又是一个不小的任务。

按道理来说，QML、QSS、Qt和C++/Python对应HTML、CSS、React/JQuery/BS/DOM、JavaScript，这两套框架分别跨了客户端以及Web端，统称前端。而MongoDB/MariaDB/IndexedDB这些SQL和NoSQL以及Node.js/Django这些Web Framework则提供服务端支持。嗯，我觉得计算机技术方面，以上就是我的技术栈了。而比如图像视觉、自然语言识别、算法，机器学习，这些地方我所能做的就是调调包，刷刷API之类，进一步的学习属于内功，急不得，需要慢慢搞。对于数据获取、清洗、分析，这些涉及信息获取、加工和过滤以及意义解释的东西，这些和我的心理学专业有关，尤其是心理测验和进化论/进化心理学框架，这也是重头戏，都是应用层的东西。

韶华易逝啊。

> 引言补充：

引言有很多话没说完，放在这里说...

我学习Python也是如此，硬着头皮翻一本《Linux/UNIX编程》的闲暇，偶然间在图书馆找资料的时候看到一本《像计算机科学家一样思考Python》，就顺手拿了过来，在自己电脑上鼓捣一阵子，来了兴趣，又入手了《Python参考手册》，囫囵吞枣的搞了三分之一个暑假，之后又陆陆续续搞了几周，就结束了。

我学习Qt的过程更是好玩，因为自己稍微具有了一点Python基础，所以又跑到图书馆翻看一些书，一会儿想要学习正则表达式，有一会儿看到一本Python Qt快速编程，一会儿又从头开始学习PyGame，想要自己写个小游戏。最后的结果是，我花了一天在图书馆用Qt写了一个简单的GUI，用来测试正则表达式，之后便迷上了Qt，硬着头皮读完了那本Python Qt快速编程，这本书写的很好，刚开始读的时候我还不会查文档，甚至不知道文档为何物，而慢慢的，我开始在读DEMO代码的时候一句一句的查文档，搞清楚流程和具体含义，比如各种部件的常用方法、调色板、小图标、鼠标、事件、键盘、图像绘制、菜单、按钮、多窗体和富文本等等。即便是我每周不间断的学习Qt Python编程，我依然用掉了近两个月时间才完成那本书的大部分内容学习。

现在想想当初还是挺胆大的，GUI是一个巨大无比的坑，Qt的客户端框架，不说精通，就连使用对于一般人而言则是望而却步的事情。Qt整合了很多C++的东西，本身庞大无比，同时又提供跨平台兼容性。但是，学习虽然枯燥，收获也是很有成就感的，通过Python驱动的Qt，我写过很多程序，有自动切换服务器登录校园网的，有替换剪贴板文字并去除空行的，有提醒我写日记并打印日记和发送邮件的，有用于心理学问卷生成的，成就感还是不错的。

虽然多线程、网络编程以及QML、QSS方面是个遗憾，还没有学习（已经大概了解了C++语法，并且有何HTML、CSS、React、BS、JQuery同时学习的计划，同时学习的原因是我认为客户端编程的整体设计模式是相通的），但是，正是这一个个的鲁莽冲动，我更加深入的了解到计算机的软件以及互联网，我们赖以生存的信息世界，是如何工作的，这对我影响很大。

我觉得最大的影响在于，我的编辑器，从2016年暑假试图学习JavaScript但失败以来，从2017年中旬学习Python以来，从文本编辑器到NotePad++，到2017年10月的Visual Studio Code 和 Qt creater 到2018年的WebStrom、 PyCharm、 Qt creater 和 VSCode。现在我可以利用互联网获取和处理更多的信息，可以用PC进行在一些条件下的事务处理和杂事管理。我学会了看文档，基本上也不再使用Office，电脑界面除了浏览器、我自己写的程序、Acrobat、编辑器和IDE、守望先锋之外再也没有其余。

当我回望10天前还对于JS一窍不通的自己，当我回望去年还头痛于JS为什么这么难的自己，当我回望半年前还不知道编程为何物的自己（虽然我曾经使用VB.NET写过一个离线数据库GUI应用，不过那个程序基本上连错误处理都没有，嗯，我甚至没有学VB.NET编程课，开学第一节直接翻到最后一章“连接数据库”看了一遍就开搞，在VS中自己拖控件，写一点句子，想要实现什么功能就去百度，完全不知道世界上有文档这个东西），我很诧异怎么自己已经不在那个最喜欢我的计算机老师的大二课堂，怎么就已经到了大四并且毕业，怎么就已经到了另一所学校，怎么就开始了新的生活？