---

title: AndroidStudio_unable to_find_valid_certification报错的解决方法 	#标题
date: 2020/7/22 00:00:00 	#建立日期
summary: 					#文章摘要
tags: 						#标签
 - android 
categories:  				#分类
 - Android
updated: 					#更新日期
keywords:				#关键词
description:				#文章描述
top_img:					#文章顶部照片
comments: true				#是否显示评论模块
cover:						#文章缩略图
toc: true							#是否显示toc
toc_number: true			#是否显示toc_number
auto_open: true				#是否自动打开toc
copyright: true					#显示文章版权模块
copyright_author: 一只修仙的猿		#文章版权作者
copyright_author_href: 			#文章版权作者链接
copyright_url:						#文章版权文章链接
copyright_info:						#文章版权声明文字
mathjax:
katex:
aplayer:
highlight_shrink: true       #代码框是否打开

---



### bug来源

今天更新了AndroidStudio最新版3.5，然后出现了一个bug，报错是：ERROR：Cause: unable to find valid certification path to requested target。确实这个报错弄了我好久的时间。虽然我到现在还不知道究竟里面是哪个源头出现了问题，经过一番百度去询问，也解决了问题。同时也知道别人也出现了相同的问题，不过处理的方法不同，下面提供几种方法给你们参考一下。

### 解决方法

#### 方法一

这个也是我自己的解决方法，最初是看到了AS这个推荐更新：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012211911236.png)
然后我就点开了：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191012212026934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc2Njc1Mw==,size_16,color_FFFFFF,t_70)
我靠，strongly recommend，那还不赶紧更新！！
然后，，就出错了。具体原因也不知道，但是改一个地方就可以了：
在项目的build.gradle文件夹下：
```java
dependencies {
        classpath 'com.android.tools.build:gradle:3.5.1'
        }
```
把这里的3.5.1改成3.5.0就好了，就像这样：
```java
dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        }
```

这里改完之后就重新build一下。

#### 方法二

经过了方法一还是不行的话，在方法一的基础上，就清空缓存Restart一下。怎么做呢，首先点开顶部菜单栏的file：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026202345290.png)
然后再选择里面的Invalidate cache
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026202429863.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzc2Njc1Mw==,size_16,color_FFFFFF,t_70)
然后就Invalidate and restart就可以了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191026202453553.png)
然后编译器就会自动重启，等待重启就可以了。
如果重启后出现无法创建Activity的情况，就需要重新sync一下项目：
还是点开顶部File菜单，再点击下面这个：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191029155720778.png)
然后静等完成就好了。如果出现了一些build的问题再解决就好了。
#### 方法三

还是改项目文件目录下的build.gradle
```java
buildscript {
    repositories {
        google()
        jcenter{ url "http://jcenter.bintray.com/"}

    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        mavenCentral()
        maven { url 'https://maven.google.com' }
        google()
        jcenter()
        maven { url "https://jitpack.io" }
    }
}
```
把缺少的代码加上去就好了，然后就等他慢慢下载

#### 方法四

这个方法献上一些博主的解决方法，上面的方法解决不了的话看看这些：
[彻底解决unable to find valid certification path to requested target](https://blog.csdn.net/Gabriel576282253/article/details/81531746)

[解决 Android Studio Error:Cause: unable to find valid certification path to requested target](https://blog.csdn.net/twilightdream/article/details/82760296)

[Android studio :Error:Cause: unable to find valid certification path to requested target](https://www.jianshu.com/p/0fd7ed2ffe82)

[Android Studio出现:Cause: unable to find valid certification path to requested target](https://blog.csdn.net/qq_17827627/article/details/99404177)

#### 方法五

方法五的核心解决点就是网络。网上有很多的去让你下载证书之类的。说实话我没有试过因为我觉得太麻烦了（没错我就是这么懒）。如果你是个大学生，那么请**不要用校园网**，然后重新build一下，因为校园网对于外网进行了一些域名的限制。
现在我也经常会遇到这个问题，这里推荐一个佛系法：不管他。不管他没办法运行啊？只需要不断地重新sync，也就是这个按钮：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200722172135393.png)

然后大概6次左右就会好了（10次以上还没解决，就重启一下吧）（人类迷惑，知道为什么的小伙伴留言告诉我一下）。
按照这个证书是网络问题，那就解决网络问题：科学上网。是的，现在AS特别的智能，基本上科学上网他可以给你解决很多的问题，但是因为国内某些众人皆知的原因，无法联网，所以科学上网一下，就可以了。怎么科学上网老司机们不要多教了吧。（本来想给你们推荐，但是，，，，有技术问题可以私信交流哈）（狗头保命）



### 小结

截止更新目前我的AS版本是4.0，目前可以顺利运行。
这个问题不同的电脑可能解决的方法不一样，上面也是我总结了几个小伙伴出来的结果，如果一种不行的话可以几种一起使用。有问题可以留言区问。
另外有好的解决方法欢迎评论区提供。
最后祝各位读者debug愉快~