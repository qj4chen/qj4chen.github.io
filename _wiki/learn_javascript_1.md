---
layout: post
title: JavaScript高级程序设计概要(上)
categories:
  - JavaScript
  - 读书笔记
---

> 我在重新阅读过《像计算机科学家一样思考Python》一书并且总结编程范式后，找到了JavaScript这门像Pyhton一样前后端通吃、兼容性强、应用广的语言。这篇文章是我在学习JavaScript中整理的一些笔记和心得。

我在阅读的时候主要参考了谋智官方的JavaScript基础、中级指南以及面向其他语言使用者的教程，以及JSON格式发明者，ECMA委员会成员Douglas写的《JavaScript语言精粹》一书，这本书让我深受启发。注意：本文只能够提供给你一个逻辑清晰的语言语法架构，按照这样的顺序进行组织：变量、类型和结构、操作符、语句、表达式、条件，循环，迭代判断和函数基础。对于类型和结构而言，按照其基本顺序进行介绍：定义、迭代和分片、元素的选择、增删改以及一些高级用法和陷阱。

Ps.为什么这篇文章叫做高级程序设计(上)呢？因为它原本叫做“JavaScript学习笔记”，但是随着使用和学习的进一步深入，其慢慢涉及了一些高级内容，但是又没有完全涉及涉及这些高级内容内部机制的内容，因此分成(中)、(下)放在之后学习过程中根据项目需要慢慢填坑。

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

不推荐使用for...in语句。此语句用于遍历object内的属性时，其难以区分私有属性和prototype的属性，并且属性的顺序是随机的，即便设置一个属性为不可枚举，但是如果其实例具有此属性的话，也会显示出来，这造成了很大的混淆。对于类似于Python之类的在列表、字符串内的枚举，JS的for...in会产生这样的结果，还算正确：

    var x=0; var y = ["233","I love JavaScript"];
    for (x in y){console.log(y[x])};

    =>  233
    =>  I love JavaScript


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

> 对象的种类

对象有三种，其一为用户自定义的对象，称之为User-defind Object，其二为Native Object，比如Bool()，是JS自带的对象，其三为Host Object，是浏览器驱动的对象。

关于对象的高级使用参考第三部分。

## 1.6 函数

函数有参数、返回值以及里面的语句，其涉及全局变量和局部变量的问题，一般而言，对于JS，常常采用var声明变量，在函数内声明的var变量是局部变量，否则，就是全局变量(如果在全局能够找到的话)。

    function calc_sales(units_a, units_b, units_c) {
        return units_a * 79 + units_b * 129 + units_c * 699;
    }; // 一个有返回值的函数

    calc_sales(2,2,3) // 通过这样调用即可。

还有一种称之为匿名函数，类似于Pyhton的lambda，如下：

    var btn = document.querySelect(".do");
    btn.onclick = function() {
        do something when push the button;
        like "document.addEventListener("click",function_name)"
    }

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

# 2 JavaScript客户端API

## 2.1 DOM简介

### 2.1.1 HTML

> 欲了解和在线测试HTML更多细节，请访问官方文档：https://developer.mozilla.org/zh-CN/docs/learn/HTML/Introduction_to_HTML

Document-Object-Model,简称为DOM。其将HTML、CSS和JavaScript结合起来了。

对于HTML来说，其组成元素为一个树状结构，比如：

    <!DOCTYPE html>
    <html lang="en">
    <head>
    <meta charset="UTF-8" />
    <title>getElementsByTagName example</title>
    <script>
        function getAllParaElems(){
            var iobj = document.getElementsByTagName('p');
            var len = iobj.length;
            alert("there is "+len+" p");
        }
    </script>
    <script src="main.js"></script>
    </head> 
    <body style="border: solid rgb(36, 204, 36) 3px">
    <p title="aaa">Some outer text</p>
    <p title="222">Some outer text</p>      

    <div id="div1" style="border: solid blue 3px">
        <p>Some div1 text</p>
        <p>Some div1 text</p>
        <p>Some div1 text</p>     
    </div>

    <p>Some outer text</p>
    <p>Some outer text</p>

    <button onclick="div2ParaElems();">
        show all p elements in div2 element</button>
            
    </body>
    </html>

其中，对于任意标签来说：

    <p title="XXX" class="233" id="666">YYY</p>

其中，p为元素节点，这些元素是嵌套的，比如html为根元素，而嵌套在里面的有两个次级元素，分别是head和body，p为次次级嵌套元素。对于一个标签来说，至少要有一个元素节点组成，元素可以有属性，称之为属性节点，其包含在元素标签前半段内，最后半段的标签用来说明元素的终止，划分其范围。YYY称之为文本节点，其会显示在页面上。

有两个特殊的属性节点称之为class和id，其被设计用来标识和选择节点。

### 2.1.2 CSS

> 欲了解和在线测试CSS样式表的更多内容，请访问官方文档：https://developer.mozilla.org/zh-CN/docs/Learn/CSS/Introduction_to_CSS

CSS是样式表Customer Style Sheet的简称，其用来处理HTML各节点元素的表达，比如颜色、形状、动画、字体、位置等。CSS类似于JS的语法，如下：

    h1 {
        font-size: 2.6rem;
    }

    p {
        font-size: 1.4rem;
        color: red;
        font-family: Helvetica, Arial, sans-serif;
    }

不过，和JS语法不同，其字段的属性不需要为特定类型，但分号表示断行，大括号表示区域划分还是一样的。

可以使用 #id 和 .class 来根据属性节点的id/class快速提取节点文本。

### 2.1.3 JavaScript

对于JS来说，其内涵了document这个Object实例，可以操纵HTML页面的元素，一些常用的函数及方法如下：

    document.getElementById("id"); // 根据ID来选择文档中的元素
    document.getElementsByTagName("p"); //根据元素名称来选择，比如p段落，h1一级标题等。
    document.getElementsByClassName("class_name"); //根据class名称选择元素。

对于上述选择到的Element节点，使用.getAttribute()方法来进行属性的值的提取，也可以使用.setAttribute()方法来对其属性进行设置。需要注意，因为document已经加载，但是这个方法亦然更改了元素属性，这表示文档是动态变化的，这赋予了网页丰富交互的能力。

注意：一般来说，JS脚本一般在HTML文档最后加载，因为这可以首先展示网页内容，提高响应速度。其次，很多时候，必须文档加载完毕后我们才可以对其内容，节点进行操作，否则首先执行脚本，而文档还没有加载出来的时候，是无法提取document任何元素的。

> JS文档在HTML文档中的位置有三种

内部的JS如下所示:

    <script>

    // JavaScript goes here

    </script>

外部的JS如下所示：

    <script src="script.js"></script>

内联的JS如下所示：

    function createParagraph() {
        var para = document.createElement('p');
        para.textContent = 'You clicked the button!';
        document.body.appendChild(para);
    }
    
    <button onclick="createParagraph()">Click me!</button>

内联是一个坏主意，而且它还不高效——你会需要在每个想要 JavaScript 应用到的按钮上包含 onclick="createParagraph()" 属性。相反，你可以使用内/外部JS动态修改属性：使用一个纯 JavaScript 结构允许你使用一个指令来选取所有的按钮：

    var buttons = document.querySelectorAll('button');

    for(var i = 0; i < buttons.length ; i++) {
    buttons[i].addEventListener('click', createParagraph);
    }

<br>

## 2.2 事件

### 2.2.1 HTML对象的事件

JavaScript事件中，作为前端最为重要的是浏览器定义的Event API，这些因浏览器引擎支持不同而不同。还有一类像是Vue.js，其作为服务器后端定义了事件的API。事件API是用户和DOM进行交互的产物，一些常见的API如下：

对于一个HTML页面上的按钮而言：

    <button class="btn1">Press Me</button>

    <script>
        var btn1 = document.querySelector(".btn1");
        btn1.onclick = function(event) {
            do something;
            console.log(event);
            console.log(event.target);
        }
        btn2.addEventListener("click",function) // 这种方法也可以，但是function不能加括号。
        btn2.removeEvent... // 可以删除事件连接
    
    </script>

`btn1.onlick` 表示按下按钮的事件，传入此lambda函数后，当浏览器检测到按下按钮时会执行函数内容。除此之外，还有鼠标按下、移入(onmouseover)、移出(onmouseleave/out)、聚焦(onfocus)、失去焦点(onblur)、双击(ondblclick)等事件。此外，window这个对象也有一些事件，其中最常用的是键盘按下(onkeydown)、抬起(onkeyup)的事件。这些事件都是浏览器API定义的。

此外，对于同一个对象的一种事件定义多个函数的话，其只会执行最后一个函数，前面的赋值都会被复写掉，比如：

    btn1.onclick = function1;

    btn1.onclick = function2;

    btn1.onfocus = function3;
    // 只有最后两句会被执行

最后，btn.event 后面可以接上匿名函数，也可以接上一个定义过的函数，但是这个之前定义的函数不能加括号，否则会被立刻执行，如果有参数，则需要嵌套一个匿名函数执行。

    var function2(*args) = {};
    
    btn1.onclick = function2(*args); // 错误示范
    
    btn1.addEventListener("click",function2(*args)) // 错误示范
    
    btn1.onclick = function(){function2(*args)} ; // 正确示范
    
    btn1.addEventListener("click",function(){function2(*args)}) //正确示范

### 2.2.2 Event对象和应用

注意到，在上述例子中，function有一个叫做 `event` 的参数，其也可以写成 “e” ，类似于Python Qt 中的 `def CloseEvent(event):` 声明。当事件被捕获后，其会将这个事件的各种参数传递到这个函数中，存在于event这个变量中，这个变量是一个对象，你可以在控制台查看它的详细信息。

这些event的属性中，最有用的是event.target属性，其表示了事件来自的对象，你可以对其各种属性，比如id、title、textContent进行操作。比如设置: `event.target.style = {}` 。事件可以中断，使用 `e.preventDefault` 进行中断，典型应用就是用户名密码检查，如果不符合格式就不进行Submit。

最后介绍一下现代浏览器的Event Capturing 和Event Bubbling流程，对于前者，浏览器从HTML root标签开始遍历查找event类型，比如我们声明的"onclick"事件在一个三级结构中:

    <html>
        <div onclick>
            <video onclick> </video>
        </div>
    </html>

这个问题在于，当我们点击视频的时候，会自动触发套在其上的div标签，因此这两个标签：div和video很难选择。当代浏览器在查找event的时候是从html标签开始向内查找，一直找到所有的onclick事件为止。接着是Event Bubbling过程，如果浏览器有一个onclick事件被触发，则从内向外进行相应，也就是从video标签到div标签进行相应，如果视频在播放，这时候点击，视频标签会被选中，然后才是div标签，这就对这两个嵌套标签的相同事件进行了区分。

对于较老的浏览器，比如IE，可以使用 `e.stopPropagation()` 进行处理。

这为我们提供了一个很绝妙的操作：当一个<ul>标签嵌套<li>标签时，只需要对ul进行事件的定义，而使用 `event.target.nodeName === "LI"` 进行子节点的判断，就可以对子节点进行操作。


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

o2 = Object.create(o1)是如何工作的呢？其传递进去一个对象，然后根据这个对象生成新的对象，后者继承所有前者的方法和属性。其内部工作原理如下：首先对o2新建一个空对象{}，之后设置其--proto--属性为o1整个对象，这样就完成了继承。如下：

    var o1 = {name:"Marvin"}; var o2 = Object.create(o1);

    // o2 Returns : {--proto--:{name:"Marvin"}

如果是一个ClassName.prototype被传递给create方法，则其传出结果中不仅包含了--proto--属性，还包含constructor属性，这也就是基于伪类的继承中，父子伪类需要重新定义子伪类的prototype以及prototype.constructor属性的原因。[见3.4使用函数构造器来实现构造函数的继承细节**]


## 3.6 Setter、Getter、defineProperty和Delete

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




<br>
<br>
<br>
————————————————————————

> 实践指南：

谋智提供了官方的《JavaScript第一步》入门指南，其中包含了一个很有意思的“猜1-100以内的一个数字”小游戏。这个游戏将会用到一些DOM和JS基础知识，比如语句、判断、表达式、函数，这对复习所学很有帮助。此外，指南还提供了一个在“浏览器调试控制台”查看错误并解决问题的说明文章，这对实践来说非常有帮助。你可以点此试一试自己的水平：https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/First_steps

此外，不同于很多官方文档，谋智的JS手册提供的Guide很好玩，里面有丰富的例子和实践，在此，你可以学习到很多HTML和JS交互的内容。你可以使用浏览器F12里面的控制台进行JS语言特性的练习，而不是自己写一个JS文件，然后放到浏览器中跑，因为JS运行错误只有在控制台中会显示，这对于调试来说非常方便。

> 更新日志:

2018年2月8日 阅读《DOM编程艺术》，查阅文档，撰写了基本语法和DOM相关部分内容。

2018年2月9日 阅读谋智的JS入门手册，跟着做了一些练习，对字符串、循环与条件、HTML和JS的联动有了一些实践和深入认识。吐槽：相比较Py，JS的一致性较差，虽然对于熟手来说这不是问题，但设计理念的不同还是造成了其应用范围的差异。

2018年2月10日 完成谋智的JS入门手册中函数/事件部分学习，完成了一个简要图库的设计，更新了这部分的文档。我发现Event这东西和Qt的信号/槽机制模型非常类似，因此学习起来也不太困难。最后，我发现WebStorm和PyCharm对于语法补全和检查的功能太棒了，所以正全面转向IDE（写作此文时正在使用VS Code）。

2018年2月12日 完成入门手册中JS对象部分的学习，通过一个跳跳球小游戏实践了对象在JS中的作用。

2018年2月13日 完成《JavaScript语言精粹》一书的阅读，发现很多自己对于JS理解上的问题、自己架构的不完善性。

2018年2月15日 完成面向其他语言开发者的《JavaScript教程》中数组、字符串、基本语法的查漏补缺，更新了此文。

2018年2月16日 进一步了解“JS对象”，完成本文“进一步了解JavaScript对象”部分，这是新年第一天，我发现这部分内容理解上较为困难，因此花了不少时间。

2018年2月17日 完成“JS对象”的其余部分，详细介绍了prototype和--proto--属性以及伪类、纯函数风格的类调用。

