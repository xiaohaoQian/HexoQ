title: Markdown 语法
author: 钱晓豪
tags:
  - markdown
categories:
  - markdown
  - 基本语法
date: 2019-11-05 10:28:00
---
本文总结了Markdown的常用语法。  
Markdown 是一种轻量级标记语言，采用纯文本格式编写文档，文档后缀为 .md, .markdown。  
Markdown 编写的文档可以导出 HTML 、Word、图像、PDF、Epub 等多种格式的文档。
***
<!--more-->

#### 标题
* **语法：**以`#`开始空格后加入标题名.
* **范围：**`#`号可表示 1-6 级标题，一级标题对应一个`#`号，二级标题对应两个`#`号，以此类推.
```
  # 标题1
  ## 标题2
  ### 标题3
  #### 标题4
  ##### 标题5
  ###### 标题6
 ```
效果如下图
![标题](/images/posts/20191105Markdown/title.gif "标题")
***

#### 列表
* **语法：**无序列表使用`*`或`+`或`-`作为列表标记.

##### 单一列表

```
	* 第一项
	* 第二项
	* 第三项

	+ 第一项
	+ 第二项
	+ 第三项

	- 第一项
	- 第二项
	- 第三项
```
效果如下图
![列表1](/images/posts/20191105Markdown/list_1.jpg "列表")

##### 列表嵌套
```
  1. 第一项:
      - 第一项嵌套的第一个元素
      - 第一项嵌套的第二个元素
  2. 第二项：
     - 第二项嵌套的第一个元素
     - 第二项嵌套的第二个元素
 ```
效果如下图
![列表2](/images/posts/20191105Markdown/list_2.jpg "列表")
***

#### 段落
* **语法：**使用两个以上空格加上回车，或在段落后使用一个空行来表示重新开始一个段落。  

效果如下图
![段落1](/images/posts/20191105Markdown/paragraph_1.png)
![段落2](/images/posts/20191105Markdown/paragraph_2.png)
***

#### 字体
* **语法：**`*`和`_`斜体文本；`**`和`__`粗体文本；`***`和`___`粗斜体文本
```
  *斜体文本*
  _斜体文本_
  **粗体文本**
  __粗体文本__
  ***粗斜体文本***
  ___粗斜体文本___
```
效果如下图
![字体](/images/posts/20191105Markdown/font.gif)
***

#### 分割线
* **语法：**一行中用三个以上的`*`或`-`或`_`底线来建立一个分隔线，行内不能有其他东西。可以在星号或是减号中间插入空格.
```
  ***
  * * *
  *****
  - - -
  ----------
```
效果如下图
![分割线](/images/posts/20191105Markdown/split.png)
***

#### 删除线
* **语法：**在文字的两端加上两个波浪线 ```~~```.  
```
  RUNOOB.COM
  GOOGLE.COM
  ~~BAIDU.COM~~
```
效果如下图
![删除线](/images/posts/20191105Markdown/delete.png)
***

#### 下划线
* **语法：**在文字的两端加上`<u>`和`</u>`. 
```
  <u>带下划线文本</u>
```
效果如下图
![下划线](/images/posts/20191105Markdown/underline.png)
***

#### 脚注
* **语法：**在`[^ ]`空格处填入脚注名称.
```
  创建脚注格式类似这样 [^RUNOOB]。
  [^RUNOOB]: 菜鸟教程 -- 学的不仅是技术，更是梦想！！！
```
效果如下图
![下划线](/images/posts/20191105Markdown/footnote.gif)
***

#### 区块引用
* **语法：**在段落开头使用`>`符号，后面紧跟空格符号.

##### 单一引用
```
  > 区块引用
  > 菜鸟教程
  > 学的不仅是技术更是梦想
```
效果如下图
![下划线](/images/posts/20191105Markdown/cite_1.jpg)
##### 嵌套引用
```
  > 最外层
  > > 第一层嵌套
  > > > 第二层嵌套
```
效果如下图
![下划线](/images/posts/20191105Markdown/cite_2.jpg)
***

#### 代码
* **语法：**用反引号把代码包起来,代码区块使用 4 个空格或者一个制表符.

##### 段落中代码
```
	`printf()` 函数
```
效果如下图
![下划线](/images/posts/20191105Markdown/code_1.jpg)

##### 代码块
	```javascript
	$(document).ready(function () {
    	alert('RUNOOB');
	});
	```
效果如下图
![下划线](/images/posts/20191105Markdown/code_2.jpg)
***

#### 链接
* **语法：**[链接名称]+(链接地址).

##### 一般链接
```
	这是一个链接 [菜鸟教程](https://www.runoob.com)
   
```
或者
 ```
	<https://www.runoob.com>
```
效果如下图
![链接1](/images/posts/20191105Markdown/link_1.jpg)
![链接2](/images/posts/20191105Markdown/link_2.jpg)

##### 高级链接
```
	这个链接用 1 作为网址变量 [Google][1]
	这个链接用 runoob 作为网址变量 [Runoob][runoob]
	然后在文档的结尾为变量赋值（网址）
  	[1]: http://www.google.com/
  	[runoob]: http://www.runoob.com/
```
效果如下图
![链接3](/images/posts/20191105Markdown/link_3.jpg)
***

#### 图片
* **语法：**开头一个感叹号`!`接着一个`[ ]`，里面放上图片的替代文字,接着一个`( )`，里面放上图片的网址，最后还可以用引号包住并加上选择性的 `title`属性的文字.
```
	![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png)
	![RUNOOB 图标](http://static.runoob.com/images/runoob-logo.png "RUNOOB")
```
效果如下图
![图片](/images/posts/20191105Markdown/picture.jpg)
***

#### 表格
* **语法：**制作表格使用 | 来分隔不同的单元格，使用 - 来分隔表头和其他行，`-:`设置内容和标题栏居右对齐；`:-`设置内容和标题栏居左对齐；`:-:`设置内容和标题栏居中对齐.
 ```
	| 左对齐 | 右对齐 | 居中对齐 |
	| :-----| ----: | :----: |
	| 单元格 | 单元格 | 单元格 |
	| 单元格 | 单元格 | 单元格 |
 ```
效果如下图
![表格](/images/posts/20191105Markdown/form.jpg)
***