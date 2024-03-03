---
title: "Markdown基础语法"
date: 2023-02-01T18:10:41+08:00
categories:
- Markdown
tags:
- Markdown
draft: false
---

## 标题
| Markdown        | HTML                | Rendered Output |
| --------------  | ------------------- | --------------- |
| # 标题一         | \<h1\>标题一\<\/h1\> | <h1>标题一</h1>  |
| ## 标题二        | \<h2\>标题二\<\/h2\> | <h2>标题二</h2>  |
| ### 标题三       | \<h3\>标题三\<\/h3\> | <h3>标题三</h3>  |
| #### 标题四      | \<h4\>标题四\<\/h4\> | <h4>标题四</h4>  |
| ##### 标题五     | \<h5\>标题五\<\/h5\> | <h5>标题五</h5>  |
| ###### 标题六    | \<h6\>标题六\<\/h6\> | <h6>标题六</h6>  |

## 段落
<pre>
Don't put tabs or spaces in front of your paragraphs.
 
Keep lines left-aligned like this.
</pre>

## 换行
<pre>
这是第一行.  
这是第二行
</pre>

这是第一行.  
这是第二行

## 引用

#### 单行引用
Markdown：
<pre>
> Dorothy followed her through many of the beautiful rooms in her castle.
</pre>

HTML显示效果：
> Dorothy followed her through many of the beautiful rooms in her castle.


#### 多行引用
Markdown：
<pre>
> Dorothy followed her through many of the beautiful rooms in her castle.
>
> Dorothy followed her through many of the beautiful rooms in her castle.
</pre>

HTML显示效果：
> Dorothy followed her through many of the beautiful rooms in her castle.
>
> Dorothy followed her through many of the beautiful rooms in her castle.


#### 嵌套引用
Markdown：
<pre>
> Dorothy followed her through many of the beautiful rooms in her castle.
>
>> Dorothy followed her through many of the beautiful rooms in her castle.
</pre>

HTML显示效果：
> Dorothy followed her through many of the beautiful rooms in her castle.
>
>> Dorothy followed her through many of the beautiful rooms in her castle.


#### 包含其它元素的引用
Markdown：
<pre>
> #### The quarterly results look great!
>
> - Revenue was off the chart.
> - Profits were higher than ever.
>
>  *Everything* is going according to **plan**.
</pre>

HTML显示效果：
> #### The quarterly results look great!
>
> - Revenue was off the chart.
> - Profits were higher than ever.
>
>  *Everything* is going according to **plan**.


## 列表


## 代码块

#### 普通代码块
Markdown：
<pre>
```cpp
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, Hugo");
    return 0;
}
```
</pre>

HTML显示效果：
```cpp
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, Hugo");
    return 0;
}
```


#### 高亮代码块
Markdown：
<pre>
```cpp {linenos=table,linenostart=10,hl_lines=[3,"5-7"]}
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, Hugo");
    printf("This is highlight line one");
    printf("This is highlight line two");
    printf("This is highlight line there");
    return 0;
}
```
</pre>

HTML显示效果：
```cpp {linenos=table,linenostart=10,hl_lines=[3,"5-7"]}
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, Hugo");
    printf("This is highlight line one");
    printf("This is highlight line two");
    printf("This is highlight line there");
    return 0;
}
```


~~doubletilde~~

<br/>
<br/>

**参考文档：**

[Markdown Basic Syntax](https://www.markdownguide.org/basic-syntax)