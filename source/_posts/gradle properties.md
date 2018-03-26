---
title: "gradle properties"
date: "2018-03-26"
categories: 
    - "ANDROID"
---

浏览器访问下载没有问题，但是Android studio下载一直失败。

```
https://dl.google.com/dl/android/maven2/com/android/tools/build/gradle/3.2.0-alpha07/gradle-3.2.0-alpha07.pom
```

查了代理设置

setting-http proxy -设置成 no proxy没有问题



折腾了后来发现，`gradle.properties`文件被设置了，删除掉代理设置，可gradle sync OK