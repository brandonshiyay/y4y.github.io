---
layout: "post"
title: "File Inclusion and Directory Traversal, what files to look at? Linux Edition"
date: "2020-08-02 04:33:45"
date_gmt: "2020-08-02 04:33:45"
modified: "2020-08-02 04:33:45"
modified_gmt: "2020-08-02 04:33:45"
slug: "file-inclusion-and-directory-traversal-what-files-to-look-at-linux-edition"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/08/02/file-inclusion-and-directory-traversal-what-files-to-look-at-linux-edition/"
guid: "http://y4y.space/?p=118"
wordpress_id: 118
parent_id: 0
categories: ["Web2", "Web Security"]
---

{% raw %}
## Introduction

File inclusion and directory traversal is always chained together. Depends on the application those vulnerabilities can do different damages. From file disclosure to code execution.

## Methodology

I always check for file inclusion when I see those URLs: http://localhost/?page=home, or the parameter is file or filename, you get the idea. I first check if home.php exists, if it does, then great, if it doesn't, maybe it's in some other directories? Maybe do a directory bruteforcing? And of course, the first file to check is always /etc/passed, since on every UNIX machine, there is this file. Then depends on the context I have different approaches. If it just is a CTF problem where I want to read files like flag.txt, then I will just grab that file and see what's in there. But if I'm doing a boot2root type of challenge where the essential goal is to achieve code execution, I will look deeper.

### Log Poisoning

The best scenario for me, as an attacker is of course code execution, that's where the log poisoning attack comes in. But keep in mind that in most recent versions of Linux systems, log files are only accessible to users in root or adm group, where the user runs web services by default is www-data which is not in the groups above. That being said, do not rule out any possible options, I'd recommend still look for log files for the web services running. Like if it's apache then check for /var/logs/apache2/access.log or error.log. For more files to look at, go search payloadallthethings github and he has many interesting files to look at.

If by any chances, log files are accessible to the user, code execution can be achieved by modifying user agent in the HTTP request. It can be done using burp or curl. Change user agent to something like:

```php
<?php system($_REQUEST['c']);?>
```

The code above takes c as parameter and execute shell command. The example URL should look like this:
http://localhost/?file=/var/log/apache2/access.log&c=id
I don't fully understand why it has to be this way, I think 'c' is the second parameter you want to pass to the file so '&' needs to be used.

### PHP Filters

If log poisoning is not possible, I would think about seeing some source codes, maybe I can get database credentials or some other hard coded credentials. In  PHP applications that's quite hard tho, because when a php file is included, web servers will automatically execute the code inside the php file. Here is when php filters comes in handy. I usually like to use base64 encoding, but I believe plain text should also work.

Let's say the URL http://localhost/?file= is vulnerable to file inclusion, and I want to see some source code using PHP filters. The syntax for base64 encoded PHP filter should look like this:
http://localhost/?file=php://filter/convert.base64-encode/resource=index.php.

If such filter is enabled, you can get a string of base64 encoded texts. But unfortunately, sometimes web servers does not have this feature enabled. If you checked you syntax is correct and there is no text, just move on.

### Remote File Inclusion

Not likely to happen, but there is always a chance. Really nothing too much to talk about, because in RFI you can control the file to include, so it's like a instant RCE.

### Enumerate File System

If all the above methods don't work or not applicable, I always try to enumerate what other interesting file I can use later. For instance, if there is directory traversal vulnerability in a FTP or SMB server, I will search for the entire file system to see if there are any credential files. Directory traversal in those services will normally let you list files, so it's not blind. In PHP applications tho, most likely it will be blind, at least that's what I think.

If there is a FTP or SMB service running but you cannot access it, maybe try to look for configuration files. By default vsftpd's configuration is /etc/vsftpd.conf, samba's is /etc/samba/smb.conf. Go check those files out and see if anything new can be found.

Otherwise maybe try to enumerate system information:
/etc/lsb-release tells the version of system on ubuntu.
/etc/passwd needs no introduction.
/proc/self/cmdline lets you see the command line of the current process.
/proc/version also tells the system version
/proc/self/environ shows the environment variables for the current process.
/proc/net/tcp shows the system tcp connections, but it's all in raw hex.
/proc/net/udp show the system udp connections, also in raw hex.
/proc/sched_debug shows the processes running in the system, if there is some hidden service, it can be useful.
/proc/self/mmap shows the virtual memory used by each process.

There are many other files in /proc but I usually look for those ones. Also note some files in /proc are dynamic generated, so if you are doing those in a HTTP request, chances are server will not send back those files because the server does not know how large the file is. There is a HTTP header called 'Range', which specifies the range of bytes of request. It can be useful when dealing with those dynamic generated files. The example Ranger header should look like this:
Range: bytes=0-4096
The header above is saying send the file from 0 byte to 4096 bytes. This can solve the dynamic generated file problem.

## Conclusion

Not just file inclusion and directory traversal, XXE can also lead to file disclosure, but I believe XXE can list files? Sometimes it can, I am sure of that.

Maybe I will update this when I learn more about Linux systems, I don't know what files to look at on Windows systems and I don't really want to know either.
{% endraw %}
