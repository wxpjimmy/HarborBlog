title: Markdown 语法学习
date: 2015-10-11 19:16:39
tags: [script, markdown]
---
## 基本符号  
> - \*,\-,\+ 3个符号效果都一样，这3个符号被称为**Markdown符号**
> - 空白行表示另起一个段落
> - `是表示inline代码，tab是用来标记 代码段，分别对应html的code，pre标签

## 换行  
- 单个回车，视为空格
- 单一段落用一个空白行 （连续回车）
- 连续两个空格 会变成一个段内换行
- 连续3个符号，然后是空行，表示 hr横线

## 标题  
* 生成h1--h6,在文字前面加上 1\-6个# 来实现
* 文字加粗是通过 文字左右各两个符号

## 引用  
- 在第一行加上 “>”和一个空格，表示代码引用，还可以嵌套

## 列表  
这个是markdown文件的主要表达方式：
1. 使用\*,\+,\-加上一个空格表示
2. 可以支持嵌套, 比如二级列表只需要在符号前面加上两个空格即可
3. 有序列表使用 数字\+英文点\+空格来表示
4. 列表内容很长，不需要手工输入换行符，css控制段落的宽度，会自动的缩放的

## 链接  
+ 直接写\[锚文本\](url "可选的title")
+ 引用: 先定义[ref_name]:url, 然后再需要url的地方，这样使用[锚文本][ref_name]，通常的ref_name一般用数组表示，这样显得专业
+ 简写url: 用尖括号包裹url，这样生成的url锚文本就是url本身

## 插入图片  
* 一行表示：\!\[alt_text\](url "可选的title")
* 引用表示法：![alt_text][id], 预先定义[id]:url “可选的title”)
* 直接使用&lt;img>标签，这样可以指定图片的大小尺寸

## 特殊符号  
- 用\来转义，表示文本中的markdown符号
- 可以在文本中直接使用html标签，但是要注意在使用的时候，前后加上空行
- 文本前后各加一个符号，表示斜体；各加两个符号，表示粗体

## 代码引用  
* 行内的代码引用可以使用\`\`, 例如`var example = true`。  
* 多行的代码块引用可以在每行行首加上四个空格，也就是缩进四个空格:  


    if (isAwesome){
        return true
    }

* GitHub也支持使用两行\`\`\`对多行代码区块进行表示，无需缩进：  
\`\`\`
if (isAwesome){  
    return true  
}  
\`\`\`  
渲染后的效果如下：
```
if (isAwesome){
    return true
}
```
  
如果想使用指定语言的语法高亮，可以在第一行的\`\`\`之后加上使用的语言，例如：  
\`\`\`javascript
if (isAwesome){  
   return true  
}  
\`\`\`  
渲染后的结果如下：  
```javascript
if (isAwesome){
   return true
}
```

## 特殊字符自动转换 
在 HTML 文档中，有两个字符需要特殊处理： < 和 & 。 < 符号用于起始标签，& 符号则用于标记 HTML 实体，如果只是想要使用这些符号，必须要使用实体的形式，像是 `&lt;` 和 `&amp;`  

## GitHub扩展语法  
GitHub支持很多MarkDown的扩展语法，以便更好的进行交互，如果想对某人进行评论，你可以在他的昵称前面加上 @ 符号, 例如：  
Hey @wxpjimmy - nice day, isn't it?

还有，任务列表可以这样表示：
- [x] This is a complete item  
- [ ] This is an incomplete item

另外，表情符号也是可以使用的： :sparkles: :camel: :boom:  
两个波浪号引起来的字符代表删除线，比如 ~~this~~  
表格可以用如下的语法创建：列与列之间用竖线`|`分隔，表头行跟其它行之间用横线`-`分割，示例如下：  
```
Name | Age
---- | ---
Jimmy | 23
Allen | 32
David backham | 40
```
渲染后的效果如下：

Name | Age  
---- | ---  
Jimmy | 23  
Allen | 32  
David backham | 40   

## 学习链接  
1. [Markdown语法说明](http://www.ituring.com.cn/article/504)
2. [怎么使用Markdown](http://www.ituring.com.cn/article/23)
3. [Markdown简明语法](http://lutaf.com/markdown-simple-usage.htm)
4. [Markdown指南](http://zipperary.com/2013/05/22/introduction-to-markdown/)

