
## DVWA
>靶场版本：v1.10 \*Development\*

### XSS (Reflected)

#### Low

>分析

一个将输入文本输出到前端界面中的小功能，点击 Submit 按钮会发起一个 Get 请求，输入内容作为 请求参数 name 的值传输给 PHP 后端处理

![](../../image/Pasted%20image%2020230424181348.png)

>利用

Low 等级 DVWA 未对输入参数做出处理

payload:  `<script>alert(document.cookie)</script>`

*** 
#### Mid

>分析

将 low 等级的 payload 输入，发现 `<script>` 标签被过滤，应该是后端使用了黑名单策略，将敏感词汇替换了。查看源代码，发现是将 script 这个标签替换掉了

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

只对某一个标签进行过滤的操作是很容易被绕过的，常用的思路如下：
1. 使用大小写绕过，输入  `<sCript>alert(document.cookie)</script>`
2. 使用双写绕过，输入 `<scr<script>ipt>alert(document.cookie)</script>`
3. 输入其他标签，如 `<IMG src=1 onerror=alert(document.cookie)>`

>利用

payload：`<script>alert(document.cookie)</script>`

***
#### High



### XSS (DOM)

#### Low

==分析==

一个单选框，点击 select 按钮会发起一个Get 请求，参数为选中的内容

![](../../image/Pasted%20image%2020230424182342.png)


利用的思路在浏览器 url 栏中将 payload 插入该 Get 请求参数后

*利用*

payload：

`http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=english<script>alert(document.cookie)</script>`

#### Mid

==分析

将 low 等级的 payload 输入，发现网站把我们重定向到了原网页，并且将我们的参数修改为 English，查看源代码，发现是后端做了一个查找`<script>` 标签的判断，如果输入中存在 这个标签，那么直接重定向到原网页

**![](../../image/Pasted%20image%2020230424193809.png)

在浏览器 url 栏中将参数值修改为 1，寻找参数在网页上渲染的位置

![](../../image/Pasted%20image%2020230424183649.png)

因为该程序在检测到 `<script>` 标签后会进行重定向且将参数修改为默认值，因此修改大小写和双写的思路是行不通的，这里我们可以使用其他标签进行注入，但是要记得插入其他标签前，**要将 `<option>` 和 `<select>` 标签人为闭合**，因为这两个标签不支持嵌套其他标签。


这里提供一些常见的使用其他标签进行注入的思路：

1. `<img src='1' onerror='alert(1)' style="display:none">`
2. `<iframe src="javascript:alert(1)">`

*利用

payload: `http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=English\</option></select><img src=1 onerror="alert(1)"  style="display:none"/>`

*源代码：

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

### XSS (Stored)

#### Low

==分析

一个文本输入框，可以将用户的输入渲染到页面上

![](../../image/Pasted%20image%2020230424190637.png)

利用方式和前面两种类似，将 payload 输入即可

*利用

payload: `<script>alert(document.cookie)</script>`

#### Mid

==分析

将 low 等级的 payload 输入，发现输出内容少了 script 标签，猜测应该也是使用了黑名单检测，尝试使用大写的 Script 绕过，无效，后续尝试在 Message 这一栏中输入了多种 HTML 标签绕过方式，都无效。

将 payload 注入的目标改为 Name 这一栏，发现前端对这个输入框做了长度限制，要解决这个问题一般有两种思路：
1. 在前端禁用限制
	1. 如果是在 HTML 代码中加的限制，直接在浏览器修改即可
	2. 如果是用 JS 代码做的限制，那么可以在浏览器禁用 JS
2. 抓包，在 HTTP 请求中修改参数

这两种方式能绕过前端对参数的限制，但是如果程序员在后端也加了校验就没办法了。

*利用

抓包，修改 Name 的参数为 `<Script>alert(1)</script>`

![](../../image/Pasted%20image%2020230425000602.png)

成功，利用这种方式也可以注入其他 payload

![](../../image/Pasted%20image%2020230425001108.png)

*源代码：

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

