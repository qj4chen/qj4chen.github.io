---
layout: post
title: Python数据处理学习笔记 - pandas数据分组和聚合篇
categories:
  - Python数据处理
  - 读书笔记
  - pandas
---

> 这是我阅读《用Python进行数据分析》一书的笔记、实验和总结。本篇文章主要讲解pandas包中数据分组和聚合技术，主要涉及groupby、aggregate、apply、transform、(q)cut、透视表。数据分组和聚合是对DataFrame进行分析和处理的关键步骤，尤其是apply()，其提供了一个编写函数进行运算的强大接口，正因如此，pandas的agg技术比MongoDB等数据库的agg技术更加先进、灵活、高效。

约定俗成的，引入以下包：

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from pprint import pprint
```

在上一章介绍过map函数可以对行列进行运算，但是，大多数时候我们需要对某些行列进行运算，我们可以使用切片来选取行列之后计算，或者使用bool型函数来筛选符合条件的行列后进行运算。如果我们需要对于多个行列进行合并，比如合并二级level，这个时候使用索引操纵列进行map是通用的做法。但是由于这些需求过于常见但是这些操作过于复杂，所以就应运而生了聚合。这里的聚合和数据库的聚合技术是一种东西，不过却灵活、强大很多，它可以按照一定的规则对于行列进行拆分，然后进行某一种运算或者多种或者你自定义的运算，然后进行各种不同层级的数据合并（得益于Python和ndarray）。

数据分组和聚合本质上是按照索引将数据进行分组、计算，然后再合并(SPLIT.APPLY.COMBINE)的过程。对于分组，我们使用groupBy函数，对于分过的组进行计算，可以使用aggregate或者apply来进一步操作。aggregate本质上都是分组计算的一种，不过其更经常生成标量，比如sum、count等（类似MongoDB的aggregate技术）。而使用apply则可以组合各种自己编写的函数，返回各种矢量并进行concat最后生成表。

## 1. 数据分组：GroupBy技术

> 分组的依据选择

groupby可以对字符串（列名称）、numpy类型、等长对等索引的数组、Series、字典（列名称的map关系）、函数、索引层次进行分组。大体来说，只要是和index对应的等长列表（lsit），都可以进行分组。而其余的字段，比如column name、function map，本质上都是转换成为list再进行groupby操作。

> 分组状况查看

对于分过的组，可以使用grouped.groups进行查看，结果会返回一个字典。对于分过的组，也可以进行迭代，在后面会详细介绍分组对象的迭代。

### 1.1 通过list进行分组、多层分组

> 一个简单的例子

```python
df = pd.DataFrame({'key1' : ['a', 'a', 'b', 'b', 'a'],
                   'key2' : ['one', 'two', 'one', 'two', 'one'],
                   'data1' : np.random.randn(5),
                   'data2' : np.random.randn(5)})
```

          data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    1 -0.296581  0.897673    a  two
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two
    4 -0.651637  0.749137    a  one
    

> 只要一个等row index length的series、list即可进行分组


```python
grouped = df['data1'].groupby(df['key1']) 
#使用Series对于Series进行分组,它们具有完全等同的row index。
print(grouped)
grouped.mean()
```

    <pandas.core.groupby.SeriesGroupBy object at 0x000001B80461ABE0>
    
    key1
    a   -0.286255
    b   -0.711685
    Name: data1, dtype: float64



> 可以使用多key分组

类似于MongoDB的$group{a,b}分层分组。


```python
means = df['data1'].groupby([df['key1'], df['key2']]).mean() 
#也可以使用一个[]，进行多层的分组，类似于SQL的多Key聚合。
print(means)
print(means.unstack())
```

    key1  key2
    a     one    -0.281093
          two    -0.296581
    b     one    -0.887990
          two    -0.535381
    Name: data1, dtype: float64

    key2       one       two
    key1                    
    a    -0.281093 -0.296581
    b    -0.887990 -0.535381
    

> 引入外部的list而不是内部的column列也可以


```python
states = np.array(['Ohio', 'California', 'California', 'Ohio', 'Ohio'])
years = np.array([2005, 2005, 2006, 2005, 2006])
df['data1'].groupby([states, years]).mean()
```

    California  2005   -0.296581
                2006   -0.887990
    Ohio        2005   -0.222965
                2006   -0.651637
    Name: data1, dtype: float64



> 可以指定column列name

其列自动会被删除而不会进行分组。这本质上是一种语法糖，填入一个Str的时候会自动对column name进行查找，类似于：

`df.groupby([df["key1"],df["key2"]]).mean()`


```python
print(df.groupby('key1').mean())
print(df.groupby(['key1', 'key2']).mean())
print(df.groupby([df["key1"],df["key2"]]).mean())
```

             data1     data2
    key1                    
    a    -0.286255  0.059693
    b    -0.711685 -0.074938
                  data1     data2
    key1 key2                    
    a    one  -0.281093 -0.359298
         two  -0.296581  0.897673
    b    one  -0.887990  1.029688
         two  -0.535381 -1.179564
                  data1     data2
    key1 key2                    
    a    one  -0.281093 -0.359298
         two  -0.296581  0.897673
    b    one  -0.887990  1.029688
         two  -0.535381 -1.179564
    

mean用于将被聚合的数据求平均值，而size则对于groupby对象进行count计数


```python
print(df.groupby(['key1', 'key2']).mean())
df.groupby(['key1', 'key2']).size()
```

                  data1     data2
    key1 key2                    
    a    one  -0.281093 -0.359298
         two  -0.296581  0.897673
    b    one  -0.887990  1.029688
         two  -0.535381 -1.179564
    
    key1  key2
    a     one     2
          two     1
    b     one     1
          two     1
    dtype: int64


### 1.2 通过字典或者Series进行分组


```python
people = pd.DataFrame(np.random.randn(5, 5),
                      columns=['a', 'b', 'c', 'd', 'e'],
                      index=['Joe', 'Steve', 'Wes', 'Jim', 'Travis'])
people.iloc[2:3, [1, 2]] = np.nan # Add a few NA values
print(people)
```

                   a         b         c         d         e
    Joe     0.222491 -0.995911 -1.014604 -0.067544 -0.869915
    Steve   0.777247 -1.353332  0.821747 -1.104266  0.156773
    Wes    -0.080765       NaN       NaN  0.737536  0.132357
    Jim    -0.733980 -0.728372 -1.101054 -0.518711 -0.099602
    Travis -0.204770 -1.357502  1.421094  0.523682  1.917264
    


```python
mapping = {'a': 'red', 'b': 'red', 'c': 'blue',
           'd': 'blue', 'f': 'red', 'e' : 'orange'} 
           #mapping中可以存在没有对应的列，程序不会对其进行匹配
```


```python
by_column = people.groupby(mapping, axis=1)
print(by_column.sum())
```

                blue    orange       red
    Joe    -1.082148 -0.869915 -0.773420
    Steve  -0.282519  0.156773 -0.576085
    Wes     0.737536  0.132357 -0.080765
    Jim    -1.619765 -0.099602 -1.462352
    Travis  1.944775  1.917264 -1.562272
    


```python
map_series = pd.Series(mapping)
map_series
print(people.groupby(map_series, axis=1).count())
```

            blue  orange  red
    Joe        2       1    2
    Steve      2       1    2
    Wes        1       1    1
    Jim        2       1    2
    Travis     2       1    2
    

区别于map和funcmap，这里的对各列进行的map分组本质上还是分组，其不保留各原始列的内容。

### 1.3 通过函数进行分组

传入一个函数作为groupby对象，函数会自动将index列各个index name值作为变量传入，然后返回一个新的值。并以这个新的值作为分组的名称。


```python
print(people)
print(people.groupby(len).sum())
```

                   a         b         c         d         e
    Joe     0.222491 -0.995911 -1.014604 -0.067544 -0.869915
    Steve   0.777247 -1.353332  0.821747 -1.104266  0.156773
    Wes    -0.080765       NaN       NaN  0.737536  0.132357
    Jim    -0.733980 -0.728372 -1.101054 -0.518711 -0.099602
    Travis -0.204770 -1.357502  1.421094  0.523682  1.917264
              a         b         c         d         e
    3 -0.592254 -1.724283 -2.115659  0.151281 -0.837160
    5  0.777247 -1.353332  0.821747 -1.104266  0.156773
    6 -0.204770 -1.357502  1.421094  0.523682  1.917264
    

函数还可以作为分组的多个key之一，因为其最后还是会被转换成为一个数组，非常灵活。


```python
key_list = ['one', 'one', 'one', 'two', 'two']
print(people.groupby([len, key_list]).min())
```

                  a         b         c         d         e
    3 one -0.080765 -0.995911 -1.014604 -0.067544 -0.869915
      two -0.733980 -0.728372 -1.101054 -0.518711 -0.099602
    5 one  0.777247 -1.353332  0.821747 -1.104266  0.156773
    6 two -0.204770 -1.357502  1.421094  0.523682  1.917264


### 1.4 分组的迭代和内部结构

使用 `grouped.groups` 可以查看分组状况。如果有更多的需求，可以对分组对象进行迭代。groupby对象迭代，其返回一个元组，其第一个对象为分组的name，如果是单key group，则返回对象是一个str字符串，如果是多key group，则这个name则是一个包含多key name的tuple。

```python
print(df.groupby('key1').mean())
for name, group in df.groupby('key1'):
    print(name)
    print(group)
```

             data1     data2
    key1                    
    a    -0.286255  0.059693
    b    -0.711685 -0.074938
    a
          data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    1 -0.296581  0.897673    a  two
    4 -0.651637  0.749137    a  one
    b
          data1     data2 key1 key2
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two
    

```python
for (k1, k2), group in df.groupby(['key1', 'key2']):
    print((k1, k2))
    print(group)
for x in df.groupby(['key1', 'key2']):
    print(x)
    break
```

    ('a', 'one')
          data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    4 -0.651637  0.749137    a  one

    ('a', 'two')
          data1     data2 key1 key2
    1 -0.296581  0.897673    a  two

    ('b', 'one')
         data1     data2 key1 key2
    2 -0.88799  1.029688    b  one

    ('b', 'two')
          data1     data2 key1 key2
    3 -0.535381 -1.179564    b  two

    (('a', 'one'),       
    data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    4 -0.651637  0.749137    a  one)
    

本质上，迭代返回的对象是：

`[((name1,namea),ndarray),((name1,nameb),ndarray),((name2,namea),ndarray),((name2,nameb),ndarray)]`

这很容易的利用dict和list函数来讲groupby对象变成一个以key name为key，以其聚合为value的dict。


```python
pieces = dict(list(df.groupby('key1')))
pprint(pieces)
print(pieces['b'])
```

    {'a':       
    data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    1 -0.296581  0.897673    a  two
    4 -0.651637  0.749137    a  one,

     'b':       
     data1     data2 key1 key2
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two}

          data1     data2 key1 key2
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two

    
### 1.5 组和列的切片选取

`df.groupby(["key1"])["data1"].mean()` 本质上是 `df["data1"].groupby(["key1"]).mean()` 的语法糖

`df.groupby(["key1"])[["data1"]].mean()` 本质上是 `df[["data1"]].groupby(["key1"]).mean()` 的语法糖

对于前者，其返回子类Groupby对象，不保留切片索引。对于后者，其仅仅是对整体的结果进行切片，保留了索引。


```python
print(df.groupby(['key1', 'key2'])[['data2']].mean())
```

                  data2
    key1 key2          
    a    one  -0.359298
         two   0.897673
    b    one   1.029688
         two  -1.179564
    

```python
print(df.groupby(['key1', 'key2'])['data2'].mean())
print(df.groupby(['key1', 'key2'])[['data1',"data2"]].mean())
print(df.groupby(['key1', 'key2'])['data1','data2'].mean())
```

    key1  key2
    a     one    -0.359298
          two     0.897673
    b     one     1.029688
          two    -1.179564
    Name: data2, dtype: float64
                  data1     data2
    key1 key2                    
    a    one  -0.281093 -0.359298
         two  -0.296581  0.897673
    b    one  -0.887990  1.029688
         two  -0.535381 -1.179564
                  data1     data2
    key1 key2                    
    a    one  -0.281093 -0.359298
         two  -0.296581  0.897673
    b    one  -0.887990  1.029688
         two  -0.535381 -1.179564
    

区别是，采用【】这种选择方法而不是【【】】生成的对象不是groupby而是serise/dataframegroupby对象（如果选择某一列进行计算的话）。

对于多个列进行选择的话，不论是【】还是【【】】，其生成的对象都是一样的。

### 1.6 axis=1的groupby

最后，也可以传递df.dtypes来作为分组依据。这里的第二个知识点是，可以指定axis为1来对column进行group，之前所有的实例都是对axis=0的row index进行的分组。


```python
print(df)
print(df.dtypes)
grouped = df.groupby(df.dtypes, axis=1)
```

          data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    1 -0.296581  0.897673    a  two
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two
    4 -0.651637  0.749137    a  one

    data1    float64
    data2    float64
    key1      object
    key2      object
    dtype: object
    

```python
for dtype, group in grouped:
    print(dtype)
    print(group)
```

    float64
          data1     data2
    0  0.089452 -1.467732
    1 -0.296581  0.897673
    2 -0.887990  1.029688
    3 -0.535381 -1.179564
    4 -0.651637  0.749137

    object
      key1 key2
    0    a  one
    1    a  two
    2    b  one
    3    b  two
    4    a  one

### 1.7 level=2的groupby


```python
columns = pd.MultiIndex.from_arrays([['US', 'US', 'US', 'JP', 'JP'],
                                    [1, 3, 5, 1, 3]],
                                    names=['cty', 'tenor'])
hier_df = pd.DataFrame(np.random.randn(4, 5), columns=columns)
print(hier_df)
```

    cty          US                            JP          
    tenor         1         3         5         1         3
    0     -1.047668 -0.056759  1.000576  0.213877  0.277200
    1      2.460159  0.134850 -0.852783  1.358721  0.783409
    2     -0.555137 -0.075603  1.629682 -0.716262 -0.981589
    3     -0.135948  0.311016  0.310988 -0.741287 -0.024857
    


```python
#按照level进行groupby，除了此level的所有子层会进行合并。
print(hier_df.groupby(level='cty', axis=1).count())
```

    cty  JP  US
    0     2   3
    1     2   3
    2     2   3
    3     2   3
    


```python
print(hier_df.groupby(level="tenor",axis=1).count())
```

    tenor  1  3  5
    0      2  2  1
    1      2  2  1
    2      2  2  1
    3      2  2  1
    

## 2. 数据聚合：agg、transform、apply

groupBy的作用主要在于其为我们提供了一个包含分组信息的可迭代的groupBy对象。这就完成了数据分组的任务，然而，真正关键的部分才刚刚开始，我们需要对这些分组进行运算，不论是内置的运算还是我们自己编写的运算，在这里需要注意，运算传入的对象是group的ndarray，至于传出什么结果，一般而言，像是agg应用count、sum这类内置函数会生成标量，然后被整个成一张表，也就是说，每个group会返回一行数据。对于transform来说，每个group中的每行元素都会返回一个数据。对于apply来说，我们可以返回任意的矢量值，也就是说每个group不一定返回相同的长度数据，反正最后会被concat进行数据合并。

### 2.1 数据聚合过程

> groupby.mean()，这个mean是怎么工作的？这涉及到分组后的数据聚合，pandas定义了很多优化过的方法，比如sum、count、first等，但是我们也可以写自己的方法，使用agg/aggregate()进行调用。

quantile可以计算分位数。其本身是一个对于series作用的函数，在这里发生了什么？

groupby对象将结果进行了切片(piece)，每一个piece对应一个series，对各片调用了piece.quantile()，然后将分别生成的结果进行了组装。


```python
print(df)
grouped = df.groupby('key1')
for name,value in grouped["data1"]:
    print("\n",name,"\n",value)
grouped['data1'].quantile(0.9)
```

          data1     data2 key1 key2
    0  0.089452 -1.467732    a  one
    1 -0.296581  0.897673    a  two
    2 -0.887990  1.029688    b  one
    3 -0.535381 -1.179564    b  two
    4 -0.651637  0.749137    a  one
    
     a 
     0    0.089452
    1   -0.296581
    4   -0.651637
    Name: data1, dtype: float64
    
     b 
     2   -0.887990
    3   -0.535381
    Name: data1, dtype: float64

    key1
    a    0.012245
    b   -0.570642
    Name: data1, dtype: float64



也可以使用自己的agg方法，传入的是经过groupby分片的piece(Series),对series进行max和min的计算后返回结果。

在这里需要注意的是，data1和data2都有根据key1分组的两个不同piece，因此就有了四个piece，所以合并之后会生成2×2的表格


```python
def peak_to_peak(arr):
    #print("\nI am in peak, \nthe att is \n%s"%arr)
    return arr.max() - arr.min()
print(grouped.agg(peak_to_peak))
```

             data1     data2
    key1                    
    a     0.741089  2.365405
    b     0.352608  2.209252
    

```python
print(grouped.describe().T[:3])
```

    key1                a         b
    data1 count  3.000000  2.000000
          mean  -0.286255 -0.711685
          std    0.370652  0.249332
    

内置的一些函数比如sum、count、mean、median、std、var、min、max、prod、first、last，这些都是经过优化的方法，因此使用起来非常快速。而自己定义的函数速度则受限，因此尽量采用内置的函数进行聚合计算。


### 2.2 agg技术: 面向列的函数应用

> 开始编写自己的过程函数

```python
tips = pd.read_csv('tips.csv')
# Add tip percentage of total bill
tips['tip_pct'] = tips['tip'] / tips['total_bill']
print(tips[:6])
```

       total_bill   tip smoker  day    time  size   tip_pct
    0       16.99  1.01     No  Sun  Dinner     2  0.059447
    1       10.34  1.66     No  Sun  Dinner     3  0.160542
    2       21.01  3.50     No  Sun  Dinner     3  0.166587
    3       23.68  3.31     No  Sun  Dinner     2  0.139780
    4       24.59  3.61     No  Sun  Dinner     4  0.146808
    5       25.29  4.71     No  Sun  Dinner     4  0.186240
    
```python
grouped = tips.groupby(['day', 'smoker'])
```

```python
grouped_pct = grouped['tip_pct']
grouped_pct.agg('mean')
```

    day   smoker
    Fri   No        0.151650
          Yes       0.174783
    Sat   No        0.158048
          Yes       0.147906
    Sun   No        0.160113
          Yes       0.187250
    Thur  No        0.160298
          Yes       0.163863
    Name: tip_pct, dtype: float64


agg可以调用str表示的内置函数，或者直接一个函数。可以传递多个函数生成多个结果，放在一个列表中即可。

需要注意，这里和groupby技术中直接调用函数并call不同，采用agg可以使用我们自己的函数，并且能够一次进行多个函数的计算。


> 多函数计算

```python
print(grouped_pct.agg(['mean', 'std', peak_to_peak]))
```

                     mean       std  peak_to_peak
    day  smoker                                  
    Fri  No      0.151650  0.028123      0.067349
         Yes     0.174783  0.051293      0.159925
    Sat  No      0.158048  0.039767      0.235193
         Yes     0.147906  0.061375      0.290095
    Sun  No      0.160113  0.042347      0.193226
         Yes     0.187250  0.154134      0.644685
    Thur No      0.160298  0.038774      0.193350
         Yes     0.163863  0.039389      0.151240


> 函数别名设置

agg不仅可以调用多个函数生成结果表，还可以对每个函数进行名称的自定义。

```python
print(grouped_pct.agg([('MEAN', 'mean'), ('STD', np.std)]))
```

                     MEAN       STD
    day  smoker                    
    Fri  No      0.151650  0.028123
         Yes     0.174783  0.051293
    Sat  No      0.158048  0.039767
         Yes     0.147906  0.061375
    Sun  No      0.160113  0.042347
         Yes     0.187250  0.154134
    Thur No      0.160298  0.038774
         Yes     0.163863  0.039389
    

> 多层分组的多函数计算

再看一个更加复杂的例子，使用多个index进行group，对结果进行多个函数的计算，最后汇聚成表。


```python
functions = ['count', 'mean', 'max']
result = grouped['tip_pct', 'total_bill'].agg(functions)
print(type(result))
print(result)
```

    <class 'pandas.core.frame.DataFrame'>
                tip_pct                     total_bill                  
                  count      mean       max      count       mean    max
    day  smoker                                                         
    Fri  No           4  0.151650  0.187735          4  18.420000  22.75
         Yes         15  0.174783  0.263480         15  16.813333  40.17
    Sat  No          45  0.158048  0.291990         45  19.661778  48.33
         Yes         42  0.147906  0.325733         42  21.276667  50.81
    Sun  No          57  0.160113  0.252672         57  20.506667  48.17
         Yes         19  0.187250  0.710345         19  24.120000  45.35
    Thur No          45  0.160298  0.266312         45  17.113111  41.19
         Yes         17  0.163863  0.241255         17  19.190588  43.11
    

```python
print(result['tip_pct'])
```

                 count      mean       max
    day  smoker                           
    Fri  No          4  0.151650  0.187735
         Yes        15  0.174783  0.263480
    Sat  No         45  0.158048  0.291990
         Yes        42  0.147906  0.325733
    Sun  No         57  0.160113  0.252672
         Yes        19  0.187250  0.710345
    Thur No         45  0.160298  0.266312
         Yes        17  0.163863  0.241255
    

> 带有别名的多层分组多函数计算

```python
ftuples = [('Durchschnitt', 'mean'), ('Abweichung', np.var)]
print(grouped['tip_pct', 'total_bill'].agg(ftuples))
```

                     tip_pct              total_bill            
                Durchschnitt Abweichung Durchschnitt  Abweichung
    day  smoker                                                 
    Fri  No         0.151650   0.000791    18.420000   25.596333
         Yes        0.174783   0.002631    16.813333   82.562438
    Sat  No         0.158048   0.001581    19.661778   79.908965
         Yes        0.147906   0.003767    21.276667  101.387535
    Sun  No         0.160113   0.001793    20.506667   66.099980
         Yes        0.187250   0.023757    24.120000  109.046044
    Thur No         0.160298   0.001503    17.113111   59.625081
         Yes        0.163863   0.001551    19.190588   69.808518
    

> 多层分组非对称列索引多函数计算

agg技术不仅能够接受一个列表，还可以像group一样接受一个dict，dict的name为需要聚合的column name，dict的value为需要应用的函数。甚至你可以传递多个函数而不仅仅是一个函数，函数们放在列表中，作为二级index显示。


```python
print(grouped.sum())
print(grouped.agg({'tip' : np.max, 'size' : 'sum'}))
print(grouped.agg({'tip_pct' : ['min', 'max', 'mean', 'std'],
             'size' : 'sum'}))
```

                 total_bill     tip  size   tip_pct
    day  smoker                                    
    Fri  No           73.68   11.25     9  0.606602
         Yes         252.20   40.71    31  2.621746
    Sat  No          884.78  139.63   115  7.112145
         Yes         893.62  120.77   104  6.212055
    Sun  No         1168.88  180.57   167  9.126438
         Yes         458.28   66.82    49  3.557756
    Thur No          770.09  120.32   112  7.213414
         Yes         326.24   51.51    40  2.785676

                   tip  size
    day  smoker             
    Fri  No       3.50     9
         Yes      4.73    31
    Sat  No       9.00   115
         Yes     10.00   104
    Sun  No       6.00   167
         Yes      6.50    49
    Thur No       6.70   112
         Yes      5.00    40

                  tip_pct                               size
                      min       max      mean       std  sum
    day  smoker                                             
    Fri  No      0.120385  0.187735  0.151650  0.028123    9
         Yes     0.103555  0.263480  0.174783  0.051293   31
    Sat  No      0.056797  0.291990  0.158048  0.039767  115
         Yes     0.035638  0.325733  0.147906  0.061375  104
    Sun  No      0.059447  0.252672  0.160113  0.042347  167
         Yes     0.065660  0.710345  0.187250  0.154134   49
    Thur No      0.072961  0.266312  0.160298  0.038774  112
         Yes     0.090014  0.241255  0.163863  0.039389   40
    

> 没有索引的聚合形式


```python
print(tips.groupby(['day', 'smoker'], as_index=True).mean())
print(tips.groupby(['day', 'smoker'], as_index=False).mean())
```

                 total_bill       tip      size   tip_pct
    day  smoker                                          
    Fri  No       18.420000  2.812500  2.250000  0.151650
         Yes      16.813333  2.714000  2.066667  0.174783
    Sat  No       19.661778  3.102889  2.555556  0.158048
         Yes      21.276667  2.875476  2.476190  0.147906
    Sun  No       20.506667  3.167895  2.929825  0.160113
         Yes      24.120000  3.516842  2.578947  0.187250
    Thur No       17.113111  2.673778  2.488889  0.160298
         Yes      19.190588  3.030000  2.352941  0.163863
        day smoker  total_bill       tip      size   tip_pct
    0   Fri     No   18.420000  2.812500  2.250000  0.151650
    1   Fri    Yes   16.813333  2.714000  2.066667  0.174783
    2   Sat     No   19.661778  3.102889  2.555556  0.158048
    3   Sat    Yes   21.276667  2.875476  2.476190  0.147906
    4   Sun     No   20.506667  3.167895  2.929825  0.160113
    5   Sun    Yes   24.120000  3.516842  2.578947  0.187250
    6  Thur     No   17.113111  2.673778  2.488889  0.160298
    7  Thur    Yes   19.190588  3.030000  2.352941  0.163863
    

> 使用内置的函数进行agg

聚合是一种特殊的分组运算，其特点是将多维数据进行运算并且生成一个标量值。


```python
print(people)
print(people.groupby(["one","two","one","one","two"]).mean())
print(people.groupby(["one","two","one","one","two"]).agg("mean"))
```

                   a         b         c         d         e
    Joe     0.222491 -0.995911 -1.014604 -0.067544 -0.869915
    Steve   0.777247 -1.353332  0.821747 -1.104266  0.156773
    Wes    -0.080765       NaN       NaN  0.737536  0.132357
    Jim    -0.733980 -0.728372 -1.101054 -0.518711 -0.099602
    Travis -0.204770 -1.357502  1.421094  0.523682  1.917264
                a         b         c         d         e
    one -0.197418 -0.862142 -1.057829  0.050427 -0.279053
    two  0.286238 -1.355417  1.121420 -0.290292  1.037019
                a         b         c         d         e
    one -0.197418 -0.862142 -1.057829  0.050427 -0.279053
    two  0.286238 -1.355417  1.121420 -0.290292  1.037019
    

### 2.3 transform技术：不合并的agg

transform和agg不同的是，其保留了所有分组组内的值，不对聚合整体进行运算。但是其结果填充了对于组内运算的值，这看起来有些奇怪。造成这种情况的原因是，transform运行的函数生成了一个标量值，而这个标量值在组内被广播了。


```python
print(people.groupby(["one","two","one","one","two"]).transform("count"))
print(people.groupby(["one","two","one","one","two"]).agg("count")) #显然下面这个看起来更清晰一些。逻辑上来说。
```

              a    b    c    d    e
    Joe     3.0  2.0  2.0  3.0  3.0
    Steve   2.0  2.0  2.0  2.0  2.0
    Wes     3.0  2.0  2.0  3.0  3.0
    Jim     3.0  2.0  2.0  3.0  3.0
    Travis  2.0  2.0  2.0  2.0  2.0
         a  b  c  d  e
    one  3  2  2  3  3
    two  2  2  2  2  2
    


```python
def defmean(arr):
    #print("\n","The arr is\n",arr)
    return arr - arr.mean()
```


```python
res = people.groupby(["one","two","one","one","two"]).transform(defmean) 
#这个函数返回的是标量值，因此比上一个看起来有意义些。
print(res)
```

                   a         b         c         d         e
    Joe     0.419909 -0.133770  0.043225 -0.117971 -0.590861
    Steve   0.491009  0.002085 -0.299673 -0.813974 -0.880246
    Wes     0.116653       NaN       NaN  0.687109  0.411410
    Jim    -0.536562  0.133770 -0.043225 -0.569138  0.179451
    Travis -0.491009 -0.002085  0.299673  0.813974  0.880246
    


```python
print(res.groupby(["one","two","one","one","two"]).mean())
```

                    a             b             c             d             e
    one -3.700743e-17  5.551115e-17  1.110223e-16 -3.700743e-17 -1.850372e-17
    two -2.775558e-17  0.000000e+00  5.551115e-17  0.000000e+00 -1.110223e-16
    

transform和agg一样，其工作必须产生一个标量或者对应大小的数组，对于前者的标量，其会进行广播。但是，对于apply，就没有这种限制。


### 2.4 apply技术：动态自定义

apply技术可以传递pieces，然后进行函数运算后返回一个任意东西，经过concat方法被合并成结果，其使用限制取决于你的想象。

```python
print(tips[:5])
```

       total_bill   tip smoker  day    time  size   tip_pct
    0       16.99  1.01     No  Sun  Dinner     2  0.059447
    1       10.34  1.66     No  Sun  Dinner     3  0.160542
    2       21.01  3.50     No  Sun  Dinner     3  0.166587
    3       23.68  3.31     No  Sun  Dinner     2  0.139780
    4       24.59  3.61     No  Sun  Dinner     4  0.146808
    

```python
def top(df, n=5, column='tip_pct'):
    return df.sort_values(by=column)[-n:]
print(top(tips, n=6))
```

         total_bill   tip smoker  day    time  size   tip_pct
    109       14.31  4.00    Yes  Sat  Dinner     2  0.279525
    183       23.17  6.50    Yes  Sun  Dinner     4  0.280535
    232       11.61  3.39     No  Sat  Dinner     2  0.291990
    67         3.07  1.00    Yes  Sat  Dinner     1  0.325733
    178        9.60  4.00    Yes  Sun  Dinner     2  0.416667
    172        7.25  5.15    Yes  Sun  Dinner     2  0.710345
    


```python
print(tips.groupby('smoker').apply(top))
```

                total_bill   tip smoker   day    time  size   tip_pct
    smoker                                                           
    No     88        24.71  5.85     No  Thur   Lunch     2  0.236746
           185       20.69  5.00     No   Sun  Dinner     5  0.241663
           51        10.29  2.60     No   Sun  Dinner     2  0.252672
           149        7.51  2.00     No  Thur   Lunch     2  0.266312
           232       11.61  3.39     No   Sat  Dinner     2  0.291990
    Yes    109       14.31  4.00    Yes   Sat  Dinner     2  0.279525
           183       23.17  6.50    Yes   Sun  Dinner     4  0.280535
           67         3.07  1.00    Yes   Sat  Dinner     1  0.325733
           178        9.60  4.00    Yes   Sun  Dinner     2  0.416667
           172        7.25  5.15    Yes   Sun  Dinner     2  0.710345
    

> 禁用group_keys

对于groupby传入参数 `group_keys=False` 来禁用index列。需要区别于 `as_index` 参数，后者会生成一个num的index，而前者则完全不生成index


```python
print(tips.groupby('smoker',group_keys=False).apply(top)[:3])
```

         total_bill   tip smoker   day    time  size   tip_pct
    88        24.71  5.85     No  Thur   Lunch     2  0.236746
    185       20.69  5.00     No   Sun  Dinner     5  0.241663
    51        10.29  2.60     No   Sun  Dinner     2  0.252672
    


```python
print(tips.groupby('smoker',as_index=False).apply(top)[:3])
```

           total_bill   tip smoker   day    time  size   tip_pct
    0 88        24.71  5.85     No  Thur   Lunch     2  0.236746
      185       20.69  5.00     No   Sun  Dinner     5  0.241663
      51        10.29  2.60     No   Sun  Dinner     2  0.252672


> 函数的参数传递

函数的参数放在apply的第二个开始的参数中。


```python
def top(df, n=5, column='tip_pct'):
    return df.sort_values(by=column)[-n:]

print(tips.groupby(['smoker', 'day']).apply(top, n=1, column='total_bill'))
```

                     total_bill    tip smoker   day    time  size   tip_pct
    smoker day                                                             
    No     Fri  94        22.75   3.25     No   Fri  Dinner     2  0.142857
           Sat  212       48.33   9.00     No   Sat  Dinner     4  0.186220
           Sun  156       48.17   5.00     No   Sun  Dinner     6  0.103799
           Thur 142       41.19   5.00     No  Thur   Lunch     5  0.121389
    Yes    Fri  95        40.17   4.73    Yes   Fri  Dinner     4  0.117750
           Sat  170       50.81  10.00    Yes   Sat  Dinner     3  0.196812
           Sun  182       45.35   3.50    Yes   Sun  Dinner     3  0.077178
           Thur 197       43.11   5.00    Yes  Thur   Lunch     4  0.115982


```python
result = tips.groupby('smoker')['tip_pct'].describe()
result.unstack('smoker')
```


           smoker
    count  No        151.000000
           Yes        93.000000
    mean   No          0.159328
           Yes         0.163196
    std    No          0.039910
           Yes         0.085119
    min    No          0.056797
           Yes         0.035638
    25%    No          0.136906
           Yes         0.106771
    50%    No          0.155625
           Yes         0.153846
    75%    No          0.185014
           Yes         0.195059
    max    No          0.291990
           Yes         0.710345
    dtype: float64


f = lambda x: x.describe()
grouped.apply(f)


### 2.5 bucket技术:分位数分组计算

在上一章介绍了简单的(q)cut技术，此技术可以将数据放入等间距的桶中。在那个时候，我们仅仅做了划分，并且提到了使用哑变量对桶进行计数，这些都太简单以至于无法满足我们的需求。我们看到，cut后生成的对象可以看作是一个带有索引的list，其可以被group传入作为参数进行分组，这样的话，我们就可以使用上述的分组聚合中强大的函数对每个不同的等间距的桶进行函数操作了。传入labels=False即可只获得分位数的编号。

```python
frame = pd.DataFrame({'data1': np.random.randn(1000),
                      'data2': np.random.randn(1000)})
print(frame[:10])
quartiles = pd.cut(frame.data1, 4)
quartiles[:10]
```

          data1     data2
    0 -0.721918  1.834826
    1  0.416299 -1.262044
    2  0.889662  0.399180
    3  0.145300  1.346413
    4  0.568909  0.237202
    5  0.443284  0.100064
    6 -2.968871 -0.762229
    7 -0.423001 -2.982941
    8  0.526367  0.502630
    9 -0.244421 -0.109955
    
    0     (-1.364, 0.262]
    1      (0.262, 1.888]
    2      (0.262, 1.888]
    3     (-1.364, 0.262]
    4      (0.262, 1.888]
    5      (0.262, 1.888]
    6    (-2.996, -1.364]
    7     (-1.364, 0.262]
    8      (0.262, 1.888]
    9     (-1.364, 0.262]
    Name: data1, dtype: category
    Categories (4, interval[float64]): 
    [(-2.996, -1.364] < (-1.364, 0.262] < (0.262, 1.888] < (1.888, 3.514]]


```python
def get_stats(group):
    # 对每个桶进行处理 
    return {'min': group.min(), 'max': group.max(),
            'count': group.count(), 'mean': group.mean()}
grouped = frame.data2.groupby(quartiles)
print(grouped.agg(["count","max","min","mean"]))
print(grouped.apply(get_stats).unstack())
```

                      count       max       min      mean
    data1                                                
    (-2.996, -1.364]     75  2.304557 -2.458287  0.044986
    (-1.364, 0.262]     537  3.270650 -2.982941  0.056324
    (0.262, 1.888]      366  2.131040 -2.574885 -0.026318
    (1.888, 3.514]       22  2.153574 -2.133735 -0.192175

                      count       max      mean       min
    data1                                                
    (-2.996, -1.364]   75.0  2.304557  0.044986 -2.458287
    (-1.364, 0.262]   537.0  3.270650  0.056324 -2.982941
    (0.262, 1.888]    366.0  2.131040 -0.026318 -2.574885
    (1.888, 3.514]     22.0  2.153574 -0.192175 -2.133735
    

```python
# Return quantile numbers 使用labels=False来返回num而不是label。
#grouping = pd.qcut(frame.data1, 10)
grouping = pd.qcut(frame.data1, 10,labels=False)
grouped = frame.data2.groupby(grouping)
print(grouped.apply(get_stats).unstack())
```

           count       max      mean       min
    data1                                     
    0      100.0  2.304557 -0.005875 -2.458287
    1      100.0  2.480745  0.136586 -2.465328
    2      100.0  3.270650  0.045831 -2.692579
    3      100.0  2.026401 -0.078111 -2.982941
    4      100.0  2.569971  0.038701 -2.771897
    5      100.0  3.176304  0.201180 -1.867928
    6      100.0  2.003798 -0.142893 -2.574885
    7      100.0  1.998057 -0.056489 -1.964135
    8      100.0  2.009951  0.096824 -2.542099
    9      100.0  2.153574 -0.038157 -2.133735
    

## 3. 特殊的分组聚合：透视表和交叉表

透视表和交叉表在上一章节也有涉及，其可以看作是分组和堆叠的特殊形式（group & stark）。

### 3.1 透视表

```python
print(tips[:5])
```

       total_bill   tip smoker  day    time  size   tip_pct
    0       16.99  1.01     No  Sun  Dinner     2  0.059447
    1       10.34  1.66     No  Sun  Dinner     3  0.160542
    2       21.01  3.50     No  Sun  Dinner     3  0.166587
    3       23.68  3.31     No  Sun  Dinner     2  0.139780
    4       24.59  3.61     No  Sun  Dinner     4  0.146808
    


```python
print(tips.pivot_table(index=['day', 'smoker']))
```

                     size       tip   tip_pct  total_bill
    day  smoker                                          
    Fri  No      2.250000  2.812500  0.151650   18.420000
         Yes     2.066667  2.714000  0.174783   16.813333
    Sat  No      2.555556  3.102889  0.158048   19.661778
         Yes     2.476190  2.875476  0.147906   21.276667
    Sun  No      2.929825  3.167895  0.160113   20.506667
         Yes     2.578947  3.516842  0.187250   24.120000
    Thur No      2.488889  2.673778  0.160298   17.113111
         Yes     2.352941  3.030000  0.163863   19.190588
    


```python
print(tips.pivot_table(['tip_pct', 'size'], index=['time', 'day'],
                 columns='smoker'))
```
    

margins可以为所有的二级column进行合并，即不考虑二级column index。


```python
print(tips.pivot_table(['tip_pct', 'size'], index=['time', 'day'],
                 columns='smoker', margins=True))
```

                     size                       tip_pct                    
    smoker             No       Yes       All        No       Yes       All
    time   day                                                             
    Dinner Fri   2.000000  2.222222  2.166667  0.139622  0.165347  0.158916
           Sat   2.555556  2.476190  2.517241  0.158048  0.147906  0.153152
           Sun   2.929825  2.578947  2.842105  0.160113  0.187250  0.166897
           Thur  2.000000       NaN  2.000000  0.159744       NaN  0.159744
    Lunch  Fri   3.000000  1.833333  2.000000  0.187735  0.188937  0.188765
           Thur  2.500000  2.352941  2.459016  0.160311  0.163863  0.161301
    All          2.668874  2.408602  2.569672  0.159328  0.163196  0.160803
    

```python
print(tips.pivot_table('tip_pct', index=['time', 'smoker'], columns='day',
                 aggfunc=len, margins=True))
```

    day             Fri   Sat   Sun  Thur    All
    time   smoker                               
    Dinner No       3.0  45.0  57.0   1.0  106.0
           Yes      9.0  42.0  19.0   NaN   70.0
    Lunch  No       1.0   NaN   NaN  44.0   45.0
           Yes      6.0   NaN   NaN  17.0   23.0
    All            19.0  87.0  76.0  62.0  244.0
    

aggfunc参数可以传递任何能够传递给groupby的函数，比如mean、count，自己写的函数等。透视表本质上就是一种数据聚合。聚合的对象是那些被分在同一个表格（cell）中的数据。（在这个例子中，聚合的是某个时间的某个size的某个人的类型在某一天的所有数据）


```python
print(tips.pivot_table('tip_pct', index=['time', 'size', 'smoker'],
                 columns='day', aggfunc='mean', fill_value=0)[:5])
```

    day                      Fri       Sat       Sun      Thur
    time   size smoker                                        
    Dinner 1    No      0.000000  0.137931  0.000000  0.000000
                Yes     0.000000  0.325733  0.000000  0.000000
           2    No      0.139622  0.162705  0.168859  0.159744
                Yes     0.171297  0.148668  0.207893  0.000000
           3    No      0.000000  0.154661  0.152663  0.000000
    

### 3.2 交叉表

交叉表是一种用来计算分组频率的特殊透视表。

```python
from io import StringIO
data = """\
Sample  Nationality  Handedness
1   USA  Right-handed
2   Japan    Left-handed
3   USA  Right-handed
4   Japan    Right-handed
5   Japan    Left-handed
6   Japan    Right-handed
7   USA  Right-handed
8   USA  Left-handed
9   Japan    Right-handed
10  USA  Right-handed"""
data = pd.read_table(StringIO(data), sep='\s+')
```


```python
print(data)
```

       Sample Nationality    Handedness
    0       1         USA  Right-handed
    1       2       Japan   Left-handed
    2       3         USA  Right-handed
    3       4       Japan  Right-handed
    4       5       Japan   Left-handed
    5       6       Japan  Right-handed
    6       7         USA  Right-handed
    7       8         USA   Left-handed
    8       9       Japan  Right-handed
    9      10         USA  Right-handed
    


```python
print(pd.crosstab(data.Nationality, data.Handedness, margins=True))
```

    Handedness   Left-handed  Right-handed  All
    Nationality                                
    Japan                  2             3    5
    USA                    1             4    5
    All                    3             7   10
    


```python
print(pd.crosstab([tips.time, tips.day], tips.smoker, margins=True))
```

    smoker        No  Yes  All
    time   day                
    Dinner Fri     3    9   12
           Sat    45   42   87
           Sun    57   19   76
           Thur    1    0    1
    Lunch  Fri     1    6    7
           Thur   44   17   61
    All          151   93  244


## 4. 实例分析

### 4.1 使用分组进行NA值自定义填充

这个需求来自于illna()函数并不能很好的根据某些特殊情况填充不同的数值到行列中，比如根据分组的不同填充不同的数据，这个时候，使用分组和trans/apply就可以很好的解决这个问题。

```python
s = pd.Series(np.random.randn(6))
s[::2] = np.nan
s
s.fillna(s.mean())
```

    0   -0.952054
    1   -1.194706
    2   -0.952054
    3   -2.519394
    4   -0.952054
    5    0.857938
    dtype: float64


```python
states = ['Ohio', 'New York', 'Vermont', 'Florida',
          'Oregon', 'Nevada', 'California', 'Idaho']
group_key = ['East'] * 4 + ['West'] * 4
data = pd.Series(np.random.randn(8), index=states)
data
```

    Ohio          0.519011
    New York      1.471375
    Vermont       1.015578
    Florida       1.821681
    Oregon       -1.135199
    Nevada        0.079404
    California   -0.148342
    Idaho         1.829694
    dtype: float64


```python
data[['Vermont', 'Nevada', 'Idaho']] = np.nan
data
data.groupby(group_key).mean()
```

    East    1.270689
    West   -0.641770
    dtype: float64


```python
fill_mean = lambda g: g.fillna(g.mean())
data.groupby(group_key).apply(fill_mean)
```

    Ohio          0.519011
    New York      1.471375
    Vermont       1.270689
    Florida       1.821681
    Oregon       -1.135199
    Nevada       -0.641770
    California   -0.148342
    Idaho        -0.641770
    dtype: float64


```python
fill_values = {'East': 0.5, 'West': -1}
fill_func = lambda g: g.fillna(fill_values[g.name])
data.groupby(group_key).apply(fill_func)
```

    Ohio          0.519011
    New York      1.471375
    Vermont       0.500000
    Florida       1.821681
    Oregon       -1.135199
    Nevada       -1.000000
    California   -0.148342
    Idaho        -1.000000
    dtype: float64



### 4.2 分组加权平均数和相关系数的计算

在分析中，对于两列进行共同操作是一件常见的事情，比如下例反映了数据和其权重（第一个例子），想要求列的相关（第二个例子），对于第一个例子，我们需要将其进行合并求得加权数据，使用分组的apply可以很好的在进行运算的同时合并不同的分组。当然，使用map也可以。

```python
df = pd.DataFrame({'category': ['a', 'a', 'a', 'a',
                                'b', 'b', 'b', 'b'],
                   'data': np.random.randn(8),
                   'weights': np.random.rand(8)})
print(df)
```

      category      data   weights
    0        a -1.927664  0.676313
    1        a -0.577289  0.931676
    2        a  0.553938  0.763263
    3        a  1.470285  0.332282
    4        b  1.900069  0.476411
    5        b -0.696027  0.615567
    6        b  0.124953  0.799935
    7        b -0.702701  0.716592
    

np.average()可以计算平均数，里面有一个参数加权。


```python
grouped = df.groupby("category")
runc = lambda g:np.average(g["data"],weights=g["weights"])
result = grouped.apply(runc)
result
```

    category
    a   -0.344069
    b    0.028049
    dtype: float64



```python
close_px = pd.read_csv('stock_px_2.csv', parse_dates=True,
                       index_col=0)
close_px.info()
print(close_px[-4:])
```

    <class 'pandas.core.frame.DataFrame'>
    DatetimeIndex: 2214 entries, 2003-01-02 to 2011-10-14
    Data columns (total 4 columns):
    AAPL    2214 non-null float64
    MSFT    2214 non-null float64
    XOM     2214 non-null float64
    SPX     2214 non-null float64
    dtypes: float64(4)
    memory usage: 86.5 KB
                  AAPL   MSFT    XOM      SPX
    2011-10-11  400.29  27.00  76.27  1195.54
    2011-10-12  402.19  26.96  77.16  1207.25
    2011-10-13  408.43  27.18  76.37  1203.66
    2011-10-14  422.00  27.27  78.11  1224.58
    

```python
spx_corr = lambda x: x.corrwith(x['SPX'])
```


```python
rets = close_px.pct_change().dropna()
```


```python
get_year = lambda x: x.year
by_year = rets.groupby(get_year)
print(by_year.apply(spx_corr))
```
              AAPL      MSFT       XOM  SPX
    2003  0.541124  0.745174  0.661265  1.0
    2004  0.374283  0.588531  0.557742  1.0
    2005  0.467540  0.562374  0.631010  1.0
    2006  0.428267  0.406126  0.518514  1.0
    2007  0.508118  0.658770  0.786264  1.0
    2008  0.681434  0.804626  0.828303  1.0
    2009  0.707103  0.654902  0.797921  1.0
    2010  0.710105  0.730118  0.839057  1.0
    2011  0.691931  0.800996  0.859975  1.0
    


```python
by_year.apply(lambda g: g['AAPL'].corr(g['MSFT']))
```

    2003    0.480868
    2004    0.259024
    2005    0.300093
    2006    0.161735
    2007    0.417738
    2008    0.611901
    2009    0.432738
    2010    0.571946
    2011    0.581987
    dtype: float64


### 4.3 分组线性回归

使用statsmodels中的ols线性回归函数进行分组的相关计算。可以很方便的对基于日期的数据进行合并统计。

```python
import statsmodels.api as sm
def regress(data, yvar, xvars):
    Y = data[yvar]
    X = data[xvars]
    X['intercept'] = 1.
    result = sm.OLS(Y, X).fit()
    return result.params
```
    

```python
print(by_year.apply(regress, 'AAPL', ['SPX']))
```

               SPX  intercept
    2003  1.195406   0.000710
    2004  1.363463   0.004201
    2005  1.766415   0.003246
    2006  1.645496   0.000080
    2007  1.198761   0.003438
    2008  0.968016  -0.001110
    2009  0.879103   0.002954
    2010  1.052608   0.001261
    2011  0.806605   0.001514
    

## 5 示例：美国联邦选举数据分析

本数据库包含2008年美国大选的数据信息，数据大约150万条，包含投票人、职业、捐赠金额等。


```python
fec = pd.read_csv("P00000001-ALL.csv",low_memory=False)
print(fec[:5])
```

         cmte_id    cand_id             cand_nm           contbr_nm  \
    0  C00410118  P20002978  Bachmann, Michelle     HARVEY, WILLIAM   
    1  C00410118  P20002978  Bachmann, Michelle     HARVEY, WILLIAM   
    2  C00410118  P20002978  Bachmann, Michelle       SMITH, LANIER   
    3  C00410118  P20002978  Bachmann, Michelle    BLEVINS, DARONDA   
    4  C00410118  P20002978  Bachmann, Michelle  WARDENBURG, HAROLD   
    
              contbr_city contbr_st contbr_zip        contbr_employer  \
    0              MOBILE        AL  366010290                RETIRED   
    1              MOBILE        AL  366010290                RETIRED   
    2              LANETT        AL  368633403  INFORMATION REQUESTED   
    3             PIGGOTT        AR  724548253                   NONE   
    4  HOT SPRINGS NATION        AR  719016467                   NONE   
    
           contbr_occupation  contb_receipt_amt contb_receipt_dt receipt_desc  \
    0                RETIRED              250.0        20-JUN-11          NaN   
    1                RETIRED               50.0        23-JUN-11          NaN   
    2  INFORMATION REQUESTED              250.0        05-JUL-11          NaN   
    3                RETIRED              250.0        01-AUG-11          NaN   
    4                RETIRED              300.0        20-JUN-11          NaN   
    
      memo_cd memo_text form_tp  file_num  
    0     NaN       NaN   SA17A    736166  
    1     NaN       NaN   SA17A    736166  
    2     NaN       NaN   SA17A    749073  
    3     NaN       NaN   SA17A    749073  
    4     NaN       NaN   SA17A    736166  
    

随便找一个数据，可以看到有以下字段：

```python
fec.iloc[12000]
```

    cmte_id                            C00431171
    cand_id                            P80003353
    cand_nm                         Romney, Mitt
    contbr_nm            FELSENTHAL, JUDITH MRS.
    contbr_city                    BEVERLY HILLS
    contbr_st                                 CA
    contbr_zip                         902105514
    contbr_employer                    HOMEMAKER
    contbr_occupation                  HOMEMAKER
    contb_receipt_amt                       1000
    contb_receipt_dt                   05-DEC-11
    receipt_desc                             NaN
    memo_cd                                  NaN
    memo_text                                NaN
    form_tp                                SA17A
    file_num                              771927
    Name: 12000, dtype: object


首先我们要对候选人分析，使用unique找到所有的候选人信息。

```python
unique_cands = fec.cand_nm.unique()
unique_cands
```

    array(['Bachmann, Michelle', 'Romney, Mitt', 'Obama, Barack',
           "Roemer, Charles E. 'Buddy' III", 'Pawlenty, Timothy',
           'Johnson, Gary Earl', 'Paul, Ron', 'Santorum, Rick', 'Cain, Herman',
           'Gingrich, Newt', 'McCotter, Thaddeus G', 'Huntsman, Jon',
           'Perry, Rick'], dtype=object)


接着我们发现候选人没有党籍信息，所以这里使用map来映射，之后填充到fec的新列。

```python
cands_partymap = {"Bachmann, Michelle":"Republican",
                 'Romney, Mitt':"Republican", 
                  'Obama, Barack':"Democrat",
       "Roemer, Charles E. 'Buddy' III":"Republican",
                  'Pawlenty, Timothy':"Republican",
       'Johnson, Gary Earl':"Republican",
                  'Paul, Ron':"Republican",
                  'Santorum, Rick':"Republican",
       'Cain, Herman':"Republican",
                  'Gingrich, Newt':"Republican",
                  'McCotter, Thaddeus G':"Republican",
       'Huntsman, Jon':"Republican", 
                  'Perry, Rick':"Republican"}
```


```python
fec.cand_nm[123456:123480].map(cands_partymap)
fec["party"] = fec.cand_nm.map(cands_partymap)
```

可以看看分组统计信息：

```python
fec.party.value_counts()
```


    Democrat      593746
    Republican    407985
    Name: party, dtype: int64


由于Obama和Romney占有很大份额，因此搞一个子集等下单独处理，这里用到了isin，用来过滤数据。可以看到，对于一个新数据集，values_counts、isin和unique这三剑客很有用。接下来就是dropna、fillna、bool-ndarray筛选和填充的时候了，比如这个，筛选出收入，只保留支出。

```python
fec = fec[fec.contb_receipt_amt > 0]
```


```python
fec_choose = fec[fec.cand_nm.isin(["Obama, Barack","Romney, Mitt"])]
fec_choose.count()
```


    cmte_id              694282
    cand_id              694282
    cand_nm              694282
    contbr_nm            694282
    contbr_city          694275
    contbr_st            694278
    contbr_zip           694234
    contbr_employer      693607
    contbr_occupation    693524
    contb_receipt_amt    694282
    contb_receipt_dt     694282
    receipt_desc           2345
    memo_cd               87387
    memo_text             90672
    form_tp              694282
    file_num             694282
    party                694282
    dtype: int64

这里我们想要统计不同党派的捐献职业。

常见的职业有：

```python
fec.contb_receipt_amt.dropna(inplace=True)
```

接下来，我们要处理一下职业问题，因为职业里有很多重复，找出最多的职业，之后使用map进行映射。这里需要注意，如果直接map，那么所有不在dict中的职业信息会丢失，因此这里设置了一个函数，然后使用dict.getr(x,x),这样如果没有的话，会填充默认值：

    get(key[, default])
    Return the value for key if key is in the dictionary, else default. 
    If default is not given, it defaults to None, 
    so that this method never raises a KeyError.

```python
occ_mapping = {
    "INFORMATION REQUESTED":"NOT PROVIDED",
    "INFORMATION REQUESTED PER BEST EFFORTS":"NOT PROVIDED",
    "C.E.O.":"CEO",
    "C.F.O.":"CFO"
}
emp_mapping = {
    "INFORMATION REQUESTED":"NOT PROVIDED",
    "INFORMATION REQUESTED PER BEST EFFORTS":"NOT PROVIDED",
    "SELF EMPLOYED":"SELF-EMPLOYED",
    "SELF":"SELF-EMPLOYED"
}
f = lambda x:emp_mapping.get(x,x)
fec.contbr_employer=fec.contbr_employer.map(f)
f2 = lambda y:occ_mapping.get(y,y)
fec.contbr_occupation=fec.contbr_occupation.map(f2)
```


```python
by_occ = fec.pivot_table("contb_receipt_amt", 
index="contbr_occupation",columns="party",aggfunc="sum")
by_occ_max = by_occ[by_occ.sum(1)>2000000]
```


```python
print(by_occ_max)
```

    party                 Democrat    Republican
    contbr_occupation                           
    ATTORNEY           11141982.97  7.477194e+06
    CEO                 2074974.79  4.211041e+06
    CONSULTANT          2459912.71  2.544725e+06
    ENGINEER             951525.55  1.818374e+06
    EXECUTIVE           1355161.05  4.138850e+06
    HOMEMAKER           4248875.80  1.363428e+07
    INVESTOR             884133.00  2.431769e+06
    LAWYER              3160478.87  3.912243e+05
    MANAGER              762883.22  1.444532e+06
    NOT PROVIDED        4866973.96  2.023715e+07
    OWNER               1001567.36  2.408287e+06
    PHYSICIAN           3735124.94  3.594320e+06
    PRESIDENT           1878509.95  4.720924e+06
    PROFESSOR           2165071.08  2.967027e+05
    REAL ESTATE          528902.09  1.625902e+06
    RETIRED            25305116.38  2.356124e+07
    SELF-EMPLOYED        672393.40  1.640253e+06
    

```python
%matplotlib inline
by_occ_max.plot(kind="barh")
```

    <matplotlib.axes._subplots.AxesSubplot at 0x1b8175fb3c8>


![png](/media/output_144_1.png)

接下来，我们要统计两个候选人和投资人职业的关系。使用groupby技术，这里定义了一个函数用来将同一职业的捐赠金额排序、合并。最后找出了前7位职业。

```python
def get_group_amont(group,key,n=5):
    totals = group.groupby(key)["contb_receipt_amt"].sum()
    return totals.sort_values(ascending=False).iloc[:n]
grouped = fec_choose.groupby("cand_nm")
grouped.apply(get_group_amont,"contbr_occupation",n=7)
```



    cand_nm        contbr_occupation                     
    Obama, Barack  RETIRED                                   25305116.38
                   ATTORNEY                                  11141982.97
                   INFORMATION REQUESTED                      4866973.96
                   HOMEMAKER                                  4248875.80
                   PHYSICIAN                                  3735124.94
                   LAWYER                                     3160478.87
                   CONSULTANT                                 2459912.71
    Romney, Mitt   RETIRED                                   11508473.59
                   INFORMATION REQUESTED PER BEST EFFORTS    11396894.84
                   HOMEMAKER                                  8147446.22
                   ATTORNEY                                   5364718.82
                   PRESIDENT                                  2491244.89
                   EXECUTIVE                                  2300947.03
                   C.E.O.                                     1968386.11
    Name: contb_receipt_amt, dtype: float64




```python
grouped.apply(get_group_amont,"contbr_employer",n=10)
```




    cand_nm        contbr_employer                       
    Obama, Barack  RETIRED                                   22694358.85
                   SELF-EMPLOYED                             17080985.96
                   NOT EMPLOYED                               8586308.70
                   INFORMATION REQUESTED                      5053480.37
                   HOMEMAKER                                  2605408.54
                   SELF                                       1076531.20
                   SELF EMPLOYED                               469290.00
                   STUDENT                                     318831.45
                   VOLUNTEER                                   257104.00
                   MICROSOFT                                   215585.36
    Romney, Mitt   INFORMATION REQUESTED PER BEST EFFORTS    12059527.24
                   RETIRED                                   11506225.71
                   HOMEMAKER                                  8147196.22
                   SELF-EMPLOYED                              7409860.98
                   STUDENT                                     496490.94
                   CREDIT SUISSE                               281150.00
                   MORGAN STANLEY                              267266.00
                   GOLDMAN SACH & CO.                          238250.00
                   BARCLAYS CAPITAL                            162750.00
                   H.I.G. CAPITAL                              139500.00
    Name: contb_receipt_amt, dtype: float64


最后，我们想要了解捐献金额的区别，看看富人和穷人对这两个候选人怎么看。使用bucket技术进行面元划分。

```python
bins = np.array([0,1,10,100,1000,10000,100000,1000000,10000000])
labels = pd.cut(fec_choose.contb_receipt_amt,bins)
labels[:3]
```


    411      (10, 100]
    412    (100, 1000]
    413    (100, 1000]
    Name: contb_receipt_amt, dtype: category
    Categories (8, interval[int64]): 
    [(0, 1] < (1, 10] < (10, 100] < (100, 1000] < (1000, 10000] < (10000, 100000] < (100000, 1000000] < (1000000, 10000000]]




```python
grouped2 = fec_choose.groupby(["cand_nm",labels])
print(grouped2["contb_receipt_amt"].sum().unstack(level=0))
```

    cand_nm              Obama, Barack  Romney, Mitt
    contb_receipt_amt                               
    (0, 1]                      318.24         77.00
    (1, 10]                  337267.62      29819.66
    (10, 100]              20288981.41    1987783.76
    (100, 1000]            54798531.46   22363381.69
    (1000, 10000]          51753705.67   63942145.42
    (10000, 100000]           59100.00      12700.00
    (100000, 1000000]       1490683.08           NaN
    (1000000, 10000000]     7148839.76           NaN
    


```python
bucket_sums = grouped2.size().unstack(level=0)
print(bucket_sums)
```

    cand_nm              Obama, Barack  Romney, Mitt
    contb_receipt_amt                               
    (0, 1]                       493.0          77.0
    (1, 10]                    40070.0        3681.0
    (10, 100]                 372280.0       31853.0
    (100, 1000]               153991.0       43357.0
    (1000, 10000]              22284.0       26186.0
    (10000, 100000]                2.0           1.0
    (100000, 1000000]              3.0           NaN
    (1000000, 10000000]            4.0           NaN
    


```python
normed_sum = bucket_sums.div(bucket_sums.sum(axis=1),axis=0)
print(normed_sum)
```

    cand_nm              Obama, Barack  Romney, Mitt
    contb_receipt_amt                               
    (0, 1]                    0.864912      0.135088
    (1, 10]                   0.915865      0.084135
    (10, 100]                 0.921182      0.078818
    (100, 1000]               0.780302      0.219698
    (1000, 10000]             0.459748      0.540252
    (10000, 100000]           0.666667      0.333333
    (100000, 1000000]         1.000000           NaN
    (1000000, 10000000]       1.000000           NaN
    


```python
normed_sum[:-2].plot(kind="barh",stacked=True)
```




    <matplotlib.axes._subplots.AxesSubplot at 0x1b81600c588>




![png](/media/output_151_1.png)



——————————————————————————————————

> 更新日志

2018-03-21 阅读《利用Python进行数据分析》并整理笔记。

2018-03-25 修改了部分目录，阅读《Python数据分析实战》并完善笔记。

