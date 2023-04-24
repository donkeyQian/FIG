
## DVWA
` v1.10 *Development*

### Low

#### XSS (Reflected)

##### 分析

一个将输入文本输出到前端界面中的小功能，点击 Submit 按钮会发起一个 Get 请求，输入内容作为 请求参数 name 的值传输给 PHP 后端处理

![](../../image/Pasted%20image%2020230424181348.png)

![](../../image/Pasted%20image%2020230424181413.png)

##### 利用

Low 等级 DVWA 未对输入参数做出处理

![](../../image/Pasted%20image%2020230424181907.png)


#### XSS (DOM)

##### 分析

一个单选框，点击 select 按钮会发起一个Get 请求，参数为选中的内容

![](../../image/Pasted%20image%2020230424182342.png)

寻找参数在网页上渲染的位置，将参数值修改为 1

http://192.168.127.200/DVWA/vulnerabilities/xss_d/?default=1

![](../../image/Pasted%20image%2020230424183649.png)

接下来的思路为输入 payload 将 option 标签 和 select 标签提前闭合，并插入我们的利用代码

