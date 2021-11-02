---
layout: post
title: JavaScript学习笔记 - API篇
categories:
  - JavaScript
  - 读书笔记
---
> 这是我学习JavaScript的第二篇文章，在第一篇文章中，重点介绍了JavaScript的基本语法和一些规范，而在本篇中则着重介绍HTML、CSS和DOM的一些API接口，基于事件相应的相关GUI编程范式。本文的最后亦介绍了Ajax——异步JavaScript和XML编程技术，这种技术被设计用来在不重载页面情况下进行数据更新，本文介绍了XHR和fetch两种操作方法。

# 2 JavaScript客户端API

JavaScript客户端API大致分为浏览器提供的API和第三方API，其中浏览器提供的API又分为和服务器交互的Fetch/XHR API、基于浏览器窗口的Notification API和Web Storage API（IndexedDB）以及基于文档的DOM和Canvas、webGL、Audio、Media等操纵文档结构的API。

所有的API都是基于对象的，并且提供了一个入口，比如DOM的入口在document，你可以通过document.attr来进行各种接口调用。比如窗口的入口在window，你可以通过window.innerWidth()调用浏览器窗口大小等。所有的API都使用事件处理状态的变化，比如对于window而言，window.onresize = function(){}可以在窗口大小发生变化时进行操作和相应。基于对象、提供入口、使用事件处理状态变化，这是JS客户端API的三大特点。

> Web浏览器的组成部分

其一是Navigator，其存储着用户的状态和标识，比如是否允许位置、相机、文件、通知的使用。其二是Window，也就是浏览器窗口，在这里JS API进行数据离线存储（IndexedDB）以及event handler事件相应，而在Window内部的就是document，也就是文档，这里是DOM、各种绘图、视频展示和播放的区域。

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


### 2.1.4 基本的DOM操作

基本的DOM操作包括元素的选取、新建、更新、移动、复制和删除。

对于选取而言，一般只推荐使用querySelector("css selector exp")这个来进行元素的选择，对于上面提供的三种方法属于较为老旧的浏览器API，因此并不推荐，除非你的程序有兼容性考虑。

1、对于新建而言，使用 `document.createElement()` 是个不错的选择。

2、对于修改而言:

- 如果是将一个存在的节点移动到某个其他节点之内，可以使用 `node.appendChild(exist Node)`。

- 对于需要在节点内添加文本则可以使用 `node.textContent = ` 来赋值一个字符串，也可以使用 `document.createTextNode()` 来添加文本。

    liitem.textContent = item;
    liitem.appendChild(document.createTextNode(item));
    // 这两个是等效的

- 对于需要在此节点添加新节点，可以使用 `node.appendChild(new Node)`。

- 对于节点属性的修改和添加，可以使用 `node.setAttribute("attr name","attr value")`。

- 删除子节点可以使用 `node.removeChild()`，如果要删除自身，使用 `node.parentNode.removeChild()` 即可。 

3、对于复制而言，可以使用 `document.cloneNode()`。

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

## 2.3 异步JavaScript和XML技术

通常Ajax指的是异步JavaScript和XML技术，这项技术可以在不重新加载页面的情况下，使用JavaScript和XML更新页面的部分内容。早期使用的方法是XMLHTTPRequest，主要操作格式为XML，现在则主要操作JSON格式数据文件。现代浏览器支持fetch方法来进行数据请求和页面加载, 其已经超出Ajax的定义，但广义上，还是使用Ajax指代这种不通过重载页面就进行内容更新的技术。

### 2.3.1 XMLHTTPRequest

使用XHR(XMLHTTPRequest)进行操作的主要步骤如下：

    var r = new XMLHTTPRequest(); var url = "http://info.mazhangjing.com/vrl/data.json"; r.open("GET",url); r.responsetype = "json";
    r.onload = function () { // do something here;}; r.send();

可以看到，XHR是一个构造函数，需要进行对象的创建，然后赋予其方法、URI地址、请求格式、相应后的事件操作函数以及相应请求。

### 2.3.2 Fetch HTTP管道

作为替代的现代版本XHR是fetch() HTTP管道。这个函数通过接受一个Request对象，然后返回一个Promise。

    var myImage = document.querySelector('img');

    fetch('flowers.jpg') //接受一个Request对象，包括地址、方法以及头部等，可以直接写地址，默认GET方法。

    .then(function(response) { //then类似于管道，其接受Promise通过then解析后生成的Response对象
        // 在这里可以定义函数对Response对象进行处理，一般用来检查是否返回成功以及返回对应格式内容。
        if (response.ok) {return response.blob();}})

    .then(function(myBlob) { //then将上一个函数返回的Promise(Body)作为myBlob输入，添加到DOM树中。
        var objectURL = URL.createObjectURL(myBlob);
        myImage.src = objectURL;
    });

当然，也可以直接在第二个then的函数中进行所有处理，使用什么写法取决于个人喜好，更新的例子添加了错误处理和Request构造。

    var myImage = document.querySelector('img');

    // 你可以声明一个header对象，以及添加方法、头部、Data到Request中。
    var myHeaders = new Headers('Content-Type': 'application/json');  
    var data = {username: 'example'};
    var myInit = { method: 'GET',
                headers: myHeaders,
                data: JSON.stringify(data),
                mode: 'cors',
                cache: 'default' };

    fetch('flowers.jpg', myInit) 
    // 这里也可以写成 var myRequest = new Request('flowers.jpg', myInit); fetch(myRequest)...
    // Request的两个参数分别是Info和Init，即地址和一些设置项的对象/字典。
    .then(function(response) {
        if (response.ok) {
            response.blob().then( 
                //res.blob返回的是Promise的body，需要经过then方法处理后才可转换成为body。
                function(myBlob) {
                var objectURL = URL.createObjectURL(myBlob);
                myImage.src = objectURL;}
            )
        throw new Error('Network response was not ok.');};





<br>
<br>
————————————————————————

> 参考书籍：

《JavaScript语言精粹》，Douglas Crockford, 电子工业出版社

《JavaScript DOM编程艺术》

[Mozilla开发者中心-教程](https://developer.mozilla.org/en-US/docs/Learn/JavaScript)

[Mozilla开发者中心-指南](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)

[Mozilla开发者中心-参考手册](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)

> 更新日志：

2018年2月10日 完成谋智的JS入门手册中函数/事件部分学习，完成了一个简要图库的设计，更新了这部分的文档。我发现Event这东西和Qt的信号/槽机制模型非常类似，因此学习起来也不太困难。最后，我发现WebStorm和PyCharm对于语法补全和检查的功能太棒了，所以正全面转向IDE（写作此文时正在使用VS Code）。

