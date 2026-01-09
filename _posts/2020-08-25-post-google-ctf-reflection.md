---
layout: "post"
title: "Post Google CTF Reflection"
date: "2020-08-25 07:21:25"
date_gmt: "2020-08-25 07:21:25"
modified: "2020-08-25 07:32:25"
modified_gmt: "2020-08-25 07:32:25"
slug: "post-google-ctf-reflection"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/08/25/post-google-ctf-reflection/"
guid: "http://y4y.space/?p=127"
wordpress_id: 127
parent_id: 0
categories: ["CTF Write Up", "Misc"]
---

{% raw %}
So Google CTF has concluded, and I was reading writeups for web challenges and hoping I can learn something new since I did not put too much time into it. Then I came across the challenge 'log-me-in'. It was an easy challenge, but I had some questions while read writeups for this one.

Essentially the solution is to post an object to the login page to get a authentication bypass. At my first glance I thought it was MongoDB auth bypass. But after reading the given source code I realized it was MySQL. The final payloads from various writeups were like:

```ini
username=michelle&password[password]=1&csrf=

csrf&username=michelle&password[username]=michelle
```

and in the backend the MySQL would look like:

```sql
SELECT * FROM users WHERE username='michelle' AND password=`password`=1;

SELECT * FROM users WHERE username='michelle' AND password=`username`='michelle';
```

And I tried to make sense of the above queries, and I just couldn't. I don't understand why there are three comparisons and why that would bypass the authentication. So I decided to test it on my own.

On my own little web server I had a MySQL service setup when I was trying to learn PHP. Though I no longer use PHP to build projects, the databases still remain. I pasted the queries above and see if I can get a result back, and there actually are results.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-24-22-41-29.png?w=969)

I thought the magic might be in those warnings, but after some researching it was nothing. I also tried like ``select * from user where name=`name`;``  and it worked as well. Then I tried to make sure what does the back-tick sign do in MySQL, and it turns out it's just to specify the field name. I also tried the above query without back-tick and it also worked. Then it's just weirder, how could that work? I googled things like 'mysql a=b=c' and I actually find my answer. [Here](https://stackoverflow.com/questions/19214675/why-does-select-a-b-c-return-1-in-mysql) is the original StackOverflow post.

In MySQL, when a string is compared with an integer, it will convert the string into float-point type. During the conversion, it will keep all the digits but neglect all the other signs. Like the string 'a2sdfda4' will be converted to '24'. Then from the example query on my own machine, it will first do the comparison of ``name=`id```, which will return false or 0 since those two fields do not equal, then it will do the comparison `0='a'`. Since by the rule of conversion, in the string 'a', letter 'a' will be ignored, so the final string will just be an empty string ''. When a string is compared with 0, will always return true or 1.

![](https://y4y.space/wp-content/uploads/2020/08/screenshot-from-2020-08-24-22-58-18.png?w=500)

So my final comparison result for my example query is 1, which will make the query like:

```sql
select * from user where 1;
```

And always evaluate as truth and return results.

Now, look back at the payloads used against the challenge. The first payload:

```sql
SELECT * FROM users WHERE username='michelle' AND password=`password`=1;
```

The first part of where clause is `username='michelle'`, which will return 1 since this user exists in the database, the first evaluation of the second comparison, which is ``password=`password```, will return true because both of those are the 'password' field in the table. And the last evaluation in the second comparison, `1=1`, you know it, is just 1. The final result is `1 AND 1`, which evaluated to be 1, and it will return a result!

Next, the second payload:

```sql
SELECT * FROM users WHERE username='michelle' AND password=`username`='michelle';
```

The first comparison requires no explanation, the second part though, is doing the ``password=`username```, which will return 0, since those two fields are not the same. Then it will proceed and do the second evaluation, `0='michelle'`. Remember that when a string is compared with 0, it always returns true or 1. Then the final result of this query is `1 AND 1`, which is 1, and thus this payload works, and authentication can be bypassed.

This is probably my biggest lesson from this CTF, other than that, I also learnt how to be patient, read source code carefully, be skeptical, and the most importantly, use IDA for reverse challenges.

That's it for this blog, hope you enjoy it. Next one should be some Jarvis OJ writeups.
{% endraw %}
