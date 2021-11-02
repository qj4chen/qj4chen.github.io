---
layout: post
title: Python数据处理学习笔记 - Series & Dataframe篇
categories:
  - Python数据处理
  - 读书笔记
  - pandas
---

> 这是我阅读《用Python进行数据分析》一书的笔记、实验和总结。本篇文章主要讲解pandas包 Series 和 DataFrame 这两大类型的选取、索引、排序、运算、映射和统计基础，以及值处理。

第一篇文章主要介绍 numpy 的 ndarray 这一多维数组对象，这是 Python 进行数据分析的基石，也是保证大量数据处理速度的必要条件。这一篇文章主要讲解 DataFrame 和 Series 这两个对象本身的使用方法。

Series 和 DataFrame 属于 2d-array（配合层次化索引可以起到 nd-array 的效果），基于 numpy 的 nd-array 对象。这两大结构都由值和索引构成，对于索引的切片以选择值的方法（基本切片符、loc/iloc切片符）精妙的扩宽了 Python 原生切片只能处理 1d-array 的限制。

本文将首先介绍 Series 和 DataFrame **基本的结构和创建、修改**，然后介绍**基本和loc系列的切片选择语法**是如何选取一个或一组或多个或多组数据的，之后介绍了对于**行列索引的增删改排**的操作，比如重建索引 reindex、映射索引 rename_index、提升索引 set_index 和删除索引 reset_index，索引和值的排序 sort_index, sort_value，之后介绍了对于 **S/DF 的值的按行、列预处理和映射**，比如 apply、map、mapapply，在最后介绍了**基本统计和结构、传播计算方法**，比如 count、sum、std、describe、`*`、`+` 等。

对于不同对象的 concat、merge、stack 以及数据导入的值预处理和不同导入格式（CSV、JSON、HDF5、Pickle、Excel）,则会在下一篇文章中介绍。而 groupby、aggregate、透视、交叉等聚合技术将会在之后逐步介绍。

约定俗成的，需要引入以下包，pandas中较为重要的两个数据结构Series和DataFrame单独引入使用更加方便。

```python
import pandas as pd
from pandas import Series, DataFrame
import numpy as np
import matplotlib as plt
```

## 1. pandas 数据结构基础

### 1.1 Series

Series 本质是一个**含有索引的一维数组**，看起来，其包含一个左侧可以自动生成（也可以手动指定）的index 和右侧的 values 值，分别使用 `s.index` ，`s.values` 进行查看。index 返回一个 index 对象，而 values 则返回一个 array。

> 为什么使用 Series 而不是字典？

Series 就是一个带有索引的列表，为什么我们不使用字典呢? 一个优势是，Series 更快，其内部是向量化运行的，和迭代相比，使用 Series 可以获得显著的性能上的优势，你可以使用 `%%timeit -n 次数` 来试验一下。

> Series的两种创建方式

生成 Series 的第一种方式（较少使用）：可以使用 list(iterable) 自动生成 Series，此种情况下，**当提供索引时，index 必须和 value 严格对应，或者不提供索引**。需要注意，如果使用 ndarray 生成 Series 则必须为 1d（维）。需要注意，这种方式需要传递一个索引进去，否则将会生成 RangeIndex。


```python
obj = pd.Series(np.array([4, 7, -5, 3])) 
# 需要注意，index必须和数据长度严格匹配
print(obj);print(obj.index,obj.values)
# index值的类型为 rangeIndex，如果手动创建的话则不同
```

    0    4
    1    7
    2   -5
    3    3
    dtype: int32
    RangeIndex(start=0, stop=4, step=1) [ 4  7 -5  3]

```python
Series([1,2,3,4],index=["a","b","c","d"])

a    1
b    2
c    3
d    4
dtype: int64
```


生成 Series 的第二种方式比较常用，你可以使用字典和列表来充当 value 和 index 生成 series。这种情况下，程序会**自动将字典的 key 作为 index name**，当然，你也可以手动指定一个，**如果字典中没有此字段，则会标记值为 NULL**。

```python
n = Series({"a":2,"b":3}) # 自动 index

a    2
b    3
dtype: int64

p = Series({"a":2,"b":3}, index=["a","c","b"]) # 手动宽松 index

a    2.0
c    NaN
b    3.0
dtype: float64
```

Series 还可以像词典一样，可以使用 in 进行判断，但是需要注意的是，其只能判断index 值。

```python
print("a" in p);  # True
print(2 in p) # False
```

> 缺失值统计和处理

```python
p.isnull()
#非null值可以使用 p.notnull()

a    False
c     True
b    False
dtype: bool

p[p.isnull()] # 筛选空值

a    2.0
b    3.0
dtype: float64

p[-p.isnull()] # 筛选非空值（numpy布尔矩阵切片语法）

c   NaN
dtype: float64
```

> 利用索引进行元素选取/筛选

Series 可以根据单个索引、花式索引读取数据并对数据进行修改后写回，也可以直接使用表达式生成 bool 型索引来区分数据。

----> 传入 index 或者 index list 切片选取

```python
obj2 = pd.Series([2,34,12,3],index=["a","c","d","b"]);print(obj2)
print(obj2['a']) # 可以根据 index 读取数据 2
obj2['d'] = 6 # 可以为读取的数据赋值，会自动写回去
obj2[['c', 'a', 'd']] # 可以使用花式索引根据 index 读取数据
```

    a     2
    c    34
    d    12
    b     3
    dtype: int64

----> 传入 numpy Bool Array 切片选取（根据 index 对应筛选）

```python
arr_t = np.arange(10).reshape((2,5))
arr_t[arr_t > 5]
#类似于np，使用bool型索引来筛选数据 类似于：

c    34
d     6
dtype: int64
```

> 自动传播的标量计算

```python
arr_t * 2 # 运算广播到每一个数据并进行单独运算
np.exp(arr_t)

a    7.389056e+00
c    5.834617e+14
d    4.034288e+02
b    2.008554e+01
dtype: float64
```

> 自动对齐的结构计算

```python
print(obj_t)
myvalue = {"hebei":10000,"beijing":23223}
myindex = ["hebei","hunan","beijing","wuhan"] # 此种情况下series会自动处理缺失值
obj_t2 = Series(myvalue,index=myindex);print(obj_t2)
print(obj_t+obj_t2) # 由于nan不为数字，因此计算后的值均为nan，如henan中所示。
```

    hebei      30000.0
    hunan      12022.0
    beijing    23223.0
    wuhan          NaN
    dtype: float64
    
    hebei      10000.0
    hunan          NaN
    beijing    23223.0
    wuhan          NaN
    dtype: float64
    
    hebei      40000.0
    hunan          NaN
    beijing    46446.0
    wuhan          NaN
    dtype: float64


> 更改名称、重建索引和 Series 拼合

Series 可以进行 value 和 index 的 name 的声明，也可以对 index 进行替换。

```python
obj_t.name = "人口分布表"; # 设置表的名称
obj_t.index.name = "城市"; # 设置 index 的名称
```

    城市
    hebei      30000.0
    hunan      12022.0
    beijing    23223.0
    wuhan          NaN
    Name: 人口分布表, dtype: float64

```python
obj_t.index = ["河北","湖南","北京","武汉"]; # 重建索引，类似于 DF 的 reindex
```

    河北    30000.0
    湖南    12022.0
    北京    23223.0
    武汉        NaN
    Name: 人口分布表, dtype: float64


> 注意：使用 append 合并两个 Series 结构时，区别于Python列表，返回了一个新的Series而不是就地修改。

    a = pd.Series([1,2,3])
    b = pd.Series([4,5])
    c = a.append(b)


### 1.2 DataFrame

区别于 Series 一维的表结构，DataFrame 是一个高维的表结构。其含有两个 index，分别为 column index 和 row index，可以在构造时直接指定索引标签名。

> DataFrame构造

- 用等长列表作为值的字典构造（key 作为 Column，value 作为 Row - Value）

- 用字典包裹字典构造（类似于二级嵌套的JSON）

- 用字典包裹Series构造

- 用2d-array构造frame

一般可以使用 columns 属性指定列顺序，index 属性指定行 index, 调用 columns 方法可以返回列的 index,调用 index 方法可以返回行的 index。

如果不提供构造 DataFrame 时的 columns，默认使用字典的 Key，如果不提供 index，则自动使用 RangeIndex。注意，columns 可能按照 A-Z 进行排序（字典的 tuple 没有顺序）

```python
DataFrame({"name":[1,2,3],"age":[3,4,5]})

    age	    name
0	3	1
1	4	2
2	5	3
```

指定 columns 可以避免列顺序问题：

```python
data = {'name':["张三","李四","王五","马六"],
       "student_id":[1,2,3,4],
       "scores":[30,42,110,150]}
frame = pd.DataFrame(data,columns=["name","student_id","scores"],
index=["A","B","C","D"]);
print(frame)
```

      name  student_id  scores
    A   张三           1      30
    B   李四           2      42
    C   王五           3     110
    D   马六           4     150

和Series的 value、index 属性来描述 Series 一样，DataFrame 可以使用 head/nail 来返回头尾几行数据，使用 index 和 columns 返回 row 索引和 column 索引。

```python
print(frame.head(5)) #返回头几列数据，一般为５列，如果不够则完全返回
print(frame.columns)
print(frame.index)
```

      name  student_id  scores
    A   张三           1      30
    B   李四           2      42
    C   王五           3     110
    D   马六           4     150
    Index(['name', 'student_id', 'scores'], dtype='object')
    Index(['A', 'B', 'C', 'D'], dtype='object')

需要注意，对于使用这种方式生成的 DataFrame，其 index 必须和 Map 的 Value 的列表的长度等长，否则报错，这和 Series 是一致的：

```python
a = Series([1,2,3,4],index=["a","b","c","d","e"])
# 运行错误
```

字典嵌套列表适合时间序列数据，在这种情况下列表是绝对齐的，而嵌套字典适用于不像数据库那种某一个维度有大量条目（比如用户），而是只有少数条目（比如季度盈利），这种情况可能 Row 不齐，因此，不统一制定 Index 会更方面些。除了词典嵌套列表、词典，此外，字典嵌套 Series 也可以，Series 会被看作成一列，Series 的 index 会自动作为 DataFrame 的 index，字典的 key 被看作是 column。

```python
data_t = {"marvin":{"s4":300,"s3":123},"lili":{"s1":123,"s2":112,"s3":998}}
DataFrame(data_t)

    lili    marvin
s1	123.0	NaN
s2	112.0	NaN
s3	998.0	123.0
s4	NaN	300.0
```

需要注意，使用嵌套字典生成的 DataFrame，即便指定了 index 字段，其 index 不齐、不使用 Series、Series Index 不齐也不会报错，缺失值会被记录为 NaN。

```python
# 上面的例子等同于，Series index 可以不齐
data_t2 = {"marvin":Series({"s4":300,"s3":123}),"lili":Series({"s1":123,"s2":112,"s3":998})}
DataFrame(data_t2)

    lili    marvin
s1	123.0	NaN
s2	112.0	NaN
s3	998.0	123.0
s4	NaN	300.0

# 此外可以使用宽松 index 字段索引
data_t = {"marvin":{"s4":300,"s3":123},"lili":{"s1":123,"s2":112,"s3":998}}
f_t = DataFrame(data_t,index=["s1","s2","s3","s4","s5"])
f_t.head(3);print(f_t.T)

            s1     s2     s3     s4  s5
lili    123.0  112.0  998.0    NaN NaN
marvin    NaN    NaN  123.0  300.0 NaN
```

和 Series 类似，可以指定 index Name、table Name，此外，还可以记录 columns Name。

```python
f_t.name = "季度盈利表"; f_t.columns.name = "产品名";f_t.index.name = "季度";
print(f_t.values);f_t
```

    [[123.  nan]
     [112.  nan]
     [998. 123.]
     [ nan 300.]
     [ nan  nan]]


    产品名	lili	marvin
    季度		
    s1	123.0	NaN
    s2	112.0	NaN
    s3	998.0	123.0
    s4	NaN	300.0
    s5	NaN	NaN

> 基于 Column 的 DataFrame 元素操作

需要注意的是，索引返回的是 DataFrame 的视图而非副本，使用 copy 命令创造一个副本。

区别于 Series，由于 DataFrame 含有两个索引轴，因此稍微复杂一些。本文将这两个轴分别称之为 row index（官方称为index），column index（官方称为column）。索引默认是 column index，DataFrame提取默认为 column index，可以提取多列，提取列之后可以提取行，但是不能直接提取行(使用ix/loc/iloc方法提取行index)。

这里需要注意的是，可以定义两种操作 DF 的方法，其一是基于 `df[C][R]` 这种写法，先提取 Columns 再提取 Row。其二是基于 `df.loc[R,C], df.iloc[Ri,Ci]` 这种写法，loc 和 iloc 区别在于基于名称还是基于下标索引，这种切片中使用逗号分隔 Row 和 Column，并且必须将其写在一个切片中。

> 基于 Column 选择方式及其缺陷

关于 DataFrame 的索引使用，请跳转至2.1.2章节进行查看。

```python
print(frame["name"]);
print(frame.student_id)
print(frame["name"]["B"]) 
# frame提取元素第一个值可以为句点表示法，也可以像词典一样提取。
# 第一个提取的元素必须为column的index而不能是row的index，
# 比如frame["A"]就是错误的
```

需要注意： `frame[["name","student_id"]]["A"];  frame[]["A"]` 这两种都是错误的，虽然从推理上看似正确，但是这种切片语法有很多限制，比如 columns 默认不支持列表名称选取后再切片 index, 仅仅可以提取多个columns的数据: `frame[["name","student_id"]]`

可以替换/新建一整列，替换/新建的内容可以为常数（全复制，广播），列表（必须等长，否则报错），以及 Series（不必等长，但必须 index 对应 DataFrame 的 index）

> 基于 Column 的 List、Series、常数替换

```python
frame["debt"] = 1; print(frame) # 为不存在的列赋值会创建新列 
#frame["debt"] = np.arange(4);print(frame) 
# # 如果是list的话，要赋值的值的长度必须等长
#frame["debt"] = np.arange(2);print(frame) 
#其实推荐给某一列赋值为Series元素，缺失的值会自动处理。
data_s = Series([1,2,3,4],index=["A","B","C","D"]);
print(data_s);frame["new_debt"] = data_s;print(frame)
# 需要注意，这种情况下Series的index必须对应 DataFrame的index，
# 否则无法处理，全部为NaN。
```

      name  student_id  scores  debt  new_debt
    A   张三           1      30     1       NaN
    B   李四           2      42     1       NaN
    C   王五           3     110     1       NaN
    D   马六           4     150     1       NaN
    A    1
    B    2
    C    3
    D    4
    dtype: int64
    
      name  student_id  scores  debt  new_debt
    A   张三           1      30     1         1
    B   李四           2      42     1         2
    C   王五           3     110     1         3
    D   马六           4     150     1         4

```python
a = DataFrame({"name":[1,2,3],"age":[3,4,5]})
a["qq"] = 2 # 广播
a["tt"] = [3,3,3] # 等长
a["rr"] = Series([4,5,6,7],index=[3,1,2,0]) # 自动处理索引映射

	age	name	qq	tt	rr
0	3	1	2	3	7
1	4	2	2	3	5
2	5	3	2	3	6
```

删除某列可以使用 del（彻底删除） ，也可以用 drop（创建新结构），?（抛弃选中数据，而不删除该行/列，详见后文）

```python
del a["rr"] # 就地处理替换

	age	name	qq	tt
0	3	1	2	3
1	4	2	2	3
2	5	3	2	3

a.drop("qq", axis=1) # 返回新对象

	age	name	tt
0	3	1	3
1	4	2	3
2	5	3	3
```

> DataFrame 索引对象

不同于 Series，DataFrame 的 Index 不可直接修改替换，并且有多种类型，多种方法可供使用。比如 `xxx in index`。其不可修改性保证了 index 在不同数组间的传递安全，但是也不容易修改 column 的名称，尽管我们可以在生成 frame 时设置、使用多种处理索引的方法，比如 rename、reset_index、set_index、reindex。

```python
frame = DataFrame(np.arange(5),index=[1,2,3,4,5],columns=["数字"]);
print(frame.index,frame.columns,3 in frame.index,"数字" in frame.columns)
```

    Int64Index([1, 2, 3, 4, 5], dtype='int64') 
    Index(['数字'], dtype='object') 
    True 
    True

index有很多方法，比如 append() 合并索引，diff() 计算差集，intersection() 计算交集，union() 计算并集，isin() 计算包含, delete() 删除已有值，drop() 删除传入值, unique/is_unique 计算重复值。

不过需要注意，这样处理之后，返回的是一个新的 Index 对象，比如下面的 index drop。

```python
frame = DataFrame(np.arange(5),index=[1,2,3,4,5],columns=["数字"]);
print(frame)
newindex=frame.index.drop(2);print(newindex)
```

       数字
    1   0
    2   1
    3   2
    4   3
    5   4
    Int64Index([1, 3, 4, 5], dtype='int64')

## 2. DF/S对象的索引、排序和运算

### 2.1 一维索引

#### 2.1.1 Series 的索引选取

对于 Series，可以使用单个/连片/多个 × 索引值/索引名称这六种方式进行选取。

即可以使用 index、index name、index和index name的切片、index和index name的花式索引进行选取。使用 bool 表达式进行过滤。

换句话说，Series 支持 `str、int、[str], [int], [str:str], [int:int], [bool]` 七种写法。

```python
obj = pd.Series([0,1,3,2], index=['a', 'b', 'e', 'c'])
print(obj)
print(obj['b'])
print(obj[1])
print(obj[2:4])
print(obj[['b', 'a', 'e']])
print(obj[[1, 2]])
obj[obj < 2]
```

    a    0
    b    1
    e    3
    c    2
    dtype: int64
    
    1
    1
    e    3
    c    2
    dtype: int64
    
    b    1
    a    0
    e    3
    dtype: int64
    
    b    1
    e    3
    dtype: int64
    
    a    0
    b    1
    dtype: int64

需要注意 `[int:int]` 和 `[str:str]` 切片的范围不同，前者不包含后面，后者包含末端。

```python
print(obj['b':'d']) #需要注意，和index切片不同，其末端包含，封闭区间。
print(obj[1:3]) # 而Python原生切片则默认不包含末端区间
```

    b    5.0
    c    5.0
    d    3.0
    dtype: float64
    
    b    5.0
    c    5.0
    dtype: float64

选取数据后，就可以利用赋值语句对其进行赋值了。需要注意的是，这样的方式均为就地更改。

```python
obj['b':'c'] = 5
obj
```

    a    0.0
    b    5.0
    c    5.0
    d    3.0
    dtype: float64


DataFrame 的数据结构及其两种索引选取的模式示意图如下，将会在下面分章节讲解：

![](/media/mdf.png)

#### 2.1.2 DataFrame 的基本索引选取

> 行列选择

在 DataFrame 中，区别于 Python 列表和 Series 对象：

- index 不可用。

- index name 指代 column index name（使用loc方法可以作用域row index name）。`str->c`

- index的切片可用，并且作用于row index `[int:int]->r`。

- index name的切片可用，作用于row index name `[str:str]->r`。

- index的花式索引不可用。

- index name的花式索引为column index name的索引 `[[str,str]]->c` 。

简单来说，DataFrame 仅支持 `str->c, [[str,str]]->c` 和 `[int:int]->r, [str:str]->r` 这四种写法。这其实是有道理的，因为 columns 一般是 str 类型，且频繁的需要选择单列和多列，row 一般需要选择一片，其索引可能是 str 或者 int，因此将切片给 index/row 很合理。

尽管如此，这样的限制很多，比如我们不能对COLUMN进行组(片)选，不能对ROW进行单、多选，在随后介绍的loc和iloc方法可以解决这个问题。

```python
data = pd.DataFrame(np.arange(16).reshape((4, 4)),
                    index=['Ohio', 'Colorado', 'Utah', 'New York'],
                    columns=['one', 'two', 'three', 'four'])
print(data)
print(data['two'],type(data["two"]))
# print(data[1])#是错误的，不能根据column index 进行选择
# data[:2] 这样的根据index的选择反而可以正常工作（切片）
print(data[['three', 'one']],type(data[["three","one"]]))
print(data["Ohio":"Utah"])
#可用，而print(data["one":"four"])是错误的
#print(data[[1,2]]) #index的花式索引不可用
```

              one  two  three  four
    Ohio        0    1      2     3
    Colorado    4    5      6     7
    Utah        8    9     10    11
    New York   12   13     14    15
    
    Ohio         1
    Colorado     5
    Utah         9
    New York    13
    Name: two, dtype: int32 <class 'pandas.core.series.Series'>
    
              three  one
    Ohio          2    0
    Colorado      6    4
    Utah         10    8
    New York     14   12 <class 'pandas.core.frame.DataFrame'>
    
              one  two  three  four
    Ohio        0    1      2     3
    Colorado    4    5      6     7
    Utah        8    9     10    11

> 元素选择

这些都是选取某一（多）行/列的方法，如果我们需要选取某一列的某一行的元素，或者某选取列的某选取行的元素，可以使用numpy语法的 [column select] [row select] 来区分维度，但是不要使用[column select,row select]来区分维度，因为其会导致语法过于复杂，在 loc，iloc 中使用后者，这样可以清晰的将两者区分开。

比如下面的例子：`data[["one","three"]][1:3]` 选取第一、三列的第1-3行。注意，这种组合选择也必须符合之前我们说的 DF 限制，即对于 columns，只能使用 `str, [[str,str]]` 而对于 index，只能使用 `[int:int],[str:str]`:

```python
a

    age	    name	qq	tt
0	3	1	2	3
1	4	2	2	3
2	5	3	2	3

a[["age","name"]][0:1] # 不可用 [1] 索引第一行
```

此外，DF还可以接受Series作为[Series]这种形式进行选择，在这种情况下，S必须和DF的row index保持一致（即只能用column的Series对row进行筛选）

```python
print(data[:2]) # 切片在DF中作用于index，而不是column
print(data['three'] > 5) # 这生成了一个Series
print(data[data['three'] > 5]) 
#索引使用了一个bool型Series对原数组进行选择，很自然的想到是对行进行选择。
print(data)
# # 不对，因为Series必须匹配DataFrame原始的row index才可以进行选择。
#print(data["one","Colorado"])
# # 以上所有情况均只能对行/列操作，而不能同时对行和列进行操作
#（虽然后面可以根据bool索引对不符合条件的行列进行排除，但依然无法直接同时选取行与列）
```
    
              one  two  three  four
    Ohio        0    1      2     3
    Colorado    4    5      6     7
    Ohio        False
    Colorado     True
    Utah         True
    New York     True
    Name: three, dtype: bool
    
              one  two  three  four
    Colorado    4    5      6     7
    Utah        8    9     10    11
    New York   12   13     14    15
    
              one  two  three  four
    Ohio        0    1      2     3
    Colorado    4    5      6     7
    Utah        8    9     10    11
    New York   12   13     14    15
    
> 配合 bool 索引实现条件过滤

```python
data < 5
data[data < 5] = 0
data
```

    one	two	three	four
    Ohio	0	0	0	0
    Colorado	0	5	6	7
    Utah	8	9	10	11
    New York	12	13	14	15

> 设置索引别名

使用 `df.index.name = "xxx"` ; `df.column.name = "yyy"` 设置列和行的索引的别名。如果是多层索引的话，需要调用names而不是name，这一点需要注意。

> 删除行/列

使用 `del df.one` 或者 `del df["one"]` 来删除一列。或者使用 `df.drop(["Utah","New York"],axis=0)` 来删除行/列，在参数中指定axis以指定轴（0 为 index，1 为 column）。

#### 2.1.3 DataFrame 的视图索引选取

iloc和loc是针对COLUMN不能组选、ROW不能单、多选搞出来的东西。一般而言，这样的使用方式其实不多，对于长格式数据，COLUMN一般为变量/特征，而ROW一般为数据编号。

iloc即loc的index版本。这两者接受两个由逗号隔开的参数，第一个为对列进行的选择，第二个为对行进行的选择，这两个参数均可为标量、由标量组成的列表，切片。loc和iloc不是函数，直接对其进行切片即可。

**这两个参数相当于执行完第一个参数后生成temp的DataFrame后进行第二个参数的切片选择，因此可以这样写 data.iloc[data.three > 5,:3]。**

使用 loc 和 iloc 可以完整选择七种模式：`loc[Rs,Cs] --> str, [str,str], [str:str]` 和 `iloc[Ri, Ci] --> int, [int,int], [int:int]`。

```python
print(data.loc["Ohio","one"]) #注意loc并不是一个函数，是对其直接切片
print(data.loc["Ohio",["one","two"]]) 
# 同行/列的元素使用列表括住，不同的则使用逗号隔开，逗号表示不同维度，不能写成：
#print(data.loc["Ohio","Utah"]) 
# 而应该写成 print(data.loc[["Ohio","Utah"]])
print(data.loc['Colorado', ['two', 'three']])
# print(data.loc['Colorado', [1,2]]) 
# #必须使用iloc在列的基础上对行进行column index而不是column index name的选取
# print(data.loc["one"],data.loc["2"])  不能这样写，虽然之前的ix可以
```

    0
    one    0
    two    1
    Name: Ohio, dtype: int32
    
    two      5
    three    6
    Name: Colorado, dtype: int32

```python
print(data.iloc[2, [3, 0, 1]])
print(data.iloc[2])
print(data.iloc[[1, 2], [3, 0, 1]])
```

    four    11
    one      8
    two      9
    Name: Utah, dtype: int32
    
    one       8
    two       9
    three    10
    four     11
    Name: Utah, dtype: int32
    
              four  one  two
    Colorado     7    4    5
    Utah        11    8    9

```python
print(data.loc[:'Utah', 'two'])
#print(data.iloc[data.three > 5,:3])
# #很不幸，之前可以用ix这样写，现在则不可以，改为
print(data.iloc[:, :3][data.three > 5]) 
# 需要注意，index索引都是不包含最后一位的，而index name索引则包含所有
```

    Ohio        1
    Colorado    5
    Utah        9
    Name: two, dtype: int32
    
              one  two  three
    Colorado    4    5      6
    Utah        8    9     10
    New York   12   13     14

#### 2.1.4 索引的唯一和映射替换 rename

> is_unique() 判断是否唯一（对于很长的索引游泳）

使用 is_unique 判断索引是否唯一，其余和索引不重复情况类似，使用切片选取的值，如果为重复索引，则返回 series 而不是标量

```python
obj = pd.Series(range(5), index=['a', 'a', 'b', 'b', 'c'])
```

```python
obj.index.is_unique # False
```

```python
df = pd.DataFrame(np.random.randn(4, 3), index=['a', 'a', 'b', 'b'])
df.loc['b']
```

```
    0	1	2
b	2.531595	-2.638139	0.861616
b	0.617906	1.463778	-0.613723
```

> rename() 本质上是一种索引的映射关系，接受一个字典，然后对应更改

使用`Frame.rename(mapper=None, axis=None, index=None, columns=None, copy=True, inplace=False, level=None)` 进行索引的重命名。

可以单独对某个轴操作：传入 mapper 和 axis 以指定对于哪个轴进行函数映射，比如去除空格: `df.rename(lambda x: x.trim(), axis = 1)`。也可以对两个轴同时操作，传入 columns 或者 index 的字典或者函数，当传入字典时指定原始值和替换值以局部替换索引名称，当指定函数时，等同于 mapper。

```python
df.rename({1: 2, 2: 4}, axis='index')
df.rename(str.lower, axis='columns')
#可以直接使用mapper方法对索引名称进行操作
df.rename(index=str.lower, columns={"A": "a", "C": "c"})
```

#### 2.1.5 索引重排 reindex

Series的索引可以直接替换，而DF则不可以。索引替换的姊妹主题就是索引重排：DF和Series使用reindexing方法进行索引的重新界定可以新的行和列的插入、对缺失值进行处理，时间序列填充等，功能更为强大。

reindex接受一个list，没有的值默认填充NaN, 你可以指定一个fill_value来覆盖NaN这个值 ,使用method方法填充时间序列值（只能为轴0也就是横向）。如果有的话，自动使用当前数据结构中的索引的值，按照 reindex list 的顺序排列。

```python
obj = pd.Series([4.5, 7.2, -5.3, 3.6], index=['d', 'b', 'a', 'c']);print(obj)
obj2 = obj.reindex(["a","b","c","d","e"],fill_value=0.0);obj2
```

```
d    4.5
b    7.2
a   -5.3
c    3.6
dtype: float64

a   -5.3
b    7.2
c    3.6
d    4.5
e    0.0
dtype: float64

```

显然，和 rename 类似，你也可以选择一个 axis，然后对这个轴进行操作，通过 fill_value，method 来进一步设置新列表和旧 index/column 索引的映射。

```python
obj3 = pd.Series(['blue', 'purple', 'yellow'], index=[1,4,5])
print(obj3)
print(obj3.reindex(range(6), method='ffill')) 
# 在处理0，2的时候，从前面最近的开始寻找值，如果没有的话，比如0，则填充NaN
print(obj3.reindex(range(6),method="bfill")) 
# 在处理0，2值的时候，从后面最近的开始寻找值
```

```
1      blue
4    purple
5    yellow
dtype: object

0       NaN
1      blue
2      blue
3      blue
4    purple
5    yellow
dtype: object

0      blue
1      blue
2    purple
3    purple
4    purple
5    yellow
dtype: object
```

需要注意，reindex不能更改index的名称，同时，最关键的一点是 `reindex之后生成新的DataFrame，原始DataFrame不会改变`

```python
data = np.random.randint(0,100,size=9).reshape(3,3);
frame = DataFrame(data,index=["A类","B类","C类"],columns=["Henan","Hubei","Beijing"]);
print(frame)
frame2 = frame.reindex(["C类","B类","A类","D类"],fill_value=0.0);
print(frame2)
# 当进行reindex时，如果不指定参数，则默认为index，推荐指定index和column
```

```
    Henan  Hubei  Beijing
A类      7      4       39
B类      3     86       32
C类     13     84       65

    Henan  Hubei  Beijing
C类   13.0   84.0     65.0
B类    3.0   86.0     32.0
A类    7.0    4.0     39.0
D类    0.0    0.0      0.0

```

```python
print(frame2["Henan"],type(frame2["Henan"])) 
# 按列提取的数据从DF变成了Series。
print(frame2.loc["B类":]);
frame3 = frame2.copy()
frame3["Nanjing"] = frame3["Sichuan"];
frame3.drop(["Sichuan"],axis=1,inplace=True); #或者使用del命令。
print(frame3.columns)
frame4=frame3.reindex(columns=["Henan","Nanjing","Hubei"]);
print(frame4)
```

```
C类    13.0
B类     3.0
A类     7.0
D类     0.0
Name: Henan, dtype: float64 <class 'pandas.core.series.Series'>

    Henan  Hubei  Sichuan  Nanjing
B类    3.0   86.0      3.0     32.0
A类    7.0    4.0      7.0     39.0
D类    0.0    0.0      0.0      0.0
Index(['Henan', 'Hubei', 'Nanjing'], dtype='object')

    Henan  Nanjing  Hubei
C类   13.0     13.0   84.0
B类    3.0      3.0   86.0
A类    7.0      7.0    4.0
D类    0.0      0.0    0.0
```


#### 2.1.6 reset_index和set_index

> reset_index 用于删除/重置某一索引，默认删除的索引自动变为列，使用 drop 覆盖此行为，默认生成新 DataFrame，使用 inplace 覆盖此行为。当有多级索引时，通过 level 定义需要删除的索引级别。

`DataFrame.reset_index(level=None, drop=False, inplace=False, col_level=0, col_fill='')`

当一个RangeIndex中的部分索引项目被删除，比如因为值存在np.NAN类型而被删除，这个时候，使用reset_index非常方便，可以快速恢复从0开始的索引。

```python
    month	sale	year
2	2	7	84	2013
3	3	10	31	2014

df.reset_index()

    month	sale	year
0	2	7	84	2013
1	3	10	31	2014
```

```python
>>> df
               speed species
                 max    type
class  name
bird   falcon  389.0     fly
       parrot   24.0     fly
mammal lion     80.5     run
       monkey    NaN    jump

>>> df.reset_index(level='class')

         class  speed species
                  max    type
name
falcon    bird  389.0     fly
parrot    bird   24.0     fly
lion    mammal   80.5     run
monkey  mammal    NaN    jump
```

> set_index 用于将Column中的某列改变成为index，其作用和 reset_index 正好相反。keys 指定需要变换的列，可以是多列，drop 用于处理当前 index，inplace 用于定义是否赋值结构，还是就地替换。

需要使用set_index `DataFrame.set_index(keys, drop=True, append=False, inplace=False, verify_integrity=False)`

```python
df = pd.DataFrame({'month': [1, 4, 7, 10],
...                    'year': [2012, 2014, 2013, 2014],
...                    'sale':[55, 40, 84, 31]});df
```

```
month	sale	year
0	1	55	2012
1	4	40	2014
2	7	84	2013
3	10	31	2014

df.set_index(["year","month"])
```

```
        	sale
year	month	
2012	1	55
2014	4	40
2013	7	84
2014	10	31
```

#### 2.1.7 索引操作函数总结

常用且容易混淆的索引操作如下：

- [处理索引位置] reindex()用来将索引位置替换，仅此而已。

- [处理索引本身] rename()用来映射索引关系，可以在保证索引位置不变的情况下对索引进行更名、应用函数等。

此外，和rename()较为相似，但是可以同时处理索引本身并且可以更改索引位置的操作是：使用 `df.column = ["zz","zz","zz"]` 直接替换索引，列表对应当前DF的顺序。这样的好处是，不用写原始的元素名称而直接可以给index更改名称。

两个特殊的索引方法：

- reset_index()用来将某一level的索引移除并变成column列。

类似的，可以直接对index进行处理，比如：`xx_df.index.droplevel(0)` 直接删除level0的索引，而不用 `reset_index(level=0).drop("newcolumnname",axis=1)`。

- set_index()用来将某一列从列中删除，并作为index索引。对于多层索引，如下：

    df.set_index(["index1","index2"])


### 2.2 高维索引

#### 2.2.1 Series 的层次化索引

> 索引的构造

不像DF的层次化索引拥有三到四个维度，必须使用xs和IndexSlice进行选取，Series的层次化索引方便许多，只用在构造时传入一个嵌套list即可，list的第一个嵌套子list为外层（level0）索引，第二个嵌套子list为内层（level1）索引，当然，你可以建立更多索引。索引之间存在对应关系，比如，对于均有1，2，3三个level1的df来说，在构造时必须声明aaabbbccc，也就是说各层索引必须等长传入才可以构造，这样的用意很简单，不同level之间的索引可以不对称，比如a有1，2，3三个子索引，而b只有1，3两个子索引。

```python
data = Series(np.random.randn(9),index=[list("AAABBBCCC"),list("123123123")]);
print(data)
print(data["A"])
print(data["A":"C"])
print(data[["A","C"]]) #和单层索引一样，可以使用index name、切片、花式索引等
```

```
A  1    0.766414
   2    0.103040
   3   -0.081656
B  1    0.290946
   2   -0.289479
   3    1.186381
C  1   -0.472114
   2   -1.061467
   3   -2.309760
dtype: float64
1    0.766414
2    0.103040
3   -0.081656
dtype: float64
A  1    0.766414
   2    0.103040
   3   -0.081656
B  1    0.290946
   2   -0.289479
   3    1.186381
C  1   -0.472114
   2   -1.061467
   3   -2.309760
dtype: float64
A  1    0.766414
   2    0.103040
   3   -0.081656
C  1   -0.472114
   2   -1.061467
   3   -2.309760
dtype: float64

```

> 索引的选择

对于Series而言，多层索引仅仅是给索引添上了一个维度，可以使用逗号分开内外层索引，如果需要保留则用 ：即可。索引这里我们看到了为什么在上面提到的DF选择元素的时候，不能够使用[,]这种语法，而只能使用[][]这种语法的必要性。因为前者是为多层索引提供的语法。

```python
print(data[:,"2"])
print(data["A","2"]) 
print(data[:,["1","2"]])
```

```
A    0.103040
B   -0.289479
C   -1.061467
dtype: float64
0.10304022968206712

```

#### 2.2.2 DataFrame 的层次化索引

但是DF本身就有两个维度的索引，因此就必须使用idx（IndexSlice）进行区分，要选择指定行，是这个样子：.loc[idx[:,"2"],:]或者.loc(axis=0)[idx[:,"2"]]

> DF层次化索引的构造

```python
data = DataFrame(np.random.randn(9,2),
index=[list("aaabbbccc"),list("123123123")],columns=list("AB"))

data #层次索引的index 或者 column采用[[],[]]书写，分别代表其不同层次
```

```
        A	B
a	1	1.219370	0.167672
        2	0.232690	0.147475
        3	-1.886220	0.024700
b	1	1.051342	-0.810769
        2	0.770830	0.713649
        3	-1.841257	1.770243
c	1	-0.514284	0.610397
        2	-0.804532	1.362790
        3	-0.430893	1.799304

```

对于层次化索引，如下：

```python
print(data.index)
print(data.A)
print(data["A"]) #这些和单维度索引都是一样的
```

```
MultiIndex(levels=[['a', 'b', 'c'], ['1', '2', '3']],
           labels=[[0, 0, 0, 1, 1, 1, 2, 2, 2], [0, 1, 2, 0, 1, 2, 0, 1, 2]])
a  1    1.219370
   2    0.232690
   3   -1.886220
b  1    1.051342
   2    0.770830
   3   -1.841257
c  1   -0.514284
   2   -0.804532
   3   -0.430893
Name: A, dtype: float64
a  1    1.219370
   2    0.232690
   3   -1.886220
b  1    1.051342
   2    0.770830
   3   -1.841257
c  1   -0.514284
   2   -0.804532
   3   -0.430893
Name: A, dtype: float64

```

> DF多层索引的选取：切片, xs和IndexSlice

一般而言，推荐使用xs选取索引，虽然使用切片也行。xs需要指定轴axis和索引层次level，以及被索引的名称（对于单个name，传入字符串，对于多个name，传入元组）。最复杂的情况是跨层次选择索引，比如选择level0的a列和level1的1列需要指定level为（0，1），指定选取列为（"a","1"）。

```python
print(data.loc["a"])
#选取单个索引
print(data.xs("A",axis=1,level=None))
#print(data.xs("A",axis=1,level=0))#对于没有分层的column index，不能出现level值
#选取多层索引之第一层索引
print(data.xs("a",axis=0,level=0)) #选取第一层row索引，指定轴为0
#选取多层索引之第二层索引
print(data.xs("1",axis=0,level=1)) #选取第二层row索引，因为1对应有三个，因此返回了3行
#选取多层索引之多层索引
print(data.xs(("a","1"),axis=0,level=(0,1))) 
#如果一二层和选，则需要声明level为两者，row索引放在set元组里。
```

```
         A         B
1  1.21937  0.167672
2  0.23269  0.147475
3 -1.88622  0.024700
a  1    1.219370
   2    0.232690
   3   -1.886220
b  1    1.051342
   2    0.770830
   3   -1.841257
c  1   -0.514284
   2   -0.804532
   3   -0.430893
Name: A, dtype: float64
         A         B
1  1.21937  0.167672
2  0.23269  0.147475
3 -1.88622  0.024700
          A         B
a  1.219370  0.167672
b  1.051342 -0.810769
c -0.514284  0.610397
           A         B
a 1  1.21937  0.167672

```

当然，如果你愿意，也可以使用切片来选取多个层次的索引。不过，由于我们几乎已经将所有可以使用的方法都使用过了，所以这里需要创建一个idx对象，或者使用slice()来辅助选取。

比如我们要选取row，则需要使用loc[这里填入提取row index的语法,:]。那么在这个位置，我们需要使用IndexSlice进行层次化索引对象定义，比如选择a行的1，2行，则使用 `syx = IndexSlice["a",["1","2"]]` ，然后将syx传递给loc进行调用， `loc[syx,:]`。如果你不想调用IndexSlice的话，那么使用slice(None)来代替“：”,使用(slice(),slice())进行第一层和第二层的切片，比如 `syy = (slice("a"),slice(["1","2"]))`，然后传入 `loc[syy,:]` 进行调用。

```python
#简单的方式是采用切片slice(None):
print(data.loc["a"])
print(data.loc[(slice("a")),:]) #提倡采用.loc[(slice()),:] 而不是.loc[slice()]
print(data.loc[(slice(None),slice("1")),:]) #如果完全保留，则需要声明slice(None)
#其实，slice等同于IndexSlice：
idx = pd.IndexSlice
print(data.loc[idx[:,"1"],idx[:]])
#这个也可以写成：
print(data.loc(axis=0)[idx[:,"1"]]) #以省去写idx[:]的麻烦
```

> swaplevel和sortlevel

有时候，我们需要调整不同level的顺序，比如将level0的一个index换到level1，或者对于level进行排序，可以使用swaplevel和sortlevel进行操作。

DataFrame.swaplevel(i=-2, j=-1, axis=0)

```python
#data.swaplevel(0,1,axis=0) #其中level可以为int或者是index.name设置的名称
data.index.names = ["abc","num"]; #names而不是name需要注意
print(data)
data = data.swaplevel("abc","num");print(data)
```

```
                A         B
abc num                    
1   a    1.219370  0.167672
2   a    0.232690  0.147475
3   a   -1.886220  0.024700
1   b    1.051342 -0.810769
2   b    0.770830  0.713649
3   b   -1.841257  1.770243
1   c   -0.514284  0.610397
2   c   -0.804532  1.362790
3   c   -0.430893  1.799304
                A         B
num abc                    
a   1    1.219370  0.167672
    2    0.232690  0.147475
    3   -1.886220  0.024700
b   1    1.051342 -0.810769
    2    0.770830  0.713649
    3   -1.841257  1.770243
c   1   -0.514284  0.610397
    2   -0.804532  1.362790
    3   -0.430893  1.799304

```

DataFrame.sortlevel(level=0, axis=0, ascending=True, inplace=False, sort_remaining=True)

```python
#一般在进行swaplevel的时候也要进行sortlevel，
#pd现在的版本已经抛弃了这个方法，改成了sort_index by level。
print(data.sort_index(axis=0,level=1))#注意看level不同的区别
print(data.sort_index(axis=0,level=0))#data.sortlevel()
```

```
                A         B
num abc                    
1   a    1.219370  0.167672
2   a    0.232690  0.147475
3   a   -1.886220  0.024700
1   b    1.051342 -0.810769
2   b    0.770830  0.713649
3   b   -1.841257  1.770243
1   c   -0.514284  0.610397
2   c   -0.804532  1.362790
3   c   -0.430893  1.799304
                A         B
num abc                    
1   a    1.219370  0.167672
    b    1.051342 -0.810769
    c   -0.514284  0.610397
2   a    0.232690  0.147475
    b    0.770830  0.713649
    c   -0.804532  1.362790
3   a   -1.886220  0.024700
    b   -1.841257  1.770243
    c   -0.430893  1.799304

```

> 按照level进行统计

多层次索引给二维的DataFrame提供了处理高纬数据的可能，我们只添加了少数的语法，就可以操纵多层索引的行列和值，这看起来很棒。很多通用函数，比如sum、count等，都提供了level参数，传递这个参数可以对整个level进行数据计算。

### 2.3 排序和排名

> sort_index

对于 DataFrame 和 Series 都可以使用 `sort_index` 进行排序，DF可以指定axis轴（0，1），DF/S都可以指定排序方式（ascending=bool）。

此外，可以针对 row index 和 column index 进行 index 的排序（而不是行列的），使用 `index.order()` 即可。

此外，还可以使用 `sort_values` 按照 index name 的值的顺序进行排名（比如总成绩一栏），这种方法更为常见。

```python
obj = pd.Series(range(4), index=['d', 'a', 'b', 'c'])
obj.sort_index() # 默认 axis = 0，ascending = True

a    1
b    2
c    3
d    0
dtype: int64

frame = pd.DataFrame(np.arange(8).reshape((2, 4)),
                     index=['three', 'one'],
                     columns=['d', 'a', 'b', 'c'])
print(frame.sort_index()) # axis = 0
print(frame.sort_index(axis=1))
frame.sort_index().sort_index(axis=1,ascending=False) # 行列均排序
```

```
       d  a  b  c
one    4  5  6  7
three  0  1  2  3

       a  b  c  d
three  1  2  3  0
one    5  6  7  4

        d   c   b   a
one     4   7   6   5  
three   0   3   2   1
```

> sort_values

sort_values:根据值排序：对于DataFrame/Series，还可以指定某个轴的某个 index name 为排序依据，其用法如下：

```
by : str or list of str
    Name or list of names to sort by. if axis is 0 or ‘index’ then by may contain index levels and/or column labels, if axis is 1 or ‘columns’ then by may contain column levels and/or index labels Changed in version 0.23.0: Allow specifying index or column level names.
    
axis : {0 or ‘index’, 1 or ‘columns’}, default 0. Axis to be sorted

ascending : bool or list of bool, default True, 
    Sort ascending vs. descending. Specify list for multiple sort orders. If this is a list of bools, must match the length of the by.
    
inplace : bool, default False, if True, perform operation in-place
```

如下所示，和 sort_index 根据 Index、Column 的字母顺序排序不同，sort_values 根据 Index、Column 的值排序，因此，需要指定 axis，默认为 0，这里的 0 指的是在某一、几列上对于行进行排序。如果选择 1，则指的是对于某一、几行上对于列进行排序。

```python
a

	age	   name	qq	tt
0	3	1	2	3
1	4	2	2	3
2	5	3	2	3

a.sort_values(by=["name"],axis=0,ascending=False)

    age	name	qq	tt
2	5	3	2	3
1	4	2	2	3
0	3	1	2	3
```

如果包含 NaN 值，那么其将会被放在最后，注意，对于任意的 ascending 都是如此。

```python
obj = pd.Series([4, np.nan, 7, np.nan, -3, 2])
obj.sort_values(ascending=False)
obj.sort_values(ascending=True)
```

```
2    7.0
0    4.0
5    2.0
4   -3.0
1    NaN
3    NaN
dtype: float64

4   -3.0
5    2.0
0    4.0
2    7.0
1    NaN
3    NaN
dtype: float64
```

此外，除了对于单个列进行 axis = 0 的排序，还可以对于多个列排序，如果第一个相等的话，通过第二个来确定最终的顺序。

```python
frame.a = Series([1,1],index=["three","one"])
frame.sort_values(by=['a', 'b'],ascending=False) 
#也可以设置两个项，先对第一个排，如果第一个相等的话，对第二个排。

        d	a	b	c
one     4	1	6	7
three	0	1	2	3
```

> idxmax/idxmin

pd.idxmax() 和 pd.idxmin() 可以对于行列两个索引进行对应单元格的值的大小的比较，然后返回某行/列对应的最大/小值所在的某列/行。其接受的第一个参数为axis。**默认的 axis 为 0，即判断每行在不同列上的最大值最小值，而当 axis 为 1 的时候，就会判断每列在不同行上的最大值最小值**。比如：

    day     a       b
    01      12      23
    02      122     123
    03      15      13

    idxmax(0)
    a 01
    b 02

    idxmax(1)
    01 b
    02 b
    03 a

> `rank()` 排名

rank 对于某行、列的数据进行排名，axis 定义对于列还是行进行排名，method 决定排名使用的方法，numeric_only 决定是否只对于数字排名，ascending 决定是升序排名还是降序排名。

```python
obj = pd.Series([7, -5, 7, 4, 2, 0, 4])
obj.rank() #rank返回的值为index和排名

obj.rank(method='first')
#first含义为根据首次出现的排名为1，此外还有min，max，average（default）
obj.rank(ascending=False, method='max')
# 最大化排名，如果有两个相等值，则均为最大值。
obj.rank(ascending=False, method='min')
# 最小化排名，如果有两个相等值，则均为最小值。
```

对于 DataFrame 而言，需要指定 axis 轴，其对轴上的数值进行排序，比如 column 横向排序，index 纵向排序

```python
frame = pd.DataFrame({'b': [4.3, 7, -3, 2], 'a': [0, 1, 0, 1],
                      'c': [-2, 5, 8, -2.5]})
frame.rank(axis='columns')
```

```
   a    b    c
0  0  4.3 -2.0
1  1  7.0  5.0
2  0 -3.0  8.0
3  1  2.0 -2.5

a	b	c
0	2.0	3.0	1.0
1	1.0	3.0	2.0
2	2.0	1.0	3.0
3	2.0	3.0	1.0
```

### 2.4 矢量和标量运算

#### 2.4.1 基本算术运算

##### 2.4.1.1 Series 之间算术运算

两个 Series 之间进行的算术运算可以直接按照 index 对应的顺序进行操作，使用运算符 “+ - * /” 或者通用函数 `sum、dot、add` 等均可。

此外，标量和 Series 之间的运算，则会被广播。

```python
s1 = Series(np.arange(5),index=list("abcde"));
s2 = Series(np.arange(6),index=list("adcbef"));
s1 + s2;
s3 = s1.add(s2,fill_value=0);print(s3)
#fll_value的意思不是最终将NaN替换为0，而是在进行运算前对于缺失的值设置为0
```

    a    0.0
    b    4.0
    c    4.0
    d    4.0
    e    8.0
    f    5.0
    dtype: float64

##### 2.4.1.2 DF之间的算术运算

```python
df1 = DataFrame(np.random.randint(10,100,size=9).reshape(3,3),
columns=("Wuhan Hubei HeNan".split(" ")));print(df1)

df2 = DataFrame(np.random.randint(10,100,size=16).reshape(4,4),
columns=("Wuhan Hubei HeNan ShanXi".split(" ")));print(df2)
```

       Wuhan  Hubei  HeNan
    0     68     66     39
    1     99     47     92
    2     90     51     55
    
       Wuhan  Hubei  HeNan  ShanXi
    0     73     38     41      16
    1     89     96     40      71
    2     42     86     16      55
    3     80     45     43      71


```python
df_s = df1 + df2;
df_s2 = df1.add(df2,fill_value=0);print(df_s2)
```

       HeNan  Hubei  ShanXi  Wuhan
    0   80.0  104.0    16.0  141.0
    1  132.0  143.0    71.0  188.0
    2   71.0  137.0    55.0  132.0
    3   43.0   45.0    71.0   80.0


一些常见的替代运算符的方法有：add、sub、div、mul 加减乘除。此外，数字和 DF 之间的运算也会被广播。

##### 2.4.1.3 DF和S之间的算术运算

只要 index 对应即可进行，series 按照 index 顺序对 dataframe 进行遍历广播运算。

```python
arr = np.arange(12.).reshape((3, 4))
print(arr)
print(arr[0])
print(arr - arr[0])
```

    [[ 0.  1.  2.  3.]
     [ 4.  5.  6.  7.]
     [ 8.  9. 10. 11.]]
    
    [0. 1. 2. 3.]
    
    [[0. 0. 0. 0.]
     [4. 4. 4. 4.]
     [8. 8. 8. 8.]]


```python
frame = pd.DataFrame(np.arange(12.).reshape((4, 3)),
                     columns=list('bde'),
                     index=['Utah', 'Ohio', 'Texas', 'Oregon'])
series = frame.iloc[0]
series2 = frame.b
print(frame)
print(series,series2)
```

              b     d     e
    Utah    0.0   1.0   2.0
    Ohio    3.0   4.0   5.0
    Texas   6.0   7.0   8.0
    Oregon  9.0  10.0  11.0
    
    b    0.0
    d    1.0
    e    2.0
    Name: Utah, dtype: float64 Utah      0.0
    
    Ohio      3.0
    Texas     6.0
    Oregon    9.0
    Name: b, dtype: float64


尴尬的是，series的index必须和Frame的row index对应（也就是必须为行而不能为列），否则计算会出现奇怪的结果，如下series2


```python
print(frame - series)
print(series-frame)
print(frame-series2)
```

              b    d    e
    Utah    0.0  0.0  0.0
    Ohio    3.0  3.0  3.0
    Texas   6.0  6.0  6.0
    Oregon  9.0  9.0  9.0
    
              b    d    e
    Utah    0.0  0.0  0.0
    Ohio   -3.0 -3.0 -3.0
    Texas  -6.0 -6.0 -6.0
    Oregon -9.0 -9.0 -9.0
    
            Ohio  Oregon  Texas  Utah   b   d   e
    Utah     NaN     NaN    NaN   NaN NaN NaN NaN
    Ohio     NaN     NaN    NaN   NaN NaN NaN NaN
    Texas    NaN     NaN    NaN   NaN NaN NaN NaN
    Oregon   NaN     NaN    NaN   NaN NaN NaN NaN



```python
series2 = pd.Series(range(3), index=['b', 'e', 'f']) 
#如果构造一个series，则其index必须和Frame的index对应，否则将产生大量NaN
frame + series2
```
    b	d	e	f
    Utah	0.0	NaN	3.0	NaN
    Ohio	3.0	NaN	6.0	NaN
    Texas	6.0	NaN	9.0	NaN
    Oregon	9.0	NaN	12.0	NaN

frame.sub 中sub的意思是清除数据，而不是删除该行/列

```python
series3 = frame['d']
series4 = frame.loc["Ohio"]
print(frame);print(series3,series4)
newframe1 = frame.sub(series3, axis='index');print(newframe1)
newframe = frame.sub(series4,axis=1);print(newframe)
```

              b     d     e
    Utah    0.0   1.0   2.0
    Ohio    3.0   4.0   5.0
    Texas   6.0   7.0   8.0
    Oregon  9.0  10.0  11.0
    
    Utah       1.0
    Ohio       4.0
    Texas      7.0
    Oregon    10.0
    Name: d, dtype: float64 b    3.0
    
    d    4.0
    e    5.0
    Name: Ohio, dtype: float64
    
              b    d    e
    Utah   -1.0  0.0  1.0
    Ohio   -1.0  0.0  1.0
    Texas  -1.0  0.0  1.0
    Oregon -1.0  0.0  1.0
    
              b    d    e
    Utah   -3.0 -3.0 -3.0
    Ohio    0.0  0.0  0.0
    Texas   3.0  3.0  3.0
    Oregon  6.0  6.0  6.0


#### 2.4.2 使用函数和映射对DF的行、列进行计算

DataFrame 也可以使用原本用于 numpy 的各种数学方法，比如 abs、max、min 等，这些函数被称之为通用函数。这些数学方法会广播到具体的元素级别然后执行。如果定义 level，那么可以对整个 level 层进行运算。

DF 计算的三个维度，**其一为整体计算，其二为行列计算，其三为选定元素计算**。apply 可以进行行列计算。applymap 可以对行列进行格式化。map 方法不仅可以传递参数，还能传递类似词典的对象，mapper 会根据词典对应关系进行一一对应并且返回结果。

简单来说，map 一般用于对 Series、从 DataFrame 中提取的 Series（行或者列） 进行函数映射，applymap 可以对 DataFrame 拍平（每列连在一起）后的每个元素进行函数映射。apply 则根据 axis 字段对于 DataFrame 的每个行、列进行函数映射。


```python
frame = pd.DataFrame(np.random.randn(4, 3), columns=list('bde'),
                     index=['Utah', 'Ohio', 'Texas', 'Oregon'])
frame
np.abs(frame)
```

	b	d	e
	Utah	0.979905	0.480454	1.323365
	Ohio	0.844071	0.180990	0.562072
	Texas	1.562622	1.150573	0.896748
	Oregon	1.797835	1.089074	1.794049


另一个常见的操作是，将函数应用于各行列组成的一维数组上，类似于每行求平均数、总和之类的 Excel 应用。

> apply() 技术

apply() 需要指定一个轴，指定是否广播，接受一个函数并且对其结果进行综合后生成新的DataFrame/Series(如果返回的是标量，则综合标量和index name返回Series，如果返回的是Series，则聚合index name成为新的DataFrame)

apply()通常和lambda一起使用，构成非常 `pandorable` 并且快速的数据处理方式。至于applymap()，由于我们并不经常需要将数据flat化再处理，所以使用到的场合很少。

其语法如下：`DataFrame.apply(func, axis=0, broadcast=None, raw=False, reduce=None, result_type=None)`

```python
print(frame);
               b         d         e
Utah    0.979905 -0.480454 -1.323365
Ohio   -0.844071  0.180990 -0.562072
Texas   1.562622 -1.150573  0.896748
Oregon  1.797835  1.089074  1.794049


f = lambda x: x.max() - x.min()
print(frame.apply(f)) #产生返回值

b    2.641906 -> Oregon - Ohio
d    2.239646 -> Oregon - Texas
e    3.117414 -> Oregon - Utah
dtype: float64

newframe = frame.apply(np.sum,axis="index");
print(newframe)

b    3.496291
d   -0.360963
e    0.805360
dtype: float64

newframe2 = frame.apply(np.sum,axis="columns") 
print(newframe2,type(newframe2));

Utah     -0.823914
Ohio     -1.225153
Texas     1.308798
Oregon    4.680958
dtype: float64 <class 'pandas.core.series.Series'>
```   

需要注意的是，这里的 axis 指的是，对于 index 还是 columns 进行 function 的 apply，因此对于 index 而言，返回的是不同 column 的 function(index) 结果。  

```python
def f(x):
    return pd.Series([x.min(), x.max()], index=['min', 'max'])
print(frame);print(frame.apply(f,axis=0)) 
#b、d、e的所有数据都被当作一维np数组传入f(x),之后返回了三个Series，
# series名字分别是b、d、e，均有两个元素，作为一个DataFrame最后返回。

               b         d         e
Utah    0.979905 -0.480454 -1.323365
Ohio   -0.844071  0.180990 -0.562072
Texas   1.562622 -1.150573  0.896748
Oregon  1.797835  1.089074  1.794049
    
            b         d         e
min -0.844071 -1.150573 -1.323365
max  1.797835  1.089074  1.794049
```

> map() 和 applymap() 技术

map() 针对 Series 的每一个值进行函数处理，而 applymap() 是 map() 针对 DF 进行的一个封装，用于自动对 DataFrame 每列 Series 进行处理。（即 applymap 的遍历顺序和 df.values 不同，其并非按照行顺序进行迭代，而是按照 column ，每一个 Series 进行迭代，即竖着迭代一个 column 后继续第二个column）

使用 applymap 可以对DF进行格式化，使用 map 可以对 Series 进行格式化。一般而言，列的 dtype 可能不同，因此，使用 map 对 Series 处理更为常用，相比较使用 applymap 对于 DataFrame 而言。

```python
frame["f"] = frame["d"].map(lambda x:"%.4f"%x)

#format = lambda x: '%.2f' % x
#frame.applymap(format)
frame
        b	  d	  e	  f
Utah	0.979905	-0.480454	-1.323365 -0.4805
Ohio	-0.844071	0.180990	-0.562072 0.1810
Texas	1.562622	-1.150573	0.896748 -1.1506
Oregon	1.797835	1.089074	1.794049 1.0891

frame['e'].map(format)

Utah      -1.32
Ohio      -0.56
Texas      0.90
Oregon     1.79
Name: e, dtype: object
```

```python
a
    age	name	qq	tt
0	3	1	2	3
1	4	2	2	3
2	5	3	2	3

a.applymap(lambda x: str("%.3f"%x))
	age	name	qq	tt
0	3.000	1.000	2.000	3.000
1	4.000	2.000	2.000	3.000
2	5.000	3.000	2.000	3.000
```

### 2.5 描述统计和数据汇总

#### 2.5.1 基本描述统计

基本的描述统计可以针对行、列进行，此外，可以选择是否去除 NA 值。**这类功能建立在没有缺失值的情况下这一假设之上**。三个重要的参数是 axis，skipna，level（针对重复索引）。

常用的方法如下：**count()**计算非缺失值数量，**quantile**计算分位数，**median**计算中位数，**mad**计算绝对离差，**var**方差，**std**标准差，**skew**偏度，**kurt**峰度、**pct_change**百分数变化


```python
df = pd.DataFrame([[1.4, np.nan], [7.1, -4.5],
                   [np.nan, np.nan], [0.75, -1.3]],
                  index=['a', 'b', 'c', 'd'],
                  columns=['one', 'two'])

    one	two
a	1.40	NaN
b	7.10	-4.5
c	NaN	NaN
d	0.75	-1.3
```


```python
df.sum() # 默认 axis 为 0，即 index。

one    9.25
two   -5.80
dtype: float64

df.sum(axis='columns')

a    1.40
b    2.60
c    0.00
d   -0.55
dtype: float64
```

```python
df.mean(axis='columns', skipna=False)

a      NaN
b    1.300
c      NaN
d   -0.275
dtype: float64
```

```python
df.idxmax() # 返回对应轴max值得key（index）

one    b
two    d
dtype: object
```

```python
df.cumsum() # 计算累加
df.describe() # 一个综合的方法，返回count、mean、std、min、四分位数和max值
```

```python
obj = pd.Series(['a', 'a', 'b', 'c'] * 4);
obj.describe()

count     16
unique     3
top        a
freq       8
dtype: object
```

#### 2.5.2 相关系数和协方差

使用 corr 和 corrwith、cov 函数，计算 Series 和 Series、Series 和 DataFrame、DataFrame 之间的相关系数和协方差。


```python
price = pd.read_pickle('examples/yahoo_price.pkl')
volume = pd.read_pickle('examples/yahoo_volume.pkl')
```

```python
returns = price.pct_change();print(returns.head())
returns.tail()
```

                    AAPL      GOOG       IBM      MSFT
    Date                                              
    2010-01-04       NaN       NaN       NaN       NaN
    2010-01-05  0.001729 -0.004404 -0.012080  0.000323
    2010-01-06 -0.015906 -0.025209 -0.006496 -0.006137
    2010-01-07 -0.001849 -0.023280 -0.003462 -0.010400
    2010-01-08  0.006648  0.013331  0.010035  0.006897


	AAPL	GOOG	IBM	MSFT
	Date				
	2016-10-17	-0.000680	0.001837	0.002072	-0.003483
	2016-10-18	-0.000681	0.019616	-0.026168	0.007690
	2016-10-19	-0.002979	0.007846	0.003583	-0.002255
	2016-10-20	-0.000512	-0.005652	0.001719	-0.004867
	2016-10-21	-0.003930	0.003011	-0.012474	0.042096



```python
print(returns['MSFT'].corr(returns['IBM'])) # corr计算两个Series之间的相关系数
print(returns['MSFT'].cov(returns['IBM'])) # cov计算两个Series之间的协方差
```

    0.49976361144151155
    8.870655479703546e-05



```python
print(returns.corr())
print(returns.cov()) 
# 如果针对某一DataFrame进行没有参数的协方差/相关系数运算，
# 则返回其相关矩阵/协方差矩阵
```

              AAPL      GOOG       IBM      MSFT
    AAPL  1.000000  0.407919  0.386817  0.389695
    GOOG  0.407919  1.000000  0.405099  0.465919
    IBM   0.386817  0.405099  1.000000  0.499764
    MSFT  0.389695  0.465919  0.499764  1.000000
              AAPL      GOOG       IBM      MSFT
    AAPL  0.000277  0.000107  0.000078  0.000095
    GOOG  0.000107  0.000251  0.000078  0.000108
    IBM   0.000078  0.000078  0.000146  0.000089
    MSFT  0.000095  0.000108  0.000089  0.000215


```python
returns.corrwith(returns.IBM) 
#不仅可以比较两个Series之间的协方差/相关，还可以计算矩阵，
# 还可以计算某个Series和Frame之间的关系
```

    AAPL    0.386817
    GOOG    0.405099
    IBM     1.000000
    MSFT    0.499764
    dtype: float64


```python
returns.corrwith(volume) #没有covwith
```

    AAPL   -0.075565
    GOOG   -0.007067
    IBM    -0.204849
    MSFT   -0.092950
    dtype: float64


## 3. pandas文件读写和数据概览

### 3.1 文件读取

使用 `pd.read_csv("url",kwargs*)` 进行CSV格式文件读写，一般而言，可以直接传递URL地址而不需要调用requests或者bs4等库进行手动解析。可以在此处直接设定column索引列名称，index索引列，以及大文件高速读写、忽略错误行等。需要注意，pandas内置了很多读写函数，比如read_csv, read_table，这些函数之间有一定的差别，最好使用比较接近的函数进行相应格式的读写。

- sep="\t" 以Table分割数据

- index_col: int or sequence or False 设置某一列为index列

- skiprows: integer or callable, default None 设置某一行为标题行

- columns = ["xxx","yyy"] 设置column索引名

- low_memory = False 关闭低内存限制

- engine = "Python" 一般不要用python引擎

- error_bad_lines = False 跳过错误行

> 使用StringIO进行读写

比如，对于下面这个数据集，分割符号为空格，但是空格数不总是三个，因此需要使用Python进行读，之后做一下转换。之后将这个变量直接传递给StringIO,直接就可以像正常的表一样读取了。

    f = open("wind.data","r+").read()
    f2 = f.replace("   "," ").replace("  "," ").replace("    "," ")

    import io
    d = io.StringIO(f2)
    data = pd.read_table(d,sep=" ")


### 3.2 数据概览

使用 **df.info()** 可以每个行列的数值类型、索引类型、内存占用。这可能很重要，因为涉及类型问题，对于数值，DataFrame 会自动将其转换为数值类型，而非 str 类型。

使用 **df.size() df.shape[x] df.ndim() df.dtypes df.astype(xxx)** 可以像 numpy一样统计行列和总格数,进行格式转换等。

使用 **df.index** 和 **df.columns** 可以查看行列索引。因为选取数据需要处理索引，因此确定索引没有包含空格很重要，必要时使用 rename 来去除索引空格。

使用 **df.head(n) 和 df.tail(n)** 可以查看前/后 n 行数据。

使用 **df.xxx.describe()** 可以快速预览某一列的情况，使用 **include="all"** 参数可以返回所有列。如果列的值为描述数据，则返回 unique、count、top、freq 值。这可以为我们提醒大量数据中的一些问题，除了使用这种方法以外，使用 numpy 的 bool 切片也是一个好主意。


——————————————————————————————————————————

> 更新日志

2018-02-25 阅读《利用Python进行数据分析》，完成笔记整理和博客发布。

2018-03-22 阅读《Python数据分析实战》，整理了部分措辞，添加了一些描述文字。

2018-03-27 细节调整，添加了标题3 文件读写和数据概览内容。

2018-03-28 对索引操作函数进行了总结，澄清了一些错误。

2018-04-25 简化索引部分内容，梳理逻辑，添加对于loc、iloc理解，修正错误。

2019-06-06 更新了部分的用例，整理了整体的思路。重写了 rename、reindex、reset_index 和 set_index 部分，重写了 DataFrame 两种数据选取策略 —— 基本的多切片和基于 loc 和 iloc 的逗号分隔切片部分。