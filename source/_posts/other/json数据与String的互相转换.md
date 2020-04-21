---
title: json数据与String的互相转换 #标题
date: 2020/4/21 00:00:00 #建立日期
updated: 2020/4/21 17:00:00 #更新日期
comments: true #开启评论
tags:  #标签
- android 
- json

categories:  #分类
- Android
---





﻿json数据本质上也是字符串，所以他们之间的转换也是比较容易的，记住方法和需要注意的事项就行了。

#### 字符串转json
在构造json的对象时候把string对象传进去即可。看例子
```java
String data = "{
    "result":"success",
    "message":null
    }";
try {
	JSONObject jsonObect = new JSONObject(data);
} catch (JSONException e){
    e.printStackTrace();
} catch(NullPointerException e){
	e.printStackTrace();
}
```

这里建立jsonObject对象的时候因为不确定该字符串是否符合json规范，如果不符合规范就会抛出JSONException异常，而如果该字符串是null的时候就会抛出空指针异常。这里也可以判断一下字符串是否为空防止空指针异常。


#### json数据转字符串
这个就比较容易了，直接调用jsonObject对象的toString方法即可。看代码
```java
//这里的jsonObject是上文的JSONObject对象
String s = jsonObect.toString();
```
