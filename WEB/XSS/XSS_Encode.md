
>引用：
	- [探索XSS利用编码绕过的原理 - SAUCERMAN (saucer-man.com)](https://saucer-man.com/information_security/103.html)
	- [防御XSS的七条原则 | Web应用安全实验室 (sking7.github.io)](https://sking7.github.io/articles/430468050.html)
	- [前端安全系列（一）：如何防止XSS攻击？ - 美团技术团队 (meituan.com)](https://tech.meituan.com/2018/09/27/fe-security.html)
	- [浏览器解析与编码顺序及xss挖掘绕过全汇总 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1516371)

所谓的 XSS 编码绕过，就是攻击者将 payload 按照某些编码规则进行编码，得到一串无法被 XSS 过滤器识别的字符串，来达到 XSS 编码绕过的目的。这种绕过方式对黑名单过滤检测比较有效。

看完一圈文章，总结了如下问题，希望自己在写文章的时候用自己的语言回答这些问题，同时也能总结出 XSS 编码防御和编码绕过的来龙去脉，最好还能总结出一些新的问题。我将自己的这种学习方式称之为日记型笔记，说白了就是想到什么记什么。

1. 能使用哪些编码进行绕过？
2. 为什么有些网站上明明显示了 `<script> </script>` 标签，但是里面的 js 代码却不会被浏览器执行？
3. 为什么编码既能既能防御 XSS 攻击，也能作为 XSS 绕过的手段？
4. 浏览器是怎么识别被编码后的 js 代码的？
5. 哪些内容可以使用编码，哪些不能？
6. 使用编码防御 XSS 攻击的总结。

## 编码相关知识

要想学习 XSS 编码绕过，首先起码得知道浏览器使用了哪些编码来解码，起码能分辨出来这些编码才行。

下面，我会介绍三种必须学会的编码，学习了这些内容，应该能解决第一个问题 `能使用哪些编码进行绕过？`

### URL 编码

- `https://image.baidu.com/search/albumsdetail?tn=albumsdetail&word=渐变风格插画&fr=albumslist&album_tab=设计素材&album_id=409&rn=30`
- `http://192.168.127.200:9000/vul/xss/xss_reflected_get.php?message=1&submit=submit`
- `scheme://login:password@address:port/path？query_string#fragment`

以上都是常见的 url 结构，可以看到，像 `? & / :// #` 这些字符是浏览器用来解析 URL 时分隔语义的保留字符，那么问题来了，如果 url 中某个部分的名称用到了这些字符，就会破坏语法，影响正常解析。再者，也是为了支持 url 中存在像汉字这样的 ASCII 码之外的特殊字符。

于是，就有了 url 编码，他的作用是将 url 中的保留字符和特殊字符编码成 `%1F` 这样的百分号后面加两个16进制数的形式来消除 url 解析过程中的歧义和字符编码问题。

详细的 url 编码规则可以参考：[URL编码规则 - zhangyukun - 博客园 (cnblogs.com)](https://www.cnblogs.com/cxygg/p/9278542.html)，我在这里就不再赘述了。

### HTML 编码

跟url的问题类似，一些字符在 HTML 中也是是预留的，像<这样的对于HTML来说有特殊意义的字符，在浏览器中会被解析成各种标签，如果要作为纯文本输出这个字符，就需要用到字符实体。

比如，在 mybatis 的 .mapper 文件中直接输入 < 是非法的，因为 < 是 xml 文件的保留字符，必须把他转义成 `&lt;`，这个文件才能被正常解析。

#### HTML 编码的类型

HTML 编码可以分为两种类型：

- 实体名称，如 < 的实体名称为 `&lt;`
- 实体编号
	- 10进制，如 < 的实体编号 10 进制形式为 `&#60;` 
	- 16进制，如 < 的实体编号 10 进制形式为 `&#x3c;` 

他们的共同点是在浏览器上的效果相同，而且都以连接符 `&` 开头以分号 `；` 结尾。不同点则是实体名称中间的字符为英文字符，更方便记忆，实体编号中间的字符为10进制或16进制的数，但是就浏览器的支持性来说实体编码要好一些，更推荐使用实体名称。

常见的实体如下：

![](../../image/Pasted%20image%2020230429155859.png)

### JS 编码

道理同上，js常见的反斜杠方式编码处理

- `\b`退格符，`\t`制表符，`\v`垂直制表符等；
- 三位数字，不足位数用0补充，按8位原字符八进制字符编码；
- 两位数字，不足位数用0补充，按8位原字符16进制字符编码，前缀 x 
- 四位数字，不足为数用0补充，按16位原字符16进制Unicode数值编码，前缀 u 。

如 `\145`、`\\x65`和 `\u0065`都代表字符e。

## 浏览器的解析和解码

解析指的是浏览器将后端传递过来的数据解析成一个网页（结构化，图形化）的过程，解码指的是浏览器根据对应的编码规则将数据中的经过编码的数据解析成 HTML实体、JS 代码，正常的 url 的过程。

### 解析 

浏览器在解析HTML文档时无论按照什么顺序，主要有三个过程：HTML解析、CSS解析、JS解析和URL解析，每个解析器负责HTML文档中各自对应部分的解析工作。

![](../../image/Pasted%20image%2020230430142735.png)

从图中可以看出浏览器主要做了三部分的工作：
1. HTML/SVG/XHTML 解析。解析这三种文件会产生一个DOM Tree。
2. CSS 解析，解析CSS会产生CSS规则树。
3. Javascript 解析。（暂时讨论JavaScript动态操作DOM Tree）。
4. URL解析。（这一步的先后顺序不一定，看语境）

从图中还可以看出，JS 代码是会对 HTML DOM Tree 和 CSS DOM Tree 的结构进行修改的，因此，浏览器在逐行解析 HTML 代码时，如果遇到了`<script>` 标签，则暂停 HTML 标签解析，将控制权转交给 JavaScript 引擎，执行完之后继续解析 HTML 代码。

举例：
```html
<script>
	var content = document.getElementById("content");
	var img = document.createElement("img");
	img.src =   "c.com";
	img.setAttribute("onerror","alert('JS代码在DOM Tree 生成前,a.com')");
	content.appendChild(img);
</script>

<div id="content"></div>

<img src="b.com" onerror="alert('对照组,b.com')"/>

<p>Hello Parser!</p>

<script>
	var content = document.getElementById("content");
	var img = document.createElement("img");
	img.src =   "c.com";
	img.setAttribute("onerror","alert('JS代码在DOM Tree 生成后,c.com')");
	content.appendChild(img);
</script>
```

用来浏览器打开上述 HTML 代码，三个图片的 onerror 属性全部弹窗成功，但是 a.com 并没有成功插入 DOM Tree。

![](../../image/Pasted%20image%2020230430145624.png)

众所周知，计算机是通过二进制流的方式进行通信的。对于浏览器与服务器的通信，可以简单的理解为服务器给浏览器返回了一长串字符串。浏览器需要识别这一长串字符串中哪些是文本字符（浏览器不需要解析，只需要显示出来），哪些是控制字符（对于HTML来说就是能够被解析为DOM Tree的字符）。

作为攻击者，`尽可能将用户输入的值让浏览器识别为控制字符`，这样就可以造成反射XSS。简而言之，尽可能让你的 payload 被 JS 解析器解析成代码执行，而不是被 HTML 解析器解析成文本展示在页面上。

举例：
```html
hello,world!
<br/>
<script>alert('script 标签不使用HTML编码')</script>
&lt;script&gt;alert('script 标签不使用HTML编码')&lt;/script&gt;
```

![](../../image/Pasted%20image%2020230430150311.png)

可以看到，经过 HTML 编码的标签并没有被浏览器识别为 JS 代码，而是被识别成了文本。

这部分的内容就能回答下面这两个问题：

- 为什么有些网站上明明显示了 `<script> </script>` 标签，但是里面的 js 代码却不会被浏览器执行？
	- 因为网站将你输入的 script 标签编码成了 HTML 实体，而浏览器不会将 HTML 实体识别为标签。
- 为什么编码既能既能防御 XSS 攻击，也能作为 XSS 绕过的手段？
	- 因为编码防御的本质是对特殊字符进行过滤，而我们提前将 payload 进行编码，等于将 payload 改头换面，变成了另一种浏览器能识别的方式，那么先前对特殊字符进行的过滤就无效了。

### 解码

首先，简单介绍一下各种编码的解码方式。

#### URL 解码

url解码过程较为简单，服务器对接收到用户传输过来的URL进行解析，遇到%便自动进行解码。

***
#### HTML 解码

首先了解一下HTML解析器的工作原理：

HTML解析器其实是一个状态机，在对HTML资源从上而下进行解析时遇到一个 `<` 符号就会进入标签开始状态（Tag Open State），然后搜寻标签，img可以被识别为正确的标签，img1则不会识别，最后在读到最近的一个 `>` 时，结束标签状态进入数据状态（Data State）。

>哪些HTML字符实体会被解析？

一般来说，HTML编码要在 Data state（标签外部和标签的text段），标签内的属性值的位置才能被解析。可以对各个部分进行测试，是否可以使用实体替换以及执行效果如何：

```html
<input name="&#105;&#110;&#112;&#117;&#116;" onclick="alert('HTML编码')" &#118;&#97;&#108;&#117;&#101;="inputBox" >&#72;&#84;&#77;&#76;&#32534;&#30721;</input>
<br>
<input type="text" name="input1" onclick="alert('对照组')" value="inputBox1" />
```

总结一下，标签里的属性值、标签之间的 HTML 实体可以被解析。

![](../../image/Pasted%20image%2020230430164917.png)

---
#### JS 解码

Js解码就简单很多，js的脚本处理模型是按照源码处理-函数解析-代码执行这个执行流来的，不管是外部引用还是直接写在script标签里，又或者是在html标签的属性里，对于js编码的解码都是相同的。

所以分别对函数编码：

```js
<script>
	\u0061lert("HelloWorld");
</script>
```

对 value 值进行编码：

```js
<script>
	alert("\u0048elloWorld");
</script>
```

效果都是一样的，xss挖掘中这样的编码适用于js代码环境中alert等函数被过滤的情况。

#### 浏览器的解码顺序

经过前面的学习，我们知道了浏览器在解析 HTML 时，遇到 `<script>` 标签会暂停 HTML 代码解析，将执行权交给 JS 引擎，等JS 代码执行完毕之后再继续 HTML 代码解析。因此，如果注入点最终会将数据渲染到 `<script>` 标签中，那么就不必使用 HTML 解码和 url 解码，也就没有解码顺序这一说了。

比如：

```html
<script>
\u0061\u006c\u0065\u0072\u0074('https%3A%2F%2Fexample%3Fmessage%3D%E6%88%91%E6%98%AF%E5%A4%A7%E5%82%BB%E9%80%BC%EF%BC%9F');
</script>

<script>
	\u0061\u006c\u0065&#114;&#116;('https%3A%2F%2Fexample%3Fmessage%3D%E6%88%91%E6%98%AF%E5%A4%A7%E5%82%BB%E9%80%BC%EF%BC%9F');
</script>
```

第二段代码无法被解析为 JS 代码，所以无法执行。

***

所以关于浏览器对这几种编码的解码顺序，我觉得只有在 JS 代码放在 HTML 标签中这种情况下讨论才有意义。

比如下面这种情况：

```html
<img src=1 onerror="alert('1')" />
```

关于执行顺序，我在写笔记的时候查阅了大量大佬们的博客，发现大家说的有相同的情况，但是也有一些差异。

相同点是基本上都是先 HTML 解码，再 JS 解码。
不同点则在于 URL 解码的顺序，有 HTML > URL > JS 的，也有 HTML > JS > URL 的，还有说 URL 解码是根据时机来的。

*** 
首先我们来对相同点进行验证：

```html
只使用 HTML 编码
<br/>
<img src="https://avatars.githubusercontent.com/u/131577364?s=120&v=4" onclick="&#97;&#108;&#101;&#114;&#116;&#40;&#39;&#20182;&#22920;&#30340;&#65292;&#23398;&#19981;&#20250;&#65281;&#39;&#41;">
<br/>
只使用 JS 编码
<br/>
<img src="https://avatars.githubusercontent.com/u/131577364?s=120&v=4" onclick="\u0061\u006c\u0065\u0072\u0074('\u4ed6\u5988\u7684\uff0c\u5b66\u4e0d\u4f1a\uff01')">
<br/>
先使用 JS 编码，再 HTML 编码
<br/>
<!-- \u0061\u006c\u0065\u0072\u0074('\u4ed6\u5988\u7684\uff0c\u5b66\u4e0d\u4f1a\uff01') -->
<img src="https://avatars.githubusercontent.com/u/131577364?s=120&v=4" onclick="&#92;&#117;&#48;&#48;&#54;&#49;&#92;&#117;&#48;&#48;&#54;&#99;&#92;&#117;&#48;&#48;&#54;&#53;&#92;&#117;&#48;&#48;&#55;&#50;&#92;&#117;&#48;&#48;&#55;&#52;&#40;&#39;&#92;&#117;&#52;&#101;&#100;&#54;&#92;&#117;&#53;&#57;&#56;&#56;&#92;&#117;&#55;&#54;&#56;&#52;&#92;&#117;&#102;&#102;&#48;&#99;&#92;&#117;&#53;&#98;&#54;&#54;&#92;&#117;&#52;&#101;&#48;&#100;&#92;&#117;&#52;&#102;&#49;&#97;&#92;&#117;&#102;&#102;&#48;&#49;&#39;&#41;">
<br>
先使用 HTML 编码，再 JS 编码
<br/>
<!-- &#97;&#108;&#101;&#114;&#116;('&#20182;&#22920;&#30340;&#65292;&#23398;&#19981;&#20250;&#65281;') -->
<img src="https://avatars.githubusercontent.com/u/131577364?s=120&v=4" onclick="\u0026\u0023\u0039\u0037\u003b\u0026\u0023\u0031\u0030\u0038\u003b\u0026\u0023\u0031\u0030\u0031\u003b\u0026\u0023\u0031\u0031\u0034\u003b\u0026\u0023\u0031\u0031\u0036\u003b('\u0026\u0023\u0032\u0030\u0031\u0038\u0032\u003b\u0026\u0023\u0032\u0032\u0039\u0032\u0030\u003b\u0026\u0023\u0033\u0030\u0033\u0034\u0030\u003b\u0026\u0023\u0036\u0035\u0032\u0039\u0032\u003b\u0026\u0023\u0032\u0033\u0033\u0039\u0038\u003b\u0026\u0023\u0031\u0039\u0039\u0038\u0031\u003b\u0026\u0023\u0032\u0030\u0032\u0035\u0030\u003b\u0026\u0023\u0036\u0035\u0032\u0038\u0031\u003b')">
<br>
对照组，未经过任何编码
<br/>
<img src="https://avatars.githubusercontent.com/u/131577364?s=120&v=4" onclick="alert('他妈的，学不会！')">
```

家人们可以把这段代码放到浏览器验证一下，上面的所有图片都有一个 onclick 功能，点击之后都会弹出一个 `他妈的，学不会！` 的弹窗，只不过我对不同图片的 onclick 功能进行了不同的编码处理。

除了第四个图片点击无反应，其他的图片都是能正常弹窗的。证明了浏览器先对内容进行 HTML 解码再 JS 解码这个顺序是正确的。

![](../../image/Pasted%20image%2020230430175540.png)

*** 
继续验证 URL 解码的顺序问题：

初步的结论是 URL 解码的顺序跟 HTML 标签的不同属性有关。

```html
<a href="javascript:alert('他妈的，学不会！')" onclick="javascript:alert('他妈的，学不会！')">对照组，不做任何编码</a>
<br/>
<a href="javascript:alert('%E4%BB%96%E5%A6%88%E7%9A%84%EF%BC%8C%E5%AD%A6%E4%B8%8D%E4%BC%9A%EF%BC%81')" onclick="javascript:alert('他妈的，学不会！')">只对 href 值做 URL 编码</a>
<br>
<a href="javascript:alert('他妈的，学不会！')" onclick="javascript:alert('%E4%BB%96%E5%A6%88%E7%9A%84%EF%BC%8C%E5%AD%A6%E4%B8%8D%E4%BC%9A%EF%BC%81')">只对 onclick 值做 URL 编码</a>
```

家人们可以把上面的代码到浏览器里验证一下，结论是在 href 里面用 url 编码的内容能被正常解析，在 onclick 里面用 url 编码的内容不能被解析。

![](../../image/Pasted%20image%2020230430182114.png)

*** 
#### 有哪些内容不能使用编码绕过？

>URL 解码时，协议类型不能使用编码绕过

URL 解析器在对 URL 进行解析时，无法识别被编码后的协议名称

```html
<a href="javascript:%61%6c%65%72%74%28%27%78%73%73%27%29">只对协议后面的path进行编码</a>
<br>
<a href="%6a%61%76%61%73%63%72%69%70%74%3a%61%6c%65%72%74%28%27%78%73%73%27%29">对协议名称也进行编码</a>
```

第一个链接成功被解码，第二个链接无法被解码

>JS 解码时，经过编码的符号无法被解析（如 `)` `(` `'` 等）

在进行JavaScript解析的时候字符或者字符串仅会被解码为字符串文本或者标识符名称，在上例中Javascript解析器工作的时候将 `\u0061\u006c\u0065\u0072\u0074` 进行解码后为 `alert` ，而 `alert` 是一个有效的标识符名称，它是能被正常解析的。像圆括号、双引号、单引号等等这些字符就只能被当作普通的文本，从而导致无法执行，比如 `<script>alert('aaa\u0027)</script>`解析后缺少闭合的单引号而无法执行成功。

>`<textarea>` 和 `<title>` 标签中的内容无法被解析为 JS 代码

![](../../image/Pasted%20image%2020230430184817.png)

所以遇到输出在 `<textarea>` 之间的情况，如果不能使用 `</textarea>` 闭合，就要提早放弃。

## 编码防御

总结 [防御XSS的七条原则 | Web应用安全实验室 (sking7.github.io)](https://sking7.github.io/articles/430468050.html) 这篇博客中的编码防御原则，推荐大家看一看原文，受益匪浅。

1. 不要在页面中插入任何不可信数据，除非这些数已经据根据下面几个原则进行了编码
2. 在将不可信数据插入到HTML标签之间时，对这些数据进行 HTML Entity 编码
	1. 例如：\<div>…插入不可信数据前，对其进行HTML Entity编码…\</div>
3. 在将不可信数据插入到**HTML文本属性**里时，对这些数据进行HTML属性编码
	1. 例如：\<div attr=”…插入不可信数据前，进行HTML属性编码…”>\</div>
	2. 对于在事件处理属性中插入数据时，采用第四条原则
4. 在将不可信数据插入到SCRIPT里时，对这些数据进行SCRIPT编码
	1. 这条原则主要针对动态生成的JavaScript代码，这包括脚本部分以及**HTML标签的事件处理**属性（Event Handler，如onmouseover, onload等）
	2. 例如：\<script>var message = “<%= encodeJavaScript(@INPUT) %>”;\</script>
5. 在将不可信数据插入到Style属性里时，对这些数据进行CSS编码
	1. 当需要往Stylesheet，Style标签或者Style属性里插入不可信数据的时候，需要对这些数据进行CSS编码。
	2. 例如：\<span style=” property : …插入不可信数据前，进行CSS编码… ”> … \</span>
6. 在将不可信数据插入到HTML URL里时，对这些数据进行URL编码
7. 使用富文本时，使用 XSS 规则引擎进行编码过滤
	1. 推荐XSS规则过滤引擎：[OWASP AntiSamp](https://www.owasp.org/index.php/AntiSamy)或者[Java HTML Sanitizer](https://www.owasp.org/index.php/OWASP_Java_HTML_Sanitizer_Project)
