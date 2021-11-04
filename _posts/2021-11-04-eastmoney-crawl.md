---
layout: post
title: 【动态请求】：用代码帮我们来“点击加载更多”
categories:
  - 爬虫
  - 财经文本

---

> 这篇博客解答了[上一篇博客](https://qj4chen.github.io/2021/11/03/regular-expression/)遗留的问题, 介绍了在浏览器中调试类似的数据加载网页，并讲解了如何用Python的requests库来编写代码，不断地获取网站后端返回给我们的数据。
>
> 本教程用到的技术栈：
>
> * Python
>   * 文本读写
>   * JSON数据的处理
>   * 异常处理
>   * 模拟浏览器发送请求
> * 网页
>   * `F12`调试发现网页规律

本博客由`果然科技有限公司独家赞助`.

# 网页调试

## 请求

## fetch/XHR

## header

## 表单数据 form data



# 简单学习requests库

# 全部代码



```python
import os
import requests

url = r'http://guba.eastmoney.com/interface/GetData.aspx'


def get_question_answer_pairs(code):
    if not os.path.exists('./output'):
        os.makedirs('./output')

    with open(f'./output/{code}.txt', mode='w') as f:
        f.write('')
    for i in range(1, 20):
        data = dict(param=f'code={code}&ps=15&p={i}&qatype=1', path='question/api/Info/Search', env=2)

        header = {'Referer': f'http://guba.eastmoney.com/qa/qa_search.aspx?company={code}&keyword=&questioner=&qatype=1',
                  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36 Edg/95.0.1020.40)'}

        response = requests.post(url=url, headers=header, data=data)

        response_json = response.json()
        with open(f'./output/{code}.txt', mode='a') as f:
            for record in response_json['re']:
                try:
                    f.write('发表时间(publish time): ' + record['post_publish_time'] + '\t' + '展示时间(display time): ' + record['post_display_time'] + '\n')
                    f.write(record['user_nickname'] + ': ' + record['ask_question'] + '\n')
                    f.write(record['ask_answer'] + '\n'*2)
                except IndexError:
                    pass


if __name__ == '__main__':
    code = 600519
    get_question_answer_pairs(code=code)
```

