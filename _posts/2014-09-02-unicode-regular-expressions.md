---
layout: post
title: unicode正则表达式书写技巧
description: "unicode正则表达式"
modified: 2014-0９-2
tags: [unicode]
image:
  background: triangular.png
comments: true
---

Unicode字符多种多样，除去ascii中的字母、数字、标点和中文字符，还包括其它多种语言和多种符号，有些符号甚至很难打出来，这时候该如何表示呢？接下来我们首先简单的介绍以下Unicode的编码特点，然后针对不同的场景分别介绍如何写正则表达式。

----

###1.Unicode的编码特点

每一个Unicode字符都对应一个唯一的Unicode编码，一种语言的Unicode编码是在一个连续区间内的，除了这些基本特征外，Unicode编码还有三种属性Property、 Block、Script。我们可以用`\p`引用Unicode的属性，

>**Property**用于表示字符本身的功能，比如符号，空格，字母等。可以用`\p{L}`表示字母、文字，而对应`\P{L}`表示不是字母、文字。Property是与语言无关的，即`\p{L}`用于表示所有语言的字符。比如：

```
L或者Letter:所有语言的字符
P或者Punctuation:所有语言的标点 
S或者Separator:所有语言的分隔符
```

>**Block**表示Uincode的一个区间，某种语言的字符通常是落在同一区间的，所以可以通过Block粗略表示某类语言的字符，比如`\p{InHebrew}`表示希伯莱语字符，`\p{InCJK_Compatibility}`表示兼容CJK（汉语、韩语、日本语）的字符。如下表所示：每一个Block表示方法都对应一段编码区间：

```
InCJK_Compatibility:\u3300–\u33F
InHebrew:\u0590–\u05F
```

>**Script**表示Uincode的一个书写系统，比如`\p{Greek}`表示希腊语字符，`\p{Han}`表示汉语（中文字符），比如如下列举了希腊，汉语和阿拉伯书写系统。

```
Greek
Han
Arabic
```

-----------


###2.Unicode正则应用案例

* 匹配单个Unicode字符，下面例子去掉"发"字。

{% highlight java%}
String testStr = “发财了，发了”;
testStr.replaceAll("\u53d1",testStr);
{% endhighlight %}


* 匹配所有标点符号，下面例子去掉所有标点符号


{% highlight java%}
String testStr = “发财了，发了！<Software Engineer is great!>”;
testStr.replaceAll("\\pP",testStr);
{% endhighlight %}


* 匹配某种语言，下面例子去掉所有汉字


{% highlight java%}
String testStr = “发财了，发了！<Software Engineer is great!>”;
testStr.replaceAll("\\pHan",testStr);
{% endhighlight %}

-------------
关于uicode编码属性可以参考[这篇文章](http://www.regular-expressions.info/unicode.html)。</br>
关于Unicode码表可以参考[这里](http://zh.wikibooks.org/wiki/Unicode)
