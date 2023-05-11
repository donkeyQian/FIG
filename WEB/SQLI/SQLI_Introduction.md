## 介绍

>SQL 注入的原理

开发时未对用户的输入数据（可能是GET或POST参数，也可能是Cookie、HTTP头等）进行有效过滤，直接带入SQL语句解析，使得`原本应为参数数据的内容却被用来拼接SQL语句做解析`，也就是说，将数据当代码解析，最终导致SQL注入漏洞的产生。

用人话来说就是应用程序将攻击者的恶意代码与SQL 语句拼接在一起解析、执行，导致程序执行了超出预期的 SQL 操作。

	举个例子
原SQL：`SELECT * FROM users WHERE id='$id' LIMIT 0,1`
payload：`1' or 1=1 -- `
数据库最终执行的SQL：`SELECT * FROM users WHERE id='1' or 1=1 -- ' LIMIT 0,1`

> SQL 注入能做什么？

1. 绕过验证
2. 篡改数据
3. 窃取数据
4. 消耗资源

> SQL 注入常用工具

`sqlMap` 是社区中最好用的 SQL 注入扫描利用工具，学会使用这个工具已经足以 cover 使用场景中的大部分情况了。

但是扫描器并不是万能的，安全从业人员脑子里的知识才是安身立命的本钱，因此，学会乃至精通手工注入是有必要的，面对扫描器无法扫描出来的漏洞时，我们也可以通过手工注入的方式测试漏洞然后编写 payload，交给 sqlMap 来利用、执行。

## 分类

从注入参数的类型的角度分，可以将 SQL 注入分为：
1. 字符型注入
2. 整型注入

这两种类型的区别为参数的类型，可以简单的理解为有没有为参数加上引号。

阅读源码能很直观的看出这两种注入的区别：
```php
# 字符型注入
SELECT * FROM users WHERE id='$id' LIMIT 0,1
# 整型注入
SELECT * FROM users WHERE id=$id LIMIT 0,1
```

如果是没有源码的黑盒测试，可以通过在参数中添加 `'` `"` 的方式来测试注入参数的类型。

***
从利用方式的角度分，可以将 SQL 注入分为：

1. 布尔型盲注（Boolean-based blind）
2. 报错型注入（Error-based）
3. 联合查询注入（Union query-based）
4. 多语句堆叠注入（Stacked queries）
5. 基于时间延迟盲注（Time-based blind）
6. 内联/嵌套查询注入（Inline queries）
