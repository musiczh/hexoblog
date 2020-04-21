---

title: Activity启动模式简析 #标题
date: 2020/4/21 00:00:00 #建立日期
updated: 2020/4/21 17:00:00 #更新日期
comments: true #开启评论
tags:  #标签
 - android 

categories:  #分类
 - Android

---



# 前言

平常我们启动活动的时候就是直接startActivity或许并没有注意活动的启动模式，默认情况下都是以默认的启动模式启动。但启动模式有时候是比较重要的。例如一个活动你想他只启动一次不要有多个实例，那么你可能需要把他设置为singleTask模式。所以有必要了解一下这一些启动模式。同时要注意一下，启动模式≠启动方式，启动方式是指显示启动和隐式启动，不要混淆，显示启动和隐式启动后续我会有专门的文章讲解。
# 关于任务栈简介
要了解启动模式，首先要了解一下关于任务栈的概念。关于任务栈的实现原理等我在这里就先不说了，这里主要简单介绍一下什么是任务栈。我们启动的活动实例都会放在一个叫做任务栈的东西里面。我们都知道栈是“后进先出”的特点。打个比方，任务栈就是一个羽毛球筒，活动实例就是一个个羽毛球，后放进去的只能先拿出来。所以当我们启动一个app的时候，就会自动创建一个任务栈，然后我们就往里面丢活动实例。当我们按返回销毁活动的时候，这些活动就依次从任务栈里面出来。当然，一个app可以拥有多个任务栈，例如使用singleInstence启动的活动就是在一个独立的任务栈中。了解完任务栈的概念，接下来就可以来看看活动的四种启动模式。
# 解析Activity的四种启动模式
### standard
这种是标准启动模式，默认就是这种启动模式。每次启动这种启动模式的活动的时候都会创建一个新的实例放入栈中，不管栈中是否已经存在相同的实例。这也是最容易理解的。
### singleTop
顾名思义，栈顶是单一实例的。什么意思呢。假设你现在启动一个ActivityA，但是这个时候已经存在一个ActivityA实例在栈顶，那么这个时候，就不会创建新的实例。但是如果，在非栈顶存在相同的实例，还是会创建新的实例的。例如，现在栈中的活动是 ABC，A处于栈顶。然后此时启动A，是不会再创建一个A活动出来，而是执行A的onNewIntent方法；但是如果此时启动C活动，由于栈顶是A不是C，那么还是会创建一个新的C实例出来，此时的栈情况就是CABC。
### singleTask
单一任务模式。这个模式的意思是，在该活动的启动栈中，只能存在单一实例，不管是否位于栈顶。与其他启动模式不同的是，这个启动模式可以指定栈去启动。例如现在有一个栈Main，但是你可以给活动A指定一个栈名dev，那么启动A的时候就会创建一个栈叫做dev。所以singleTask的意思就是，当你启动一个启动模式为singleTask的活动的时候，如果栈中没有相同的实例，那么就会创建一个新的实例放入栈中；如果指定栈中存在相同的实例，例如栈中有ABC，然后你启动B，那么这个时候不会去创建新的B实例，而是把B放到栈顶，并把A顶出去，再执行B的onNewIntent方法，此时栈的情况就是BC。
细心的读者会发现“顶出去”。是的，我们都知道栈是后进先出的特点，例如你往筒里放了3个羽毛球，那你想要拿到中间那个羽毛球，是不是只能先把上面那个抽出来呢，同样的道理，要想把B提到栈顶，那么必须把A顶出来。可能会有很多读者误以为启动后是BAC，但其实是BC，因为A得先出栈，B才能出来。同理，如果栈中是ADFBC，这个启动B，也是BC，上面的全部被出栈了。
### singleInstance
单例模式。这个是singleTask的强化版本。他会自己新建一个栈并把这个新的实例放进去，而且这个栈只能放这个活动实例。所以当重复启动这个活动的时候，只要他存在，都是调用这个活动onNewIntent方法并切换到这个栈中，并不会去创建新的实例。
# 设置启动模式的两种方法
了解了活动的四种启动模式，接下来看看如何给他指定启动模式。
### 静态设置
静态设置就是在AndroidManifest中给具体活动设置启动模式。通过给活动指定launchMode参数来设置启动模式。例如：
```java
 <activity android:name=".MainActivity"
            android:launchMode="singleInstance"/>
```
### 动态设置
动态设置是在启动活动的时候再指定启动模式，例如：
```java
Intent intent = new Intent();
intent.setClass(this,SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```
可以看到我们通过intent.addFlags这个方法来指定启动模式，这个方法传入一个参数来指定启动模式，其他的参数有：
 - FLAG_ACTIVITY_NEW_TASK：singleTask模式
 - FLAG_ACTIVITY_SINGLE_TOP：singleTop模式
 - FLAG_ACTIVITY_CLEAR_TOP:清除该活动上方的所有活动。一般和singleTask一起使用。但是如果你的启动模式是standard，那么这个活动连他之上的所有活动都会被出栈再创建一个新的实例放进去。例如现在栈中是ABCD，以FLAG_ACTIVITY_CLEAR_TOP+standard模式启动C的时候，首先清理掉ABC，是的，C也会被清理，然后再创建一个新的C放进去，执行之后就是CD。
# 特别注意的坑
### singleInstance返回任务栈
现在模拟一个场景：现在有三个活动 A,B,C。A和C的启动模式都是standard，B的启动模式是singleInstance。先启动A，再启动B，然后再启动C。这个时候问题来了，如果我这个时候按下返回键，是回到B吗？答案是回到A。再按一下呢，返回桌面吗？答案是回到B，再按一下再回到桌面。其实不难理解。我们都知道singleInstance会创建一个独立的栈，当我们启动A的时候，A位于栈First中，启动B的时候，就会创建一个栈Second并把B实例放进去。这个时候再启动C，就会切换到栈FIrst，因为singleInstance创建的栈只能放一个，所以C会放到栈First中，当按下返回的时候，栈First中的活动就会依次出栈，直到全部出完，才会切换到栈Second中。所以要注意这个点。
### singleTask多任务栈启动问题
这个问题和上面singleTop的本质是一样的。模拟一个场景：现在有两个栈：First：ABC；Second：QWE。栈First位于前台，栈Second位于后台。A位于栈顶。这个时候以singleTask的模式启动W，会发生什么样的情况呢？首先会切换到栈Second，再把Q出栈，W提到栈顶，并执行W的onNewIntent方法。这个时候按返回键就会把Second栈中的活动依次出栈，全部出完后才会切换到栈First。
### singleTask的TaskAffinity与allowTaskReparenting参数
前面我们讲到给singleTask模式指定要启动的任务栈的名字，怎么指定呢？可以在AndroidManifest中指定相关的属性，如下：
```java
<activity android:name=".Main2Activity"
            android:launchMode="singleTask"
            android:taskAffinity="com.huan"
            android:allowTaskReparenting="true"/>
```
这里解释一下这两个参数
 - taskAffinity：指定任务栈的名字。默认的任务栈是包名，所以不能以包名来命名。
 - allowTaskReparenting：这个参数表示可不可以切换到新的任务栈，通常设置为true并和上面的参数一起使用。
 我前面讲到可以给singleTask的活动指定一个栈名，然后启动的时候，就会切换到那个栈，并把新的活动放进去。但是如果设置allowTaskReparenting参数为false的话是不会切换到新的栈的。这个参数的意思是可不可以把新的活动转移到新的任务栈。简单点来说：当我们启动一个singleTask活动的时候，这个活动还是留在启动他的活动的栈中的。但是我们指定了taskAffinity这个参数，或者启动的活动是别的应用中的活动，那么就会创建一个新的任务栈。如果allowTaskReparenting这个参数是true的话，那么这个活动就会放到那个新的任务栈中。这样应该就可以明白了。所以这两个经常是配套一起使用的。

# 总结
 活动的启动模式有四种，每种的功能都不一样，可以结合具体需要去使用，但是最重点还是要了解他的实现原理，栈中是怎么变化的，这个是比较重要的。了解这个之后那些特殊情况也就很容易理解了。
 上面我讲的只是简单的使用，关于活动启动模式还有很多要了解。后续可能会解析一下，读者也可以自行去深度了解。

### 参考资料
《Android开发艺术探索》 --任玉刚
