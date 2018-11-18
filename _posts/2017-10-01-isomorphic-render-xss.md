---
layout: post
title: 同构渲染的常见风险
description: "同构渲染状态传递过程中常见的问题"
modified: 2017-10-01
tags: [react, isomorphic render, xss]
image:
  background: triangular.png
comments: true
---


使用同构渲染可以明显地改善页面的首屏时间，在各大前端团队中，该技术方案已是标配。同构渲染时，我们往往通过如下方法，把服务端的初始页面状态数据传递到浏览器。

``` javascript
function renderPage(renderedString, initialState){
    return `
        <html>
            <head> isomorphic render </head>
            <body>
                <div id="root">{renderedString}</div>
                <script>
                    window.__initialState = ${JSON.stringify(initialState)}
                </script>
            </body>
        </html>
    `
}
```

通过在script标签中对window对象赋值，我们就将服务端初始状态传递到浏览器了。但这种方法至少有两个安全性问题。


###1. xss攻击

由于将initialState未经转义地的传递到页面上，很容易出现XSS攻击。比如

```
initialState={
	id: "1",
	name: "foio</script><script>alert('xss')</script>xss"
}
```

解决方式是转义`<`、`>`、`/` 等字符


###2. 编码兼容性

首先要明确一点：历史上很长一段时间，JSON的字符集是大于Javascript的字符集。也就是说，JSON的某些字符（\u2028\u2029）会导致Javascript语法错误。而只有实现了[proposal to make all JSON text valid ECMA-262](https://github.com/tc39/proposal-json-superset)提案的JS引擎，才能正确的这种差异。作为最先进的浏览器，Chrome也是2018年4月才实现了该提案。目前大多上移动设备的浏览器内核都没有实现提案（包括微信）。一个常见的风险是，由于js引擎的差异，我们在node(v8)中能够正确处理，但是到了移动端(微信)中就会报错：“SyntaxError: Invalid or unexpected token”。

解决方式是转义`\u2028`、`\u2029`字符


最后安利一个能够解决以上两个问题的库：https://github.com/yahoo/serialize-javascript

```javascript
serialize(initialState, {isJSON: true});
```
我们也可以自己实现，将五个敏感字符全部转义：

```javascript
function javscriptEncode(source){
	if(!source)	{
		return ''
	}
	let distStr = '';
	for(let i = 0; i < source.length; i++){
		const ch = source.charCodeAt(i);
		console.log(ch);
		if (
				ch == '<' 
				|| ch == '/'
				|| ch == '>' 
				|| ch == '\u2028' 
				|| ch >= '\u2029'
			){
				distStr +=  ("\\x" + ch.toString(16));
		}else{
			distStr += source[i];	
		}
	}
	return distStr;
}
```

https://medium.com/javascript-security/avoiding-xss-in-react-is-still-hard-d2b5c7ad9412
https://github.com/yahoo/serialize-javascript
https://github.com/yahoo/xss-filters
https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet
https://medium.com/node-security/the-most-common-xss-vulnerability-in-react-js-applications-2bdffbcc1fa0
https://github.com/punkave/sanitize-html
https://gist.github.com/joni/3760795
https://github.com/tc39/proposal-json-superset

