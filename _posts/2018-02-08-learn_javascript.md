---
layout: post
title: JavaScript学习笔记 - 语法篇
categories:
  - JavaScript
  - 读书笔记
---

> 我在重新阅读过《像计算机科学家一样思考Python》一书并且总结编程范式后，找到了JavaScript这门像Pyhton一样前后端通吃、兼容性强、应用广的语言。这篇文章是我在学习JavaScript中整理的一些笔记和心得。

我在阅读的时候主要参考了谋智官方的JavaScript基础、中级指南以及面向其他语言使用者的教程，以及JSON格式发明者，ECMA委员会成员Douglas写的《JavaScript语言精粹》一书，这本书让我深受启发。注意：本文只能够提供给你一个逻辑清晰的语言语法架构，按照这样的顺序进行组织：变量、类型和结构、操作符、语句、表达式、条件，循环，迭代判断和函数基础。对于类型和结构而言，按照其基本顺序进行介绍：定义、迭代和分片、元素的选择、增删改以及一些高级用法和陷阱。

# 1 JavaScript 语法概要

## 1.0 综述

JavaScript像别的语言一样，都提供了三种基本的功能：

- Math：接收输入、存储和计算一些变量的值；

- Loop：JS允许使用事件监听，当你点击一个按钮或者进行某些操作时会自动触发；

- Operation：JS允许对HTML网页元素进行提取、改变和进一步处理；

本文所讲的JavaScript为客户端JS，其含义为下载到用户的计算机上并执行，用来改变网页外观和状态、相应用户行为的动态语言。还有一类称之为服务端JS，比如Node.js,用来进行服务器和用户的交互。

JS原生的三大类Application Programming Interface，其分别为DOM(Document Object Model) API，提供操纵HTML和CSS文档的能力、Geo API，提供地理位置标注的能力，Canvas/webGL API，提供2D、3D绘图的能力，Audio/Video API，提供音频和视频处理的能力。此外还有第三方API，比如各种天气、快递查询API等。

> 更详细了解JavaScript的具体细节，请访问：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript 查阅详细文档。

JavaScript是一门毁誉参半的语言，其设计之目的就是为了互联网前端的交互展示，因此掺杂了很多特性，由于W3C和ECMA，网景和微软的各种利益争夺，这门语言看起来不伦不类，其中包含大量的毒瘤和糟粕，比如全局变量，为了满足基于类的使用者方便上手搞出来的构建器等，但是这门语言也有很多精华，比如基于原型的对象，对象字面量等。因为其所处地位的重要性，使得Web前端不得不学习这门语言，因此取其精华，弃其糟粕是一个好主意。

JavaScript的很多写法都是可以运行的，比如未经定义的变量、未加分号的语句以及在声明之前就调用的函数，这些在运行中不会有什么问题，但是却是不推荐的，因为在大型项目中可能存在难以调试的各种潜在问题。对于一个工程，能否运行是关键，但是，我们认为语言风格更代表了一种可贵的品质。JavaScript饱经诟病，并认为是一门玩具语言，但讽刺的是，这在很大程度上是由于使用者造成的，由于缺乏对语言的深入理解和良好的编程风格，导致的各种错误和问题，当然这也不是说JS语言没有缺点，相反，其优缺点分明，我们完全可以避开缺点。因此，强烈建议使用IDE，打开语法格式检查，按照一个最小子集的标准来写JavaScript脚本。

## 1.1 书写规则

### 1.1.1 语句和空白

JS的语句区别于Python，没有强制的换行要求；JS会自动添加分号进行语句的分隔（ASI），但是为了避免使用上的麻烦以及以后调试和维护的方便，推荐使用分号进行语句的分隔。

    var a = 5; var b = "hello";

### 1.1.2 注释

JS的注释标准规范为前两种，第三种对于多行的JS来说，需要在每一行的开头加<!--,但是不要求加后面的类似于HTML的封闭符号，不建议使用。由于采用IDE和快捷键，这里不推荐使用除了双井号的其余任何注释方式。

    // 这是注释 （唯一推荐使用）
    
    /* 多行注释 
    这是多行的注释
    */

    <!-- HTML注释方式 -->

## 1.2 变量

不同于Python不需要声明变量，对于JS来说，其设计之初，为了方便脚本运行以及简易，所以可以不用声明直接使用，但是，在严格模式下这样会导致Reference Error，因此，强烈推荐使用var语句声明变量，刚声明的变量值为undefind，需要赋值才可以正常使用。

    var variable_a , variable_b , variable_c = variable_d = 5 , variable_e = "Hello"

变量分为全局变量和局部变量，具体请参见本页[函数/全局变量和局部变量]部分。


## 1.3 操作符

算术操作符有：

    1 + 1 - 1 * 1 / 1; //加减乘除均可

    var a = 0 ; a++ //++表示累加1，--表示累减1

    var b = 0 ; b+=2 //+=，-=是一般语言的简要运算表示方法

字符串拼接：

    "a" + "b" = "ab";

    "a" + 2018 = "a2018"; // 这个要注意

比较操作符有：

    <=, >= , == , === , !=== , != // 其中三个等号表示其类型也必须相同，而两个等号表示其值相同，比如 null 等于 ""。

逻辑操作符有：

    && 与 ；|| 或 ，!() 非。

in操作符，这个操作符很奇特，区别于Python，其不可以在一个字符串中搜索子字符串，也不能用来在列表中搜索列表的值的下标，更不能用在对象方法中寻找对象方法的值，但是，其可以判断列表下标、对象的方法名称，如下：

    var trees = new Array("redwood", "bay", "cedar", "oak", "maple");
    0 in trees;        // returns true
    6 in trees;        // returns false
    "bay" in trees;   

    var myString = new String("coral");
    "coral" in myString; // returns false

    var mycar = {make: "Honda", model: "Accord", year: 1998};
    "make" in mycar;  // returns true
    "model" in mycar; // returns true

in 操作符只可以用来检测对象中是否存在某个属性，而不能用来在字符串、数字或者其他基本类型的数据中进行检索。并且对于对象的属性检索，也需要保证对象不为null/undefined，否则会报错：

    var foo = null;
    "bar" in foo;
    // TypeError: invalid 'in' operand "foo"

备注：不建议以任何方式使用in操作符以及for...in结构块。

## 1.4 条件、嵌套、递归

### 1.4.1 IF、Switch、三元表达式和块级作用域

if语句的表示和Python类似，不过采用括号而不是缩进代替内部语句范围区分，除此之外，不使用冒号而是用括号表示表达式运算：

    if (expression){
        do somethings;
    } else if (expression) {
        do something;
    } else {
        do something;
    }

另外介绍一个和If很像的条件判断，叫做switch：

    switch (var_name) {
        case value1 :
            do something;
            break;
        case value2 :
            do something;
            break;
        default:
            do something;
    }

这就相当于一个简化版的If语句，`var_name` 字段放置变量，而 `value` 字段则只放置值。需要注意的是，如果在case分支中不加 `break;` 字段的话，程序会自动执行其余的语句，哪怕他们不符合剩下语句case的判断要求。`defualt` 字段表示如果都不符合，则执行此段语句，因为我们一般会在平常的 `case`字段中使用 `break;` ，所以也就不会执行 `default;` 字段了。

三元操作符如下：

    var a = b = 0;

    (a = b) ? if yes then do something ; else do something

第一个字段为判断条件，后面用问号隔开，第二个字段为这个条件为真时的操作，只能写一句话，可以放个函数，第三个字段是条件为假时的执行语句，第二、三个字段采用分号区隔。最后，第三个字段不能嵌套另外一个三元操作符。

备注，在条件语句中不存在本地变量和全局变量的区别，所有变量均为全局性质的。使用let(ECMA6)可以定义块级作用域，只在{}中起作用。一般使用var在除了函数之外定义的任何变量均为全局变量。

> let和块级作用域

    var x=10; if (1+1 !== 2) {} else {var x =20}; x; // Result: 20

    var x=10; if (1+1 !== 2) {} else {let x =20}; x; // Result: 10

### 1.4.2 while、label、continue和break

while也很简单：

    var x = 1
    while (x < 10){
        console.log("x is small than 10");
        x++
    }

特别的，有一个叫做do...while的循环，其不论如何，循环先进行一次，接下来才进行判断。

    var result = '';
    var i = 0;
    do {
    i += 1;
    result += i + ' ';
    } while (i < 5);

for循环不太一样，其在expression字段指定了三个语句，分别是初始条件、判断条件、迭代语句，如下：

    for (var count = 0; count < 5; count++){
        do something.
    }; //这句话的意思是，声明一个count变量为0，然后count执行第二个语句的判断，如果小于5则执行语句，之后累加count，继续这个循环，一直到count不小于5，自动结束。

    var i = 0;
    for (; i < 9; i++) {
        console.log(i);
        // 当然，你可以刻不写初始值，然后在外面界定。
    };

    for (var i = 0;; i++) {
        console.log(i);
        if (i > 3) break;
        // 同样的，你也可以不指定判断值，然后在中间嵌套一个if break语句。
    };

    var i = 0;
    for (;;) {
        if (i > 3) break;
        console.log(i);
        i++;
        // 我已经不想说话了...
    }

区别于Python，break和continue对于使用label标识的循环来说更为方便，其可以决定label标定的循环的终止或者重新开始，如下：

    label:
        statement;

    var x = 0;
    var z = 0
    labelCancelLoops: while (true) {
        console.log("外部循环: " + x);
        x += 1;
        z = 1;
    while (true) {
        console.log("内部循环: " + z);
        z += 1;
        if (z === 10 && x === 10) {
            break labelCancelLoops;
        } 
        else if (z === 10) {
            break;
            }
        }
    }

对于break和continue来说，其一般用于打破当前循环，但是label的使用可以让break/continue label打破外部循环。比如下面这个，break cmLoop中如果只使用break的话，程序不会停止（因为只有for循环被终止了，while循环还没有停止）。

比如：

    var i = 0; 
    cmLoop: while (true) { 
        i += 1; console.log("NOW i is ",i); 
        for (let q=0; q<20; q+=2){
            console.log("q is ",q);
            if (i==10){
                break cmLoop; 
                // 这里不能换成break，否则无法在这里终止外部的while循环
                // 因此必须定义label以指定语法块以终止。
            }
        }
    };

一个等价的Python语句为：

    i = 0
    while 1:
        i += 1
        print("NOW i is ",i)
        x = 0
        for t in range(10):
            x += 2
            print("x is",x)
            if x == 20 : break
        if i == 10:break


> 注意：

不推荐使用for...in语句。此语句用于遍历object内的属性时，其难以区分私有属性和prototype的属性，并且属性的顺序是随机的，即便设置一个属性为不可枚举，但是如果其实例具有此属性的话，也会显示出来，这造成了很大的混淆。对于类似于Python之类的在列表、字符串内的枚举，JS的for...in会产生这样的结果，还算正确，但其实，由于JavaScript并没有一个真正可以称之为List的东西，其Array其实是一个伪数组，所以for...in并不能保证数组遍历的顺序。更可怕的是，JS竟然没有一种方法可以直接对数组和对象进行区分。

    var x=0; var y = ["233","I love JavaScript"];
    for (x in y){console.log(y[x])};

    =>  233
    =>  I love JavaScript

    typeof []; // Returns object 233333333333


## 1.5 类型和结构

JS有整数、小数、字符串、布尔值等类型。需要注意，其布尔值类型为“true、false”，开头首字母小写。

JS的类型可以采用下列语句查看：`var a = "Hello" ; typeof a;`

### 1.5.1 undefined 和 null

对于任何声明但没有赋值的语句进行调用，会返回undefined作为其值。对于一个数组进行超过其下标的调用也会返回undefined值，而不会报错。

undefined在bool判断时被看作是false，在数学运算时会得到undefined结果，比如：

    typeof(undefined); // 结果为 undefined

    var a; if (!a) {console.log("Hello Here!")}; // 这句话会被执行

    var a; console.log(a+100); // 这句话的结果为 undefined

对于null，其在bool判断是会被看作false，在数学运算时会被看作0，比如：

    typeof(null)； // 结果为 obj

    var a = null; if (!a) {console.log("null is false!")}; // 可以执行

    var a = null; console.log(a+100); // 这句话的结果为 0+100 = 100

NaN是一个特殊的数字类型，其和任何数字相加都等于NaN。

undefined、null、NaN在和bool值进行表达式判断相等时结果都为否。

### 1.5.2 NaN 和 0

在布尔判断条件下，由上可知undefined和null均为false，起始NaN,一个不属于数字的数组对象和数字0都被判断成为false使用。

### 1.5.3 字符串

字符串常用方法如下，包含了长度判断、分片、查找、替换和变形等常用操作，需要注意，其一般不使用Python冒号，分片采用slice()而非[]。

> 声明、遍历和分片：

`var s = "Hello, World!";`

`s.length` 

显示字符串长度，属性而非方法

`s[num]`

从某个元素下标提取字符元素，其不支持多个字符的提取，即不支持s[m:n]，但可以使用slice进行提取。推荐使用slice而不是使用此种方式，因为容易造成混淆。

`s.slice(m,n)` 

分片，m,n 不写为复制字符串，类似于[:]，只写一个数字为从这个数字开始到最后。写两个数字为从第一个数字到第二个数字，包含第一个数字但是不包含第二个数字对应索引的字母元素。

> 元素的查找、替换和处理：

`s.indexOf("llo")` 

返回查找内容的字符串开头匹配位置

`s.split(",")` 

根据某个元素分片，结果返回一个数组。

`s.replace("World","Marvin")` 

注意，替换返回一个全新的结果，你需要保存这个结果到变量中去。其次，其只会匹配查找到的第一个，而不会匹配其余的。替代方法是使用正则表达式：

`s.replace(/\World/g,"Marvin")` /g 表示全局模式。

`s.toUpperCase()`; `s.toLowerCase()` // 大小写转换 

> 字符串和Number的转换可以这样：

    var num = Number("123455");

### 1.5.4 数组

JS的数组并非是在内存中按照顺序排列的典型“列表”，而是一个类似于列表的东西，其可以按照下标进行选择、切片和遍历，有相应的顺序，但是其效率不如典型的列表好。正因为其伪数组的特性，所以可以使用类似于Python字典的方式： array_a["string"] 进行元素提取，但一般不建议这样使用。

JS的数组表示方法为：[value1,value2,value3] ,其暗含了下标，分别为0，1，2。列表是可以变化的。

新建数组可以采用：
    
    var variable_a = Array(); //也可以声明Array(num)指定列表的大小，不过一般不使用。

    var variable_a = Array(value1,value2,value3,value4); //这种声明方式直接给指定下标赋值。

    var fruits = ['Apple', 'Banana']; //也可以直接采用列表形式创建数组

数组支持分片，可以给其中的元素赋值：

    array_obj[2] = 233

JavaScript的数组会自动吞掉最后一个逗号，比如 `var list_a = [1,2,3,,4,,,];` 。这个数组的长度为7，最后一个逗号之前的才算元素，对于两个逗号隔开的元素，其值为undefined。对于超出实际数组长度的索引来说，list_a[10000] 结果返回 undefined 。
    
    
数组自带一些函数，比如：

`array_obj.length` //一个属性，返回数组的长度，不是方法，不需要加括号。

`array_obj.slice(m,n);` // Make a Copy/ 切片取m到n的数值。

`array_obj.splice(m,n,"String");` // m为要插入的位置，n为替换1/插入0，string为替换/插入内容。

`array_obj.push(item)` //添加到列表结尾, 类似的还有 `a_o.unshift(item)` 添加到开头 

`array_obj.pop()` //删除最后一个元素, 类似的还有 `a_o.shift()` 。从开头删除第一个元素，对于JS，没有remove、del的数组元素删除方法。

注意：数组可以采用 `array_obj["string"] = xxx` 的方式给予一个特殊字段以赋值，而非是下标，这类似于Python的字典，但不完全是，这种方法不推荐使用，因为容易混淆，并且其破坏了数组的结构。

> 数组和其他数组、字符串的转换

数组转换成字符串：

    var myList = ["a",12,[2,1,2]];

    myList.join(";") => a;12;2,1,2 
    
和Python不同，join是Array的方法而不是字符串方法。对于Python来说，如下：";".join(["a","b","c"])。

字符串转换成数组：

    str.split(" ") // 可以从str切片成为一个列表 

    另外还有一个arr.toString() 但是不推荐使用。

数组和数组的合并：

    var list_1 = [1,2,3]; var list_2 = [4,5,6]; var new_list;
    
    new_list = list_1.concat(list_2) => [1,2,3,4,5,6];

    list_1.push(list_2) => Array [ 1, 2, 3, [4,5,6] ];

    list_1 + list_2 => 1,2,34,5,6 //不推荐使用，莫名其妙的错误

> 注意：在数组中慎用算术操作符

不推荐对于数组使用算术运算符，比如＋号，在Python里这样表示的是两个数组的合并，但是在JS里其会默认将两者转换成为字符串然后相加。

对于数组，也不推荐使用 ` += ` 操作符。因为其会将可变的数组强制转化成为字符串然后进行拼合。

> 注意：ECMAScript7 中计划添加类似于Python的 `x for x in [1,2,3,4] if y == 0 else None` 数组推导式，和蹩脚的 `in` 操作符一样，不建议任何人使用（即便浏览器引擎已经支持ES7）。


### 1.5.5 对象

对象是JS一个非常神奇的结构，其采用 { } 表示，内部使用 key-value表示，比如：

    var object = {
        foo: 'bar',
        age: 42,
        baz: {myProp: 12}
        } // 这看起来像是Python字典，但其实不是

    var obj_1 = new Object(); // 这样也可以新建一个对象，这看起来像是Python的一个类，但其实也不是

但是，其可以采用obj_a.attr1这样的句点表示法来提取value1这个值，这类似于Python的类。因此，对于JS来说，其attr1这个字段是可以不用加双引号作为一个字符串的key值的，而value则必须为整数、小数或者字符串，其不能为一个未声明的变量。

操作对象的属性，可以用：

    object.foo; // "bar"
    object['age']; // 42

对象内不仅包含采用句点表示法可以提取的元素/称之为属性，还有一些可供使用的函数。

对象属性的选择面临一个问题，即此属性字段到底存不存在的问题。对于Pyhton而言，可以通过hasattr来判断，而对于JS，如果你尝试检索一个不存在的成员属性的值，将返回undefined。但是如果继续尝试从undefined中取值，则会导致TypeError异常。这时可以采用||和&&运算符：

    var status = flight.status || "unknow"; // ||运算符可以在前者返回undefined时使用后者的值
    var status = flight["status"] || "(none)";

    flight.equipment // undefined
    flight.equipment.abs // TypeError
    flight.equipment && flight.equipment.abs // undefined

对于所有的句点表示法，如果其属性存在，那么属性的第一级不存在的子属性将返回undefined，而对于这个不存在的子属性的属性将报错。

> 对象的种类

对象有三种，其一为用户自定义的对象，称之为User-defind Object，其二为Native Object，比如Bool()，是JS自带的对象，其三为Host Object，是浏览器驱动的对象。对象是可以继承的，继承赋予了对象更多的使用可能性。所有通过继承而来而非私有的属性都存在于原型链中，任何对象都通过原型链连接到Object.prototype中。

关于对象的高级使用参考第三部分。

## 1.6 函数

函数有参数、返回值以及里面的语句，其涉及全局变量和局部变量的问题，一般而言，对于JS，常常采用var声明变量，在函数内声明的var变量是局部变量，否则，就是全局变量(如果在全局能够找到的话)。

    function calc_sales(units_a, units_b, units_c) {
        return units_a * 79 + units_b * 129 + units_c * 699;
    }; // 一个有返回值的函数

    calc_sales(2,2,3) // 通过这样调用即可。

函数传入值不需要像C++一样声明类型，在结构体中的使用时，对于未声明的变量需要使用var声明，对于传入的值的变量则不需要重新声明。

当定义的传入参数多于调用时的参数时，未调用值会被自动忽略，当调用值多于定义的传入参数值时，超出的参数也会被自动忽略。

还有一种称之为匿名函数，类似于Pyhton的lambda，如下：

    var btn = document.querySelect(".do");
    btn.onclick = function() {
        do something when push the button;
        like "document.addEventListener("click",function_name)"
    }

任何函数被创建之后，其函数对象都会链接到Function.prototype上（此对象本身链接到Object.prototype上）。每个函数在创建时都会在prototype属性中被创建一个constructor的属性，此属性的值为此函数的对象。由于JS没有办法判断哪个函数是构造函数/将用来作为构造器，因此对所有函数都有prototype属性...

### 1.6.1 变量和变量提升

对于变量，其声明可以放在使用之后，因为在解释器运行的时候变量会被自动提升。对于函数也是如此，但是匿名函数则不会被提升，比如：

    function1(); var function1 = function(){console.log(23333)};
    // 引用错误，因为匿名函数不能被提升

因此，作为一条严格的要求，推荐所有的变量声明都放在开头或者尽可能放在开头，这样的话就不用在意这个问题。

### 1.6.2 嵌套函数

你可以在一个函数里面嵌套另外一个函数。嵌套（内部）函数对其容器（外部）函数是私有的。它自身也形成了一个闭包。一个闭包是一个可以自己拥有独立的环境与变量的的表达式（通常是函数）。

可以总结如下：

- 内部函数只可以在外部函数中访问。

- 内部函数形成了一个闭包：它可以访问外部函数的参数和变量，但是外部函数却不能使用它的参数和变量。


        function outside(x) {
            function inside(y) {
                return x + y;
            }
            return inside;
        }
        fn_inside = outside(3);

        result = fn_inside(5); // returns 8

        result1 = outside(3)(5); // returns 8

- 一个闭包必须保存它可见作用域中所有参数和变量。因为每一次调用传入的参数都可能不同，每一次对外部函数的调用实际上重新创建了一遍这个闭包。只有当返回的 inside 没有再被引用时，内存才会被释放。

- 当同一个闭包作用域下两个参数或者变量同名时，就会产生命名冲突。更近的作用域有更高的优先权，所以最近的优先级最高，最远的优先级最低。这就是作用域链。链的第一个元素就是最里面的作用域，最后一个元素便是最外层的作用域。

### 1.6.3 函数闭包：构造对象以及更安全的函数

JS函数可以用来替代构造器构造对象。函数访问其上下文、内部外部变量隔离这点形成了函数的闭包，这是JS最为强大的特性之一，举例如下，对于Python的话，则需要声明一个类来实现这个对象。因为JS没有基于类的实现，而是基于原型链，加之其对象可以直接作为字面量返回，因此这赋予了其函数强大的能力。

    var createPet = function(name) {
        var sex;
        return {
            setName: function(newName) {
                name = newName;
            },  
            getName: function() {
                return name;
            },  
            getSex: function() {
                return sex;
            },
            setSex: function(newSex) {
                if(typeof newSex == "string" 
                    && (newSex.toLowerCase() == "male" || newSex.toLowerCase() == "female")) {
                    sex = newSex;
                    }
                }
            }
    }
    var pet = createPet("Vivie");
    pet.getName();                  // Vivie
    pet.setName("Oliver");
    pet.setSex("male");
    pet.getSex();                   // male
    pet.getName();                  // Oliver

等同于Python基于类的语法：

    class Pat():
        def __init__(self,name="Unnamed",sex="Unset"):
            self.name = name
            self.sex = sex

        def setSex(self,sex="Unset"):
            self.sex = sex

        def setName(self,name="Unnamed"):
            self.name = name

        def getName(self):
            return self.name

        def getSex(self):
            return self.sex

    cat = Pat("ketty")
    print(cat.name,cat.sex)
    cat.setName("New Ketty")
    cat.setSex("girl")
    print(cat.name,cat.sex)

函数闭包的另一个优点是，其可以放置外部环境对函数的未授权修改。内嵌函数的内嵌变量就像内嵌函数的保险柜，它们会为内嵌函数保留稳定而又安全的数据以参与运行。而这些内嵌函数甚至不会被分配给一个变量，或者不必一定要有名字。

    var getCode = (function(){
        var secureCode = "233333333";
        // 一个不希望在函数外被更改的变量，可以使用闭包特性进行嵌套返回
        return function () {
            return secureCode;
        };
    })();

    getCode();    // Returns the secret code

### 1.6.4 函数式编程

在函数中，对于变量而言，使用var variables_a = 233 声明的变量为私有的，其不会影响到外部同名变量。如果没有使用var声明，则默认为全局变量，尽可能不要采用全局变量。对于对象而言，如果对象作为一个参数传递到函数内进行了处理，这个改变对于对象来说是可见的，即其会影响到传入的对象本身，尽管这个函数有可能并不返回这个对象。

相反，我们推荐采用纯函数式编程，即在函数内部新建一个对象，之后将作为参数传入的这个对象构建成为这个新对象，之后将这个新对象返回。这样的话，函数所做的对象的改变不会影响到之前作为参数传入的那个对象。


### 1.6.5 剩余参数、默认参数*以及其调用

函数可以接受多个参数，你不一定要在构造函数的时候完全传递它们。对于传递的参数，可以使用arguments[i]来访问，使用arguments.length来进行遍历。比如：

    var myConcat = function (sep) {
        var result = '';
        for (var i = 1; i < arguments.length; i+=1) {
            result += arguments[i] + sep
        }
        return result.toString().trim()
    };

    myConcat(" ","I","Love","JavaScript"); // Results "I Love JavaScript"

arguments并非是一个数组对象，其仅仅是类似而已，并不具备全部Array对象的方法，其之后length和下标索引。除了使用arguments[i]函数来接受多个参数外，剩余参数可以使用“...args”表示。而对于函数参数的默认值而言，则可以直接赋值，比如在之前，需要两个数相加，需要：

    var add = function (a,b) {
        if (typeof b === undefined) {
            b = 0   
        }
        return a + b
    };

    var add = function (a,b=0) {reutrn a + b};

对于剩余参数的写法，可以如下：

    var add = function (a,...args) {
        return args.map(function(element){return a*element})
    }

...args中可以看到args是一个Array对象（区别于arguments的伪Array对象，是一个真的Array对象）。

对于默认参数，可以在声明函数时即可处理，如下：

    var add = function (a=0,b=0) {console.log(a+b)}; add(1,1);

但是显然这和Python过于相似了，传统的JS这样写：

    var add = function (a,b) {var a = a || 0; var b = b || 0 ; console.log(a+b)}; add();

为了体现JS的灵活和一致性，我个人建议采用传统JS的写法，即双竖杠的标识判断符号。

### 1.6.6 胖箭头函数*

ES6中引入了胖箭头函数，这是为了解决this指代和只有变量没有方法的函数写法问题：

    var a = function(){
        var that = this;
        setInterval(function(){that.say = "Hello";},1000)
        // this.say 是错误的，因为这是另一个函数，其this指代已经改变
    }

    var a = function(){setInterval(() => {this.say="Hello"},1000)};

    var a = [1,2,3]; var b = a.map(function(e){return e*e}); console.log(b);

    var a = [1,2,3]; var b = a.map(e=>{return e*e}); console.log(b);

这类似于Python的lambda匿名函数，此外如果没有变量需要传入，则需要写一个空括号，反之则不需要括号，胖箭头函数构造为 `() => {}` .

注意：胖箭头函数是JavaScript的一个特性，由于其替代方案并不会增加多少成本，并且具有较高的跨语言一致性，因此不推荐使用胖箭头函数。

## 1.7 错误处理

在JavaScript中有两种错误处理方法，其一为 `throw expression` 抛出异常，其二为使用 `try...catch(e)...finally` 语句块。

对于throw表达式，一般可以定义自己的错误类或者直接使用现成的错误类 `Error()`。如下：

    try {
        if (1+1 !== 2) {
            console.log("You are right!");
        }
        else {
            throw "ErrorFind";
            throw new Error("ErrorAgain!");
            throw new myException("Find My Error!").toString();
        }
    }
    catch(e) {
        console.log("[ERROR]:",e);
    }
    finally {
        console.log("Finished");
    }

    => [ERROR]: ErrorFind/ ErrorAgain!
    => Finished

比较高级一些的玩法比如定义自己的Error类，如下：

    var myException = function (message) {
        this.message = message;
        this.name = "myException";
    };

    myException.prototype.toString = function (){
        return this.name + " : " + this.message; 
    }

这样的话，返回的结果就是 `[ERROR]: myException : Find My Error!` 。















<br>




<br>
<br>
————————————————————————

> 参考书籍：

《JavaScript语言精粹》，Douglas Crockford, 电子工业出版社

[Mozilla开发者中心-教程](https://developer.mozilla.org/en-US/docs/Learn/JavaScript)

[Mozilla开发者中心-指南](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)

[Mozilla开发者中心-参考手册](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)

> 实践指南：

谋智提供了官方的《JavaScript第一步》入门指南，其中包含了一个很有意思的“猜1-100以内的一个数字”小游戏。这个游戏将会用到一些DOM和JS基础知识，比如语句、判断、表达式、函数，这对复习所学很有帮助。此外，指南还提供了一个在“浏览器调试控制台”查看错误并解决问题的说明文章，这对实践来说非常有帮助。你可以点此试一试自己的水平：https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/First_steps

此外，不同于很多官方文档，谋智的JS手册提供的Guide很好玩，里面有丰富的例子和实践，在此，你可以学习到很多HTML和JS交互的内容。你可以使用浏览器F12里面的控制台进行JS语言特性的练习，而不是自己写一个JS文件，然后放到浏览器中跑，因为JS运行错误只有在控制台中会显示，这对于调试来说非常方便。

> 更新日志:

2018年2月8日 阅读《DOM编程艺术》，查阅文档，撰写了基本语法和DOM相关部分内容。

2018年2月9日 阅读谋智的JS入门手册，跟着做了一些练习，对字符串、循环与条件、HTML和JS的联动有了一些实践和深入认识。吐槽：相比较Py，JS的一致性较差，虽然对于熟手来说这不是问题，但设计理念的不同还是造成了其应用范围的差异。



