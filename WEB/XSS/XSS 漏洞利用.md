
## DVWA
` v1.10 *Development*

### Low

#### XSS (Reflected)

##### 分析

一个将输入文本输出到前端界面中的小功能，点击 Submit 按钮会发起一个 Get 请求，输入内容作为 请求参数 name 的值传输给 PHP 后端处理

![[Pasted image 20230424181348.png]]

![[Pasted image 20230424181413.png]]

##### 利用

Low 等级 DVWA 未对输入参数做出处理

![[Pasted image 20230424181907.png]]


#### XSS (DOM)

##### 分析

一个单选框，点击 select 按钮会发起一个Get 请求，参数为选中的内容

![[Pasted image 20230424182342.png]]

