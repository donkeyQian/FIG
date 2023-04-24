
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

![](../../image/Pasted%20image%2020230424182342.png)


利用的思路在浏览器 url 栏中将 payload 插入该 Get 请求参数后

>利用

payload：`http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=english<script>alert(document.cookie)</script>`

***
#### Mid

>分析

将 low 等级的 payload 输入，发现网站把我们重定向到了原网页，并且将我们的参数修改为 English，查看源代码，发现是后端做了一个查找`<script>` 标签的判断，如果输入中存在 这个标签，那么直接重定向到原网页。

![](../../image/Pasted%20image%2020230424193809.png)

在浏览器 url 栏中将参数值修改为 1，寻找参数在网页上渲染的位置

![](../../image/Pasted%20image%2020230424183649.png)

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

![](../../image/Pasted%20image%2020230425044204.png)

但是在这之前，我们的请求是要发送给后端的，如果后端直接将请求给拒绝了，或者是向 mid 难度那样直接重定向再修改参数，那前端也是接收不到我们的 payload 的。

在这里要科普一个小技巧，Get 请求中的 # 号表示位置标识符，某些页面内跳转的功能就是这么实现的，比如：
https://github.com/donkeyQian/FIG/blob/dev/WEB/XSS/XSS%20漏洞利用.md#xss-reflected

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

![](../../image/Pasted%20image%2020230424190637.png)

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

![](../../image/Pasted%20image%2020230425000602.png)

成功，利用这种方式也可以注入使用其他标签的 payload

![](../../image/Pasted%20image%2020230425001108.png)

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

> 分析

经过 Mid 等级的测试，发现无法通过 message 参数输入 payload，因此，还是尝试通过 Name 参数注入 payload。这次，我们不使用抓包，尝试通过修改 HTML 破解对 Name 参数长度的限制。使用浏览器的审查元素功能，将 Name 输入框的 maxlength 属性值修改为较大的数字或者直接删除这个属性。

![](../../image/Pasted%20image%2020230425041127.png)

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

