---
layout: post
title: Python学习笔记 - 装饰器和元类基础
categories:
  - Python
  - 读书笔记
  - 代码重用
---

> 这是我学习Python装饰器和元类的读书笔记。代码在iPython Notebook中运行，这篇文章由Notebook直接导出为md文档，我添加了一些标题和注释文字。以备日后用到装饰器和元类的时候提供一些基本概念以及常用方法、操作陷阱。

# 装饰器基础

## 1. 装饰器语法基础

装饰器提供了一种方法，在函数和类定义语句的末尾插入自动运行代码——
对于函数装饰器，在def的末尾；对于类装饰器，在class的末尾。

```python
def print123(func):
    def inner():
        print(123)
    return inner
def print456(func):
    def inner():
        print(456)
    return inner
```

> 装饰器语法和调用

需要注意顺序，按照从下往上的顺序调用。


```python
@print456 #函数先执行，之后是print123之后才是print456
@print123
def hello():
    return 2 + 2
hello()
#等价于
print456(print123(hello()))
```

    456
    456

    <function __main__.print456.<locals>.inner>



装饰器是一种代码的重用方式，其可以让函数看起来更整齐。其接受一个函数的返回值，然后将其传递给装饰器，然后在调用这个函数的时候，会自动先读取函数，之后传递给其上的装饰器，最后返回结果。

装饰器用来在函数运行前进行确认（常见的比如用户授权或者URI地址转向），或者是在函数运行之后进行输出的清理或者异常处理（比如将dict转换成为json或者返回一些错误之类）。


## 2. 函数装饰器实例

### 2.1 计数器类的函数装饰器

被类装饰时，被装饰者相当于充当装饰器类的一个实例。在__call__魔术方法中传递参数并且进行函数的保存和使用


```python
class tracer:
    def __init__(self,func):
        self.count = 0
        self._func = func
    def __call__(self,*args,**kwargs):
        #因为作为装饰器只需要返回一个callable对象即可。装饰器会调用此处的call代码。
        self.count += 1
        print("The count now is %s：%s"%(self.count,self._func.__name__))
        return self._func(*args,**kwargs)
@tracer
def add3(x,y,z): #这个例子本质上是这样的：add3变成了装饰器tracer的一个实例。
    return x+y+z
add3(1,2,3)
add3(1,2,4)
add3(6,6,6)
@tracer #对于每个被装饰的函数，都会默认调用一个全新的实例。
def muti(x,y):
    return x**y
muti(6,4)
#这种方法应用与装饰类方法有问题
```

    The count now is 1：add3
    The count now is 2：add3
    The count now is 3：add3
    The count now is 1：muti
    
    1296

### 2.2 计数器函数的函数装饰器


```python
#下面是一个很糟糕的版本，因为使用了全局的变量count。
count = 0
def tracer(func):
    def wrapper(*args,**kwargs):
        global count #这个global让count只能呆在全局变量的位置
        count += 1
        print("the count now is %s:%s"%(count,func.__name__))
        return func(*args,**kwargs)
    return wrapper
@tracer
def add4(x,y,z):
    return x+y+z
add4(1,2,3)
@tracer
def muti(x,y):
    return x**y
muti(6,4)
```

    the count now is 1:add4
    the count now is 2:muti

    1296

```python
#一个改进版本,使用了nonlocal来修改封闭的函数作用于变量，允许上层的变量使用。
#而每次被不同装饰函数调用tracer时，像创建类实例一样，count被归零。
def tracer(func):
    count = 0
    def wrapper(*args,**kwargs):
        nonlocal count
        count += 1
        print("the count now is %s:%s"%(count,func.__name__))
        return func(*args,**kwargs)
    return wrapper
@tracer
def add4(x,y,z):
    return x+y+z
add4(1,2,3)
add4(1,2,6) #相当于不使用@tracer的：tracer(add4)(1,2,6)调用，可以看到：
#每次调用相同函数时，相当于func = tracer(add4) func在这里返回了一个叫做wrapper的
#callable对象，同一函数的调用相当于只调用wrapper（1，2，6）这个对象，因此count不会归零。
@tracer
def muti(x,y):
    return x**y
muti(6,4)
```

    the count now is 1:add4
    the count now is 2:add4
    the count now is 1:muti

    1296



```python
#还有一种非常隐晦的调用方式，不过在新版本Python下好像不可用。
count = 0
def tracer(func):
    def wrapper(*args,**kwargs):
        wrapper.count += 1
        # 在python 3.6下不可用
        # 'function' object has no attribute 'count'
        print("the count now is %s:%s"%(count,func.__name__))
        return func(*args,**kwargs)
    return wrapper
@tracer
def add4(x,y,z):
    return x+y+z
#add4(1,2,3)
#add4(1,2,6)
@tracer
def muti(x,y):
    return x**y
#muti(6,4)
```

```python
def a(x,y):
    def inner():
        return x + y
    return inner()
print(a(1,2))
def b(x,y):
    z = x + y 
    def inner():
        return z
    return inner()
print(b(1,2))
def c(x,y):
    z = y - x
    def inner():
        #q = z + 1
        #z += 1 #错误 未声明本地变量
        #print(z) #可以进行
        return z
    return inner()
print(c(1,2))
```

    3
    3
    1
    

### 2.3 类方法的函数装饰器


```python
class tracer:
    def __init__(self,func):
        self.count = 0
        self._func = func
    def __call__(self,*args,**kwargs):
        #因为作为装饰器只需要返回一个callable对象即可。装饰器会调用此处的call代码。
        self.count += 1
        print("The count now is %s：%s"%(self.count,self._func.__name__))
        return self._func(*args,**kwargs)
class Person:
    def __init__(self):
        pass
    @tracer
    def sayHi(self,name):
        print("Hello %s"%name)
    @tracer
    def sayHello(self):
        print("Hello guys")
        # TypeError: sayHello() missing 1 required positional argument: 'self'
        # 这里的问题是此self不等于彼self，其实这个self指的是tracer,
        # 所以sayHello(tracer)当然出错
p = Person()
#p.sayHi("Corkine")
#p.sayHello()
```


```python
#所以我们只有通过tracer嵌套函数来解决这个问题。
def tracer(func):
    count = 0
    def wrapper(*args,**kwargs):
        nonlocal count
        count += 1
        print("the count now is %s:%s"%(count,func.__name__))
        return func(*args,**kwargs)
    return wrapper
class Person:
    def __init__(self):
        pass
    @tracer
    def sayHi(self,name):
        print("Hello %s"%name)
    @tracer
    def sayHello(self):
        print("Hello guys")
p = Person()
p.sayHi("Corkine")
p.sayHello()
p.sayHello()
```

    the count now is 1:sayHi
    Hello Corkine
    the count now is 1:sayHello
    Hello guys
    the count now is 2:sayHello
    Hello guys
    


```python
#当然，其实我们还是可以使用装饰器类的，可以通过这样的方法解决问题：
class tracer:
    def __init__(self,func): #这里的func指的是sayHi
        self.count = 0
        self._func = func
    def __call__(self,*args,**kwargs):
        self.count += 1
        print("The count now is %s：%s"%(self.count,self._func.__name__))
        return self._func(*args,**kwargs)
    def __get__(self,instance,owner): #get会在调用sayHi时自动调用，
        #get本质上应该返回找到的instance的相应attr的value。
        #instance <__main__.Person object at 0x000001C742665780>
        #owner <class '__main__.Person'>
        return wrapper(self,instance)#返回tracer和sayhi
        # 注意，需要的是instance而不是class，这里的self返回的是instance而不是tracer class
        # 所以返回的是两个instance
        # 注意：这里返回的是一个类实例，而不是 wrapper(self,instance)() 是一个callable对象
        #所有作为装饰器的东西都不允许返回call,而只是callable stuff 因为这个return并非一个return，而会调用别的东西，返回最后的一个值。这里指的是
        #tracer ins(Person instance,*args,**kwargs) 这里会继续调用tracer的call，最后返回Person.sayHi(*args,**kwargs)
class wrapper:
    def __init__(self,desc,subj):
        #desc <__main__.tracer object at 0x000001C7426C6860>
        #subj <__main__.Person object at 0x000001C7426C6898>
        self.desc = desc
        self.subj = subj
    def __call__(self,*args,**kwargs):1
        # 这里的desc指的是tracer的instance，subjj指的是person的instance。
        return self.desc(self.subj,*args,**kwargs)
        # 这里调用tracer的__call__方法，但是区别是第一个参数变成了person而不是tracer。
class Person:
    def __init__(self):
        pass
    @tracer
    def sayHi(self,name):
        print("Hello %s"%name)
    @tracer
    def sayHello(self):
        print("Hello guys")
p = Person()
p.sayHi("Corkine")
p.sayHello()
```

    The count now is 1：sayHi
    Hello Corkine
    The count now is 1：sayHello
    Hello guys
    


```python
#也可以简单的写成这个样子：
class tracer:
    def __init__(self,func):
        self.count = 0
        self._func = func
    def __call__(self,*args,**kwargs):
        self.count += 1
        print("The count now is %s：%s"%(self.count,self._func.__name__))
        return self._func(*args,**kwargs)
    def __get__(self,instance,owner):
        def wrapper(*args,**kwargs):
            return self(instance,*args,**kwargs)
        return wrapper
        #不返回wrapper()的原因是因为参数问题
        #因为本质上是这样调用的 tracer(PersonInstance.sayHi)(name)所以里面所有的return不用写()，否则参数难以处理
        #直到最终关系链条确认后才进行参数传递，可以将其看作一个准备好但是没有参数的待工作状态
        #这里是触发了tracer instance的call方法，然后将self复写为Person instance，最后返回 Persion.sayHi(待传递参数占位符),当调用()时会自动
        #写入参数返回一个最终值。wrapper为一个并未隐藏内部接口，但是隐藏了外部调用接口的完整weapper()
        #也就是说虽然没有一个外部接口，但是weapper的内部接口还是会继续工作和连接。等到使用的时候直接打开就好。
        #有无外部接口的唯一区别就是能否调用内部call方法以及立马返回return语句。如果没有外部接口，则不会当即调用call（在此例子中self()调用
        #了call方法而不是weapper），return会列好算式，但是不会马上计算。
class Person:
    def __init__(self):
        pass
    @tracer
    def sayHi(self,name):
        print("Hello %s"%name)
    @tracer
    def sayHello(self):
        print("Hello guys")
p = Person()
p.sayHi("Corkine")
p.sayHi("Corkine")
p.sayHello()
```

    The count now is 1：sayHi
    Hello Corkine
    The count now is 2：sayHi
    Hello Corkine
    The count now is 1：sayHello
    Hello guys
    

`object.__get__(self, instance, owner)`

Called to get the attribute of the owner class (class attribute access) or of an instance of that class (instance attribute access). 

owner is always the owner class, while instance is the instance that the attribute was accessed through, or None when the attribute is accessed through the owner. 

This method should return the (computed) attribute value or raise an AttributeError exception.

### 2.4 装饰器的参数保存


```python
import time
class timer:
    def __init__(self,func,trace=True,label=""):
        self.func = func
        self.alltime = 0
        self.trace = trace
        self.label = label
    def __call__(self,trace=True,label="",*args,**kwargs):
        now = time.clock()
        r = self.func(*args,**kwargs)
        end = time.clock()
        t = end - now
        if self.trace:
            print("【%s】Time for %s is %.5f"%(self.label,self.func.__name__,t))
        return r
#错误，错误的原因是无法区分开装饰器参数和被装饰的函数或者类方法。所以推荐使用嵌套函数，如果非要使用类的话，使用函数创造一个作用域
#来放置参数。
```


```python
import time
def timer(trace=True,label=""): #同理，这里也不需要写func传递进去。因为func这个参数的传递是timer()(func)在这个位置
    #如果在@时使用了timer(trace,label)这种形式的话。反之，如果@trace不加参数，则直接调用timer(func)，这个时候就需要传递func参数
    class Timer:
        def __init__(self,func):
            self.func = func
            self.alltime = 0
            self.trace = trace
            self.label = label
        def __call__(self,*args,**kwargs):
            now = time.clock()
            r = self.func(*args,**kwargs)
            end = time.clock()
            t = end - now
            if trace:
                print("%s Time for %s is %.5f"%(label,self.func.__name__,t))
            return r
    return Timer #注意这里默认不需要写()传递func进去。
#因为本质上是这样调用的：timer(trace,label)(func)(n=10000)
@timer(trace=True,label="CCC==>")
def test(n):
    return [x**20 for x in range(n)]
test(100000)
test(2000000)
@timer(trace=False,label="CCC==>")
def test2(n):
    return [x**20 for x in range(n)]
test2(1000)
print("This is END")
```

    CCC==> Time for test is 0.06214
    CCC==> Time for test is 1.00973
    This is END
    

所以说，最佳实践证明，最好装饰器外面套上一层函数作为参数保存的位置。在里面可以放置之前的嵌套函数装饰器或者类装饰器。对于后者需要注意类装饰器应用于类方法的问题，但是其更灵活，相比较嵌套函数装饰器的话，因为后者总是要声明nolocal。

## 3. 类装饰器实例

类装饰器有两个作用，其一为管理所有的类实例，其二为管理实例的所有可能接口。

### 3.1 管理单体类：直接的类装饰器

一个管理单体类（类实例级别）的例子。这和元类的功能重合，所以不算类装饰器的一个显著工作领域。

区别于函数装饰器或者下方的包裹在函数装饰器内的类装饰器，传入对象是类级别的而非是实例级别的。因此可以看到最后被覆盖的结果。


```python
class singleton:
    def __init__(self, aClass): # On @ decoration
        self.aClass = aClass
        self.instance = None
    def __call__(self, *args): # On instance creation
        if self.instance == None:
            self.instance = self.aClass(*args) # One instance per class
        return self.instance
@singleton
class Person: # Rebinds Person to onCall
    def __init__(self, name, hours, rate): # onCall remembers Person
        self.name = name
        self.hours = hours
        self.rate = rate
    def pay(self):
        return self.hours * self.rate
cm = Person("Corkine",23,12)
print(cm.pay())
mv = Person("Corkine2",230,32)
print(mv.pay())
#如果当前有实例的话，则直接调用此实例而不是新建一个实例。所以结果相同。

```

    276
    276
    

装饰器可以用来管理类本身，也可以用来管理类的实例。这两者的最重要区别在于前者不需要参数传入而后者则需要实例初始化的参数，因此需要多一个包装器来接受类，然后返回一个类以接受实例的参数。由于实例创建时会自动调用这个类的init，所以在以后会自动记住这个实例，因此可以用来保持状态。

而管理类本身，则不需要包装器，只需要将类传入装饰器即可。在调用时的参数通过call方法传递进去。

类装饰器用来管理类的实例和类本身都很拿手。而元类则主要用来管理类本身。此外，需要注意，区别于之前在函数装饰器讲到的主体问题，这里的装饰器作用对象都是整个类。而那些作用于类方法的装饰器，我们把它分到函数装饰器部分。

### 3.2 管理类接口：通过setattr和包装器

一个管理实例接口的例子（通过调用getattr）

这是一个假的函数装饰器，可以称其为带有包装器的类装饰器。


```python
def tracer(aclass): #虽然这里传递的是class，但是在tracer内self.func已经通过init成为instance。而每次对于类实例的装饰，都会直接调用Tracer的对应
    #属于该被装饰类实例的tracer实例，所以其信息就得到了保存。
    print(aclass)
    class Tracer:
        def __init__(self,*args):
            self.func = aclass(*args) #类实例而非类本身
            print(self.func)
            self.count = 0
        def __call__(self,*args):
            return self.func(*args)
        def __getattr__(self,attrname):
            self.count += 1
            print("Tracer %s:%d"%(attrname,self.count))
            return getattr(self.func,attrname)
    return Tracer
@tracer
class Person: # Rebinds Person to onCall
    def __init__(self, name, hours, rate): # onCall remembers Person
        self.name = name
        self.hours = hours
        self.rate = rate
    def pay(self):
        return self.hours * self.rate
cm = Person("Corkine",23,12)
print(cm.pay())
cm.name
cm.name
cm.name
cm.pay()
mv = Person("Corkine2",230,32)
print(mv.pay())
cm.pay()
```

    <class '__main__.Person'>
    <__main__.Person object at 0x000001FCF2D39F28>
    Tracer pay:1
    276
    Tracer name:2
    Tracer name:3
    Tracer name:4
    Tracer pay:5
    <__main__.Person object at 0x000001FCF2D39FD0>
    Tracer pay:1
    7360
    Tracer pay:6
    
    276



如果类装饰器没有包装器包装的话，那么保留的信息会被复写。因为被传递的是一个类而非类的实例。因此只能实现对于类级别的更改。

这样其实很容易理解，因为Tracer被当作包装器的时候，如果要调用pay方法，其经历了：
Tracer(Person_class).--call--("Corkine",23,12).--getattr--(pay)，因此其必然不能传递参数到init方法中，所以就没办法保存已经构造好的Person instance。

而对于包装器而言，其默认如下：
Tracer(Person_class)("Corkine",23,12).--getattr--(pay)。因为包装器封装了类这个参数，因此可以在返回的内部装饰器中直接用这个类加上传递的参数直接构造instance，这样的话就绑定到类实例级别，也就能够保存所有实例的信息了。


```python
class Tracer:
    def __init__(self,aClass):
        self.aClass = aClass
        print(aClass)
        self.count = 0
    def __call__(self,*args):
        self.wrapper = self.aClass(*args)
        return self
    def __getattr__(self,attrname):
        self.count += 1
        print("Tracer %s:%d"%(attrname,self.count))
        return getattr(self.wrapper,attrname)
@Tracer
class Person: # Rebinds Person to onCall
    def __init__(self, name, hours, rate): # onCall remembers Person
        self.name = name
        self.hours = hours
        self.rate = rate
    def pay(self):
        return self.hours * self.rate
cm = Person("Corkine",23,12)
print(cm)
print(cm.pay())
cm.name
cm.name
cm.name
cm.pay()
mv = Person("Corkine2",230,32)
print(mv.pay())
cm.pay()
```

    <class '__main__.Person'>
    <__main__.Tracer object at 0x000001FC80F3F668>
    Tracer pay:1
    276
    Tracer name:2
    Tracer name:3
    Tracer name:4
    Tracer pay:5
    Tracer pay:6
    7360
    Tracer pay:7
    
    7360



> 使用装饰器的优缺点分析

缺点：类型被修改了，如上可以看到所有的类信息都被替换成Tracer的信息。
存在额外的调用，尤其是对于含有包装器的类装饰器来说。但是，其实你不使用装饰器，一样存在额外的调用。因此这一条不太成立。

优点：就像__init__变成一种编程的格式而不是用.init()来进行初始化一样，装饰器提供了一种清晰的语法，方便代码复用以及代码的封装和一致性。

### 3.3 装饰器用以扩展的使用方法

比如注册一个函数或者类的新方法和变量之类


```python
traceMe = False
def trace(*args):
    if traceMe: 
        print('[' + ' '.join(map(str, args)) + ']')
def Private(*privates): # 用于变量保存的外层函数
    def onDecorator(aClass): # 包装器，用以保存类
        class onInstance: # 真正的类装饰器，装饰实例
            def __init__(self, *args, **kargs):
                self.wrapped = aClass(*args, **kargs) #将类转化成实例
            def __getattr__(self, attr):
                trace('get:', attr)
                if attr in privates: #如果调用的attr为我们指定的那些，就报错而不调用
                    raise TypeError('private attribute fetch: ' + attr)
                else:
                    return getattr(self.wrapped, attr) #否则正常调用实例的attr并且返回结构
            def __setattr__(self, attr, value): #
                trace('set:', attr, value)
                if attr == 'wrapped': # Allow my attrs
                    self.__dict__[attr] = value # 因为和setattr(self.weapped)冲突，所以换用这种方法设置
                elif attr in privates: #不允许设置制定的attr
                    raise TypeError('private attribute change: ' + attr)
                else:
                    setattr(self.wrapped, attr, value) 
                    #如果是其它的attr，则正常设置 也可以直接用slef.wrapped.__dict__[attr]=value进行设置
        return onInstance
    return onDecorator

#if __name__ == '__main__':
traceMe = True
@Private('data', 'size') # Doubler = Private(...)(Doubler)
class Doubler:
    def __init__(self, label, start):
        self.label = label # Accesses inside the subject class
        self.data = start # Not intercepted: run normally
    def size(self):
        return len(self.data) # Methods run with no checking
    def double(self): # Because privacy not inherited
        for i in range(self.size()):
            self.data[i] = self.data[i] * 2
    def display(self):
        print('%s => %s' % (self.label, self.data))
        
X = Doubler('X is', [1, 2, 3])
Y = Doubler('Y is', [-10, -20, -30])
print(X.label) # Accesses outside subject class
X.display(); X.double(); X.display() # Intercepted: validated, delegated
print(Y.label)
Y.display(); Y.double()
Y.label = 'Spam'
Y.display()


#Y.data = [1,2,3] 不允许运行，因为是一个私有方法。
```

    [set: wrapped <__main__.Doubler object at 0x000001FCF9A85898>]
    [set: wrapped <__main__.Doubler object at 0x000001FCF9A85860>]
    [get: label]
    X is
    [get: display]
    X is => [1, 2, 3]
    [get: double]
    [get: display]
    X is => [2, 4, 6]
    [get: label]
    Y is
    [get: display]
    Y is => [-10, -20, -30]
    [get: double]
    [set: label Spam]
    [get: display]
    Spam => [-20, -40, -60]
    

```python
traceMe = False
def trace(*args):
    if traceMe: print('[' + ' '.join(map(str, args)) + ']')
def accessControl(failIf):
    def onDecorator(aClass):
        if not __debug__:
            return aClass
        else:
            class onInstance:
                def __init__(self, *args, **kargs):
                    self.__wrapped = aClass(*args, **kargs)
                def __getattr__(self, attr):
                    trace('get:', attr)
                    if failIf(attr):
                        raise TypeError('private attribute fetch: ' + attr)
                    else:
                        return getattr(self.__wrapped, attr)
                def __setattr__(self, attr, value):
                    trace('set:', attr, value)
                    if attr == '_onInstance__wrapped': #????????????????为什么是这种形式？前一个短线，加上一个类，加上两个短线，加上attr？
                        self.__dict__[attr] = value
                    elif failIf(attr):
                        raise TypeError('private attribute change: ' + attr)
                    else:
                        setattr(self.__wrapped, attr, value)
                def __str__(self):
                    return str(self.__wrapped)
                def __add__(self, other):
                    return self.__wrapped + other
                def __getitem__(self, index):
                    return self.__wrapped[index] # If needed
                def __call__(self, *args, **kargs):
                    return self.__wrapped(*arg, *kargs) # If needed
            return onInstance
    return onDecorator
def Private(*attributes):
    return accessControl(failIf=(lambda attr: attr in attributes))
def Public(*attributes):
    return accessControl(failIf=(lambda attr: attr not in attributes))

traceMe = True
@Public('data', 'size',"label","display","double")
class Doubler:
    def __init__(self, label, start):
        self.label = label # Accesses inside the subject class
        self.data = start # Not intercepted: run normally
    def size(self):
        return len(self.data) # Methods run with no checking
    def double(self): # Because privacy not inherited
        for i in range(self.size()):
            self.data[i] = self.data[i] * 2
    def display(self):
        print('%s => %s' % (self.label, self.data))
        
X = Doubler('X is', [1, 2, 3])
Y = Doubler('Y is', [-10, -20, -30])
print(X.label) # Accesses outside subject class
X.display(); X.double(); X.display() # Intercepted: validated, delegated
print(Y.label)
Y.display(); Y.double()
Y.label = 'Spam'
Y.display()

@Private('age')
class Person:
    def __init__(self):
        self.age = 42
    def __str__(self):
        return 'Person: ' + str(self.age)
    def __add__(self, yrs):
        self.age += yrs
X = Person() # Name validations still work
print(X)
print(X + 10) #TypeError: unsupported operand type(s) for +: 'onInstance' and 'int'
```

    [set: _onInstance__wrapped <__main__.Doubler object at 0x000001FC8004D9B0>]
    [set: _onInstance__wrapped <__main__.Doubler object at 0x000001FC8007D8D0>]
    [get: label]
    X is
    [get: display]
    X is => [1, 2, 3]
    [get: double]
    [get: display]
    X is => [2, 4, 6]
    [get: label]
    Y is
    [get: display]
    Y is => [-10, -20, -30]
    [get: double]
    [set: label Spam]
    [get: display]
    Spam => [-20, -40, -60]
    [set: _onInstance__wrapped Person: 42]
    Person: 42
    None
    

注意，在--setattr--中，我们也必须使用完整扩展的名称字符串
('_onInstance__wrapped')，因为这是Python对其的修改。

即便使用self.__wrapped也不行。因为完整扩展的名称字符串应该如上。之前在私有实现的时候，不用考虑这个问题，而现在对于公有，则必须考虑了。

对于py3之后，所有类都是新式类，所以，对于运算符重载等，不经过getattr，所以必须手动加以实现。

### 3.4 函数注解和参数限制


```python
def rangetest(func):
    def onCall(*pargs, **kargs):
        argchecks = func.__annotations__
        code = func.__code__
        vlist = code.co_varnames[:code.co_argcount]
        print(argchecks)
        for check in argchecks:
            if isinstance(argchecks[check],tuple) and len(argchecks[check]) == 2:
                tmin = argchecks[check][0]
                tmax = argchecks[check][1]
                print(tmin,tmax)
                for x in enumerate(vlist):
                    if check == x[1]:
                        index = x[0]
                    print("x0:",index)
                for x in enumerate(list(pargs)):
                    if x[0] == index:
                        val = x[1]
                    else:pass
                if check in kargs:
                    val = kargs[check]
                print("test",check,val)
                if int(val) <= tmax and int(val) >= tmin:
                    pass
                else:
                    raise ValueError("Not Range")
        return func(*pargs, **kargs)
    return onCall
@rangetest
def func(a:(1, 5), b, c:(0.0, 1.0)): # func = rangetest(func)
    print(a + b + c)
func(1, 2, c=1)
#func(0, 2, c=1)#错误，超出范围
```

    {'a': (1, 5), 'c': (0.0, 1.0)}
    1 5
    x0: 0
    x0: 0
    x0: 0
    test a 1
    0.0 1.0
    x0: 0
    x0: 0
    x0: 2
    test c 1
    4
    

# 元类基础

和装饰器类似，元类也可以完成对于类的管理以及对于类的实例的管理。其二者互补，其中在对于类的管理上，彼此能力差别不大，在对于类的实例管理上，元类稍逊一筹。

## 1. 元类、类继承和管理器函数

和类装饰器不同的地方在于，元类总是在类创建的时候起作用，因此其不会像类装饰器一样更改代码的TrackBack位置。其有利于提供一种简介的语法模式，同时提供了灵活定义类的能力。

对于类而言，通常我们希望一个方法重用，最笨的方法是给每个子类写一个方法。如果考虑到代码重用问题，那么就使用一个叫做管理器的函数，将这个函数放置在任何需要添加方法的地方。

稍微高明一点的做法是将这些方法抽象化，然后创建一个超类，这些子类都从超类中继承这个方法。这样的话，方便代码的统一管理。但是这种方法有一个弊端，那就是，我们只能够静态的创建一个方法到超类，否则将要面临多个现有类的重构。因此，动态添加方法的方法就是使用元类。

管理器实例：


```python
'''def extra(self, arg): ...
    def extras(Class): # Manager function: too manual
    if required():
        Class.extra = extra
class Client1: ...
    extras(Client1)
class Client2: ...
    extras(Client2)'''
```


> 元类实例


```python
def extra(self,value):pass
class Extra(type):
    def __init__(Class,classname,superclass,attrdict):
        Class.extra = extra
class Person(metaclass=Extra):
    def __init__(self):
        pass
a = Person();a.extra()
```

## 2. 元类模型

在Python3中：

type是一个类，所有用户定义的类对象名（而非类实例）是type类的对象的实例（而非子类）。

在Python2.X中：

type是一个类，object是type类的子类（而不是实例），所有新式类/用户定义的类的对象名都是object类对象的实例。传统类则是type类对象的实例。


```python
#type()函数用来返回其构造的类
print(type(list),type(type),type([])) #list是type的实例，[]是list的实例。
class A:
    def __init__(self):
        pass
a = A()
print(type(A),type(a)) #用户定义类是type的实例，a是A的实例。
```

    <class 'type'> <class 'type'> <class 'list'>
    <class 'type'> <class '__main__.A'>
    

虽然元类和类装饰器在很大程度上功能交叉，但是这一点是其根本区别，也就是说，元类只对类起作用，因为类是元类的实例，但是对这个类的子类不起作用。因此，其只在类创建的时候有用，而不是想装饰器一样随时有用，这也限制了元类管理类实例的能力（当然，你可以在元类中手动调用type()来构造一个装饰器，不过那样很麻烦）

### 2.1 元类和type类的关系

元类是type类的一个子类，通过使用元类，所有的其他类都变成了元类的实例而不是type类的实例。这样的话，就可以在构造类的时候进行一些操作。

### 2.2 Class语句的本质

class语句相当于这样 class = type(classname,superclass,attrdict)

其调用类type类的type.--new--(typeclass,classname,superclass,attrdict)以及
type.--init--(class,classname,superclass,attrdict)。new方法创建了一个新的class对象，而init方法则初始化了新创建的对象。重新定义init和new方法是使用超类时最常用的两种方法。

比如：


```python
class A(B):
    self.data = 1
    def go(self)：
     pass
A = type("A",(B,),{"data":1,"go":go,"--model--":"--main--"})
```

### 2.3 元类的构造

采用 `class Spam(Eggs, metaclass=Meta):` 这种方式，其中超类放在前方的args中，而在kwargs中指定metaclass的名称即可。

构造元类类似于： `Meta("A",(B,),{"data":1,"go":go,"--model--":"--main--"})` 

这句话会调用type类的call魔术方法，type类的call把创建和初始化新类的任务交给了元类，如果元类有自己的init和new方法的话。

new方法接受四个参数，分别是元类、类名称、超类元祖、类的值。init方法接受四个参数，分别是通过new方法构造好的类，这个类的名称，超类元组，类的值。


```python
class MetaOne(type):
    def __new__(meta, classname, supers, classdict):
        print('In MetaOne.new:', meta, classname, supers, classdict, sep='\n...')
        return type.__new__(meta, classname, supers, classdict)
    def __init__(Class, classname, supers, classdict):
        print('In MetaOne init:', Class, classname, supers, classdict, sep='\n...')
        print('...init class object:', list(Class.__dict__.keys()))
class A(metaclass=MetaOne):
    def __init__(self):
        pass
    def go(self):
        pass
a = A()
```

    In MetaOne.new:
    ...<class '__main__.MetaOne'>
    ...A
    ...()
    ...{'__module__': '__main__', '__qualname__': 'A', '__init__':
     <function A.__init__ at 0x000001F6A2ECC840>, 'go': <function A.go at 0x000001F6A2ECC950>}
    In MetaOne init:
    ...<class '__main__.A'>
    ...A
    ...()
    ...{'__module__': '__main__', '__qualname__': 'A', '__init__': 
    <function A.__init__ at 0x000001F6A2ECC840>, 'go': <function A.go at 0x000001F6A2ECC950>}
    ...init class object: ['__module__', '__init__', 'go', '__dict__', '__weakref__', '__doc__']
    

这里很好玩，因为如果我们不重定义new方法，而只有init方法的话，默认会使用type的new来创建一个新类，而和类装饰器相比，这里的init方法的第一个参数竟然是原始的类而不是MetaOne。因为type()会触发call，然后触发new，返回一个类，之后触发init来初始化类的一些参数。对于一般的类或者类装饰器而言，返回的第一个参数都是其类本身。对于元类来说，第一个参数是其实例类本身，对于类来说，第一个参数是其自身，对于类实例来说，第一个参数是实例本身。

## 3. 元类可能不是“类”

元类可以是一个工厂函数，比如下面这个：


```python
def MetaTwo(classname,superclass,attrvalue):
    print("I am in MetaTwo",classname,superclass,attrvalue,sep="\n   ")
    return type(classname,superclass,attrvalue)
class A(metaclass=MetaTwo):
    def go(self):
        pass
a = A()
```

    I am in MetaTwo
       A
       ()
       {'__module__': '__main__', '__qualname__': 'A', 
       'go': <function A.go at 0x000001F6A2ECC048>}
    


```python
class MetaFather(type):
    def __call__(meta,classname,superclass,attrvalue):
        print("I am in MetaFather:",classname,superclass,attrvalue,sep="\n   ")
        return type.__call__(meta,classname,superclass,attrvalue)
class MetaSon(type,metaclass=MetaFather):
    def __init__(Class,classname,superclass,attrvalue):
        print("I am in MetaSon: init",Class,classname,superclass,attrvalue,sep="\n   ")
    def __new__(Meta,classname,superclass,attrvalue):
        print("I am in MetaSon: new",Meta,classname,superclass,attrvalue,sep="\n   ")
        return type.__new__(Meta,classname,superclass,attrvalue)
class A(metaclass=MetaSon):
    def go(self):
        pass
a = A()
```

    I am in MetaFather:
       A
       ()
       {'__module__': '__main__', '__qualname__': 'A', 
       'go': <function A.go at 0x000001F6A2E8DD08>}
    I am in MetaSon: new
       <class '__main__.MetaSon'>
       A
       ()
       {'__module__': '__main__', '__qualname__': 'A',
       'go': <function A.go at 0x000001F6A2E8DD08>}
    I am in MetaSon: init
       <class '__main__.A'>
       A
       ()
       {'__module__': '__main__', '__qualname__': 'A', 
       'go': <function A.go at 0x000001F6A2E8DD08>}
    

需要注意，虽然type()这样call只需要三个参数，但是如果使用type.--call--则必须使用四个参数，像--new--一样，包含元类、类名、超类和类方法。

## 4. 向元类中添加方法

下面一个例子可以使用装饰器同样完成，因为其添加的方法为静态方法。因为函数总是在被分配给类的时候自动添加了第一个参数self，所以我们可以做到函数独立于主体进行定义。


```python
def eggsfunc(obj):
    return obj.value * 4
def hamfunc(obj, value):
    return value + 'ham'
class Extender(type):
    def __new__(meta, classname, supers, classdict):
        classdict['eggs'] = eggsfunc
        classdict['ham'] = hamfunc
        return type.__new__(meta, classname, supers, classdict)
class Client1(metaclass=Extender):
    def __init__(self, value):
        self.value = value
    def spam(self):
        return self.value * 2
class Client2(metaclass=Extender):
    value = 'ni?'
X = Client1('Ni!')
print(X.spam())
print(X.eggs())
print(X.ham('bacon'))
```

    Ni!Ni!
    Ni!Ni!Ni!Ni!
    baconham
    

我们也可以将元类封装成为更加动态的句子，这样在进行判断后会自动赋予类更多的扩展能力。


```python
class MetaExtend(type):
    def __new__(meta, classname, supers, classdict):
        if sometest():
            classdict['eggs'] = eggsfunc1
        else:
            classdict['eggs'] = eggsfunc2
        if someothertest():
            classdict['ham'] = hamfunc
        else:
            classdict['ham'] = lambda *args: 'Not supported'
        return type.__new__(meta, classname, supers, classdict)
```
