@[toc]
# 前言
我们都知道，活动的启动方式有两种：一种是显示启动，或者很简单，指定一个活动的class就可以了；另外一种就是隐式启动，这种要指定action，category，data信息，例如我们在启动系统相机的时候。看一下代码：
```java
Intent intent = new Intent("android.media.action.IMAGE_CAPTURE");
                intent.putExtra(MediaStore.EXTRA_OUTPUT,imageUri);
                startActivityForResult(intent,1);
```
其中的"android.media.action.IMAGE_CAPTURE"就是相机的action，这样就可以启动相机了。
隐式启动我们在平时也用的比较少，对于自己应用中的Activity都是直接显示启动了。那什么时候用到隐式启动呢？一般是在启动别的应用的activity的时候，例如上面讲到的相机。
上面讲到的action，category，data就是intent-filer，也就是过滤器，筛选要启动的activity。
intentFiler有什么用？就像给自己上个标签。例如，你给自己上个标签是大学生，那么，当说学生出来，欸那么就匹配到你了。这个就是intentfiler的作用。用于筛选匹配。
那么这三个action，category，data究竟是什么？他们的具体匹配规则又是什么样的？上面讲到intentFiler是用于启动别的应用，有哪些常用的intentfiler可以使用？接下来我们就来看看。
# intentFiler的结构
前面讲到intentFiler包含三个：action，category，data，让我看一下代码熟悉一下：
```java
<intent-filter>
                <action android:name="huan"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
```
另外包括我们最熟悉的：
```java
<intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
```
这三个分别表示不同的意义。你想要启动什么样的activity就通过设置这些属性来启动到对应的activity。当我们自己设置intentFiler的时候也要注意他的意义性，虽然很多可以随便设置，但是就像变量名称一样，不要随便起。
# Action
action是最简单也是最常用的。
 - 意义：这个参数表示启动这个活动要干嘛。例如上面相机的是android.media.action.IMAGE_CAPTURE，很明显就是拍照功能。action的本质也是一个字符串，匹配就必须每个字符都一样，包括大小写。上面说过，虽然可以随便写这个字符串，但是要有意义。
 - 匹配规则：action的匹配规则也很简单，Intent中的action和intentFilter中的任意一个action匹配，那么匹配成功。但是如果Intent中的action是空的，那么匹配失败。
# Category
这个参数平时用得比较少，一般在一些比较特殊的情况才会用到
 - 意义：这个参数平常使用的意义是表示实现这个action动作的类别，也就是可以响应这个Intent的组件类别。例如上面的category android:name="android.intent.category.LAUNCHER"，表示这个action将会在顶级执行，什么意思呢？就是我们每次打开应用都会打开的第一个activity。
 - 匹配规则：可以设置多个category。但是intent中的每一个category都必须和intentFilter中的其中一条category匹配才能匹配成功。
 - 注意：给activity设置intentFilter的时候，如果没有其他的category，必须设置category android:name="android.intent.category.DEFAULT"这个category。原因是startActivity或者starActivityForResult这两个方法执行的时候，如果intent中没有category的话，那么就会自动加上"android.intent.category.DEFAULT"这个category。
# Data
data是三个中最复杂的一个，顾名思义，这个参数就是用来传递数据的。data不同于前面两个，他由两部分组成：Uri+mimeType.
很多读者可能还不怎么了解Uri这个东西，可以通过这个[Android URI总结](https://www.jianshu.com/p/7690d93bb1a1)简单了解一下Uri。这里就不展开了。我们先来看看data的组成：
```java
<data android:scheme=""
                    android:host=""
                    android:port=""
                    android:path=""
                    android:pathPattern=""
                    android:pathPrefix=""
                    android:mimeType=""/>
```
data一共由7个参数组成，一起来看看分别是什么意思：
 - scheme：这个表示uri的模式，有最熟悉的http：//这就是一种模式，另外安卓中还有比较常见的两种是：content：//和file：//。有学过ContentProvider的读者应该对content模式就很熟悉了。
 - host，port：host是主机，port是端口号，这两个合称authority。例如www.baidu.com这个应该就很熟悉了吧。在ContentProvider中表示哪一个contentProvider。
 - path，pathPattern，pathPrefix：这三个表示路径信息。一是完整的路径，二是可以用通配符来表示例如image/*，三是路径的前缀。
 - mimeType：这个表示媒体类型。例如image/jpeg

讲完他的结构后，有的读者可能会发现，这个data不就是一个地址+文件类型吗？是的，uri本身就是地址的意思。我们平时什么时候用到data呢？举个例子，我们调用相机拍照并存储到指定的文件夹，那么怎么让相机知道地址呢？就是data了，我们通过intent启动相机，并把地址放在data传输过去。这里的uri还涉及到安卓版本的影响有所不用，有兴趣的读者可以去了解一下。

那么，data的匹配规则是怎么样的呢？
和action是一样的，要求intent中必须要有data，而且和intentFilter中的一个相匹配就可以匹配成功。

 - 注意：如果在intentFilter中的data没有设置uri，那么默认的schme就是content和file。
# 设置intentFilter
看完了上面知道intentFilter中的三个参数怎么去匹配了，那怎么给活动设置intentFilter，怎么给intent传输参数知道吗？这个比较简单也简单讲一下：

给活动设置intentFilter比较简单，只要在AndroidManifest中设置就可以了，看示例代码：
```java
<activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

给intent设置参数也不难，一个一个来看：
 - action：可以在新建Intent对象的时候顺便写进去，例如：```
 Intent intent = new Intent("android.intent.action.GET_CONTENT"); ```或者调用Intent的setAction方法：```intent.setAction("android.intent.action.GET_CONTENT");```
  - caterogy：通过intent的方法```intent.addCategory();```
  - data：这个比较特殊一点因为他有两个部分：uri和mimeType。有三个方法：其中setType和setData分别是设置mimeType和uri的。但是这两个方法都分别会清空另一个的数据。什么意思呢？例如我通过setData设置了一个uri，然后再通过setType设置一个mimeType，那么第一个的uri就会不见了，被删除了。所以就有第三个方法：```intent.setDataAndType```。这个方法接受两个参数，uri和mimeType，同时设置两个参数，就不会被清除了。
# 常用的intentFiler
上面讲到intentFilter主要是用来启动别的应用的，例如相机，电话，那么有什么是比较常用的呢？具体可以查看这篇博客[android 常用URI 值得记住](https://blog.csdn.net/lo5sea/article/details/38308513)。不懂得也可以百度或者评论区留言。
# 小结
我们上面讲到intentFilter可以用来筛选要启动的activity，同样对于service和broadcast也是一样，也同样可以给他们设置intentFilter来隐式启动对应的组件。而平时用的最多还是隐式启动活动，特别是在调用别的应用的活动的时候。要掌握一些常见的调用，这也是很重要的。
同时intentFilter的匹配规则也是很重要，熟记才不会在自己设置intentFilter的时候出错。
其中还有很多细节没有讲清楚，有疑问的读者可以评论区留言。
·
·
·

### 参考资料
[Android URI总结](https://www.jianshu.com/p/7690d93bb1a1)
[ContentProvider和Uri详解](https://www.cnblogs.com/linjiqin/archive/2011/05/28/2061396.html)
[详解Intent](https://www.jianshu.com/p/67d99a82509b)
《Android开发艺术探索》任玉刚
