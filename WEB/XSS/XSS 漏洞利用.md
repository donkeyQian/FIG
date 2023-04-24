
## DVWA
` 靶场版本：v1.10 *Development*

### Low

#### XSS (Reflected)

**分析**

一个将输入文本输出到前端界面中的小功能，点击 Submit 按钮会发起一个 Get 请求，输入内容作为 请求参数 name 的值传输给 PHP 后端处理

![](../../image/Pasted%20image%2020230424181348.png)

![](../../image/Pasted%20image%2020230424181413.png)

**利用**

Low 等级 DVWA 未对输入参数做出处理

![](../../image/Pasted%20image%2020230424181907.png)


#### XSS (DOM)

**分析**

一个单选框，点击 select 按钮会发起一个Get 请求，参数为选中的内容

![](../../image/Pasted%20image%2020230424182342.png)

寻找参数在网页上渲染的位置，将参数值修改为 1

\http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=1

![](../../image/Pasted%20image%2020230424183649.png)

接下来的思路为输入 payload ，插入我们的利用代码

**利用**

payload：

\http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=english\<script>alert('donkeyQian')\</script>

![](../../image/Pasted%20image%2020230424190320.png)

#### XSS (Stored)

**分析**

一个文本输入框，可以将用户的输入渲染到页面上

![](../../image/Pasted%20image%2020230424190637.png)

利用方式和前面两种类似，将payload 输入即可

**利用**

![](../../image/Pasted%20image%2020230424190929.png)

### Mid

#### XSS (Reflected)

**分析**

将 low 登记的 payload 输入，发现 \<script> 标签被过滤，应该是后端使用了黑名单策略，将敏感词汇替换了。查看源代码，发现是将 script 这个标签替换掉了

```php
`<?php      
header ("X-XSS-Protection: 0");      
// Is there any input?   
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {    
	// Get input    
	$name = str_replace( '<script>', '', $_GET[ 'name' ] );    
	// Feedback for end user    
	echo "<pre>Hello ${name}</pre>";   
}      
?>`
```

但是 str_replace() 这个函数式区分大小写的，因此我们只需要将 \<script> 修改为 \<Script> 就能绕过这个规则。

**利用**

payload：\<script>alert(1)\</script>

#### XSS (DOM)

**分析**

将 low 等级的 payload 输入，发现网站把我们重定向到了原网页，并且将我们的参数修改为 English，查看源代码，发现是后端做了一个查找 \<script> 标签的判断，如果输入中存在 这个标签，那么直接重定向到原网页，且参数为 english

**![](../../image/Pasted%20image%2020230424193809.png)

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

**利用

stripos() 这个函数是不区分大小写的，所以修改大小写的策略在这里就用不了了，但我们还有很多其他的绕过策略，并不一定要使用 script 标签。

payload: \http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=English\</option>\</select>\<img src=1 onerror="alert(1)"  style="display:none"/>

这里将 option 标签和 select 标签提前闭合的原因是这两个标签中不能放 img 标签，所以我人为将他们闭合，并插入 img 标签
![](../../image/Pasted%20image%2020230424195045.png)

这里提供一些常见的替换 script 标签的思路：

1. \<img src='1' onerror='alert(1)' style="display:none">
2. \<iframe src="javascript:alert(1)">


1231231