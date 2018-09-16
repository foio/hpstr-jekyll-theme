---
layout: post
title: 一种提取关键样式的方法
description: "使用chrome的代码覆盖能力和postcss工具获取准确的关键样式"
modified: 2017-07-01
tags: [关键样式, Critical CSS, postcss, Chrome debug protocol, Code Coverage]
image:
  background: triangular.png
comments: true
---

以React服务端渲染为代表的现代化web前端技术，使得web页面的首屏时间不再依赖于JS文件的加载时间(JS文件的加载和执行只影响页面的可交互时间)。

一个典型的web架构中，首屏渲染主要经历如下几个步骤：

```
（1）浏览器向接入层请求HTML

（2）接入层向后台服务请求数据后通过服务端渲染将HTML返回给浏览器

（3）浏览器向CDN请求CSS后开始渲染首屏
```

如下图所示:

![no critical css](/images/no-critical-css.png)

浏览器为了渲染首屏需要两次请求：一次向接入层请求HTML文档；另外一次向CDN请求CSS文件。有没有只需一次请求即可渲染出首屏的方案呢？ 通过使用内联样式，我们就可以使得浏览器不需要通过网络请求CSS；但对于一个中等规模的页面，需要内嵌的CSS内容过多，使得HTML文件本身过大，进而影响HTML本身的网络耗时，导致首屏耗时增加。我们需要一个折中方案来调和这种矛盾。

## 1.关键样式的作用

如果我们只是将首屏所需的关键样式内嵌到HTML页面中，然后通过网络请求获取非首屏所需的其他样式，就可以在保证一次请求即可渲染出首屏，同时尽量地减少了所需请求HTML文档的大小。

如下图所示：通过内嵌少量的首屏关键样式，我们就使得CSS和JS加载（由于此处的CSS不影响首屏，我们可以把它同JS一起放到HTML文档的末尾）不影响首屏时间了。

![critical css](/images/critical-css.png)

接下来的问题是: 我们如何准确的获取尽量小的关键样式？

## 2. 提取关键样式的方案

提取关键样式有多种方案：比如基于组件的方案，对于[CSS in JS](https://github.com/css-modules/css-modules)结构的前端项目就非常有效，只需要将首屏中所有组件的样式抽取出来即可；另外一种是基于运行时分析的方案，也就是本文提出的方案。

### 2.1. 使用Chrome code coverage能力提取关键样式

新版的chrome新增了code coverge能力，通过运行时分析能够准确地得知当前代码中未被使用的比例。如下图所示，
其中一个CSS文件有超过80%的代码未被使用；这也就意味着只有20%的CSS代码对于首屏渲染是有效的。

![critical css](/images/code-coverage.png)

有没有程序化的手段，来获取具体哪些代码是有效的呢？ 通过[Chrome Debug Protocol](https://github.com/cyrus-and/chrome-remote-interface)接口，我们可以在页面执行某个时间点(比如domReady时)，收集到当前已经使用到的CSS。具体伪代码如下：


``` javascript
//DOMContentLoaded时触发
Page.domContentEventFired(() => {
    const styleSheetIds = {};
    const styleSheetMap = {};

    CSS.takeCoverageDelta().then(rules=>{
        //获取已经被使用的CSS规则的ID
        const usedRules = rules.coverage.filter((rule) => {
            styleSheetIds[rule.styleSheetId] = 1;
            return rule.used;
        });

        //获取所有已经被使用规则片段
        Object.keys(styleSheetIds).forEach(styleSheetId => {
            stylesSheetsPromises.push(
                CSS.getStyleSheetText({
                    styleSheetId: styleSheetId
                }).then(stylesheet => {
                    styleSheetMap[styleSheetId] = stylesheet.text || '';
                })
            )
        })

        //根据规则片段拼接出关键css文本
        for (const usedRule of usedRules) {
            const curStyleId = usedRule.styleSheetId;
            const styleSeg = styleSheetMap[curStyleId].slice(usedRule.startOffset, usedRule.endOffset);
            //收集关键样式
            cirticalCssArr.push(styleSeg);
        }
        const cirticalStyle = cirticalCssArr.join('');

    })
})
```

以上只截取了部分代码,全部代码请参考[critical.js] (https://github.com/foio/chromeCriticalCss/blob/master/critical.js) :

虽然我们已经能够通过Chrome Debug Protocol来获取关键样式，但该关键样式仍然无法直接应用于生产环境。一个主要的原因是它丢失了Media Query信息；Chrome Code Coverage能力只能针对特定的User Agent工作，这就使得我们获取的关键样式不具有普适性。我们需要额外的解决方案。

### 2.2 使用Postcss获取关键样式中的Media Query

[postcss](https://github.com/postcss/postcss)作为业界最强大的CSS处理器，借助于它我们可以结构化地处理CSS，比如抽取关键样式中Media Query。通过结合Code Coverage和postcss，我们就能获取到准确的、UA无关地关键样式了。


![post critical css](/images/post-critical-css.png)

核心代码如下所示：

``` javascript
//解析全部css
const allStyleParser = postCss.parse(allStyle);
//结构化地遍历CSS
allStyleParser.walkAtRules(rule => {
    let isCritical = false;
    //遍历Media Query表达式
    if (rule.name.match(/^media/)) {
        //只选取关键的media query
        rule.each((subRule) => {
            const selector = subRule.selector;
            if(cirticalStyle.indexOf(selector) > 0){
                isCritical = true;
            }else{
                subRule.remove();
            }
        });
        if(isCritical){
            mediaQueriesCss += rule.toString();
        }
    }
})
//关键样式+MediaQuery为最终首屏核心样式
const coveredStyle = cirticalStyle + mediaQueriesCss;
```

以上只截取了部分代码,全部代码请参考[critical.js] (https://github.com/foio/chromeCriticalCss/blob/master/critical.js) :

本文提出的关键样式提取方案只适用于形态相对稳定的页面（尤其适合静态页面）；对于千人千面的个性化页面，可以尝试其他方案（基于组件的关键样式提取）。

https://www.gideonpyzer.com/blog/runtime-coverage-using-chrome-devtools/ 

https://github.com/cyrus-and/chrome-remote-interface 

https://github.com/foio/chromeCriticalCss

https://github.com/postcss/postcss


