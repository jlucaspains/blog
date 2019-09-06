---
layout: post
title: "Simple concurrency tester"
date: 2019-09-06
comments: true
sharing: true
categories: [util]
description: 
---

You probably know how hard it is to purposefully cause a concurrency issue so you can test your code in these edge scenarios. I ran into this issue not too long ago and found that my favorite web service tester [Postman](https://www.getpostman.com/) doesn't support sending multiple messages concurrently. I'm sure there are plenty of tools out there that do a great job, but for the simplest possible way you can use a combination of cURL and scripting. Here is an example using Windows, cURL and cmd script.

```cmd
ECHO you can use set VARIABLE_NAME and %variable_name% to set/read variables
ECHO you can do anything curl supports. The example below sends headers (including authentication) and a json body
ECHO in CMD, the ``start /b`` option creates a new cmd process without a window and runs curl in it
ECHO in my tests, the messages arrived with as little as 2ms time difference

set Auth_Token="NotAValidToken"
set URL="https://yourserver.com/some/api/path"

start /b curl -X POST "%URL%" -H "accept: application/json" -H "authorization: Bearer %Auth_Token%" -H "Content-Type: application/json" -d "{payload:\"payload 1\"}"
start /b curl -X POST "%URL%" -H "accept: application/json" -H "authorization: Bearer %Auth_Token%" -H "Content-Type: application/json" -d "{payload:\"payload 2\"}"
```

This is very simple. In fact, too simple to handle anything other than debug. But hey, I'm a developer! ;)

Cheers!