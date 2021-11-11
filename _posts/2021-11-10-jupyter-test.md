---
layout: post
title: 测试 jupyter notebook 是否能正常显示
typora-root-url: ..
categories:
  - data science
---
```python
import pandas as pd

from pandas.util.testing import makeDataFrame
```


```python
df = makeDataFrame()
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>A</th>
      <th>B</th>
      <th>C</th>
      <th>D</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>btGRTrh9LU</th>
      <td>-0.742145</td>
      <td>-0.893328</td>
      <td>-1.304956</td>
      <td>1.034842</td>
    </tr>
    <tr>
      <th>BV2CsQIPdw</th>
      <td>1.824368</td>
      <td>-0.386302</td>
      <td>0.678756</td>
      <td>0.404220</td>
    </tr>
    <tr>
      <th>jEIGok5qYf</th>
      <td>-0.267949</td>
      <td>0.584441</td>
      <td>-0.744999</td>
      <td>1.339014</td>
    </tr>
    <tr>
      <th>bFacdWMgsD</th>
      <td>-1.010524</td>
      <td>-1.649774</td>
      <td>-0.754501</td>
      <td>-0.684241</td>
    </tr>
    <tr>
      <th>soAEDHnanD</th>
      <td>0.269725</td>
      <td>-0.083877</td>
      <td>0.602573</td>
      <td>-0.896465</td>
    </tr>
    <tr>
      <th>U7EVhbcWNZ</th>
      <td>0.455026</td>
      <td>1.387828</td>
      <td>-2.045870</td>
      <td>-0.961229</td>
    </tr>
    <tr>
      <th>Ug84kJ8EEu</th>
      <td>2.225741</td>
      <td>0.447942</td>
      <td>1.604812</td>
      <td>0.049973</td>
    </tr>
    <tr>
      <th>uVFU0kqoXp</th>
      <td>-1.182075</td>
      <td>-0.212537</td>
      <td>0.676468</td>
      <td>-0.099554</td>
    </tr>
    <tr>
      <th>BERsx5D6rd</th>
      <td>-0.967468</td>
      <td>-1.494943</td>
      <td>1.130980</td>
      <td>-2.050188</td>
    </tr>
    <tr>
      <th>FCfT0fjoSq</th>
      <td>-0.403244</td>
      <td>-1.646424</td>
      <td>2.097804</td>
      <td>-1.564483</td>
    </tr>
    <tr>
      <th>q0I0xD3vie</th>
      <td>-0.049393</td>
      <td>0.146242</td>
      <td>0.521698</td>
      <td>0.846845</td>
    </tr>
    <tr>
      <th>ppwOCxqTl0</th>
      <td>0.067105</td>
      <td>0.432056</td>
      <td>-0.730543</td>
      <td>0.490270</td>
    </tr>
    <tr>
      <th>z0RGWHIYIb</th>
      <td>-0.262235</td>
      <td>-0.999249</td>
      <td>-0.733476</td>
      <td>0.567824</td>
    </tr>
    <tr>
      <th>YHVIsWEgQb</th>
      <td>0.655289</td>
      <td>1.298319</td>
      <td>-0.939714</td>
      <td>-0.623303</td>
    </tr>
    <tr>
      <th>Ip4bikgZbZ</th>
      <td>-0.262531</td>
      <td>-0.033943</td>
      <td>1.294955</td>
      <td>-0.490842</td>
    </tr>
    <tr>
      <th>xJZeCSG96a</th>
      <td>-0.119527</td>
      <td>0.505828</td>
      <td>0.510533</td>
      <td>-1.566203</td>
    </tr>
    <tr>
      <th>jCJQ3aPSO1</th>
      <td>-1.332722</td>
      <td>-0.094682</td>
      <td>-0.188290</td>
      <td>-1.749978</td>
    </tr>
    <tr>
      <th>rc7nMgpMjp</th>
      <td>-0.088242</td>
      <td>1.217513</td>
      <td>0.170726</td>
      <td>1.032927</td>
    </tr>
    <tr>
      <th>PlM4c1j08c</th>
      <td>-0.316564</td>
      <td>-0.303150</td>
      <td>1.853104</td>
      <td>0.161671</td>
    </tr>
    <tr>
      <th>d5yxJs1zl6</th>
      <td>1.135084</td>
      <td>-1.554537</td>
      <td>0.466070</td>
      <td>0.382947</td>
    </tr>
    <tr>
      <th>vGUAcPGaK5</th>
      <td>-0.073617</td>
      <td>0.337186</td>
      <td>-0.168406</td>
      <td>-1.041977</td>
    </tr>
    <tr>
      <th>OnV8UXwVr4</th>
      <td>0.159056</td>
      <td>0.786618</td>
      <td>1.259272</td>
      <td>-0.881401</td>
    </tr>
    <tr>
      <th>6cyGD5lb18</th>
      <td>0.039900</td>
      <td>0.041941</td>
      <td>0.706655</td>
      <td>-0.917063</td>
    </tr>
    <tr>
      <th>0lLebAGL3k</th>
      <td>0.266503</td>
      <td>0.334693</td>
      <td>1.129501</td>
      <td>0.248132</td>
    </tr>
    <tr>
      <th>iD7o5BWJAS</th>
      <td>1.971463</td>
      <td>1.616939</td>
      <td>0.788988</td>
      <td>0.516291</td>
    </tr>
    <tr>
      <th>M4A3YjbjLI</th>
      <td>-0.107087</td>
      <td>-1.496449</td>
      <td>-0.264386</td>
      <td>0.946963</td>
    </tr>
    <tr>
      <th>q3s1p2wQ88</th>
      <td>-0.459087</td>
      <td>1.033434</td>
      <td>-0.082343</td>
      <td>-0.598603</td>
    </tr>
    <tr>
      <th>6deAxa0QVz</th>
      <td>0.687213</td>
      <td>-0.417025</td>
      <td>0.858427</td>
      <td>1.399216</td>
    </tr>
    <tr>
      <th>arlIugFbzB</th>
      <td>-0.836555</td>
      <td>-2.293708</td>
      <td>-0.958141</td>
      <td>-0.852751</td>
    </tr>
    <tr>
      <th>I6ZGEiHAtw</th>
      <td>-0.599726</td>
      <td>0.117627</td>
      <td>-1.063599</td>
      <td>0.654528</td>
    </tr>
  </tbody>
</table>
</div>




```python
import matplotlib.pylab as plt

df.plot()
```




    <AxesSubplot:>




![png]({{"/img/test_files/test_2_1.png"}})
​    

