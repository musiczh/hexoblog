@[toc]
# 前言
数据库操作一直都是比较繁琐而且单一的东西，平时开发中数据库也很常见。有学过mysql的读者可能会觉得sql语句确实让人很难受。同样android中，虽然有内置数据库SQLite，但是操作起来还是非常的不方便。跟网络请求类似，当我们用原生的HttpURLConnection请求数据再用json解析，过程很繁琐，所以我们一般是封装成一个工具类，但是retrofit出现了，他帮我们解决了网络请求和解析数据的封装，同时还支持RxJava的异步，十分强大。不了解retrofit的读者也建议你们去学习一下retrofit确实非常好用。LitePal也是同样的道理，把创建数据库和增删查改等等操作都封装起来，所以我们用起来会非常的方便。同时还支持异步操作，不需要我们自己去开启子线程，代码非常的整洁，简单。那接下来就来看看这个神奇的框架LitePal。
# 简述映射
LitePal是采用映射的方式来把数据存储在数据库中的，和GSON的道理是一样的。例如我们现在有一个类，这个类必须是javaBean类：
```java
public class Student extends LitePalSupport {
    private String name;
    private int age;
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```
那么他在数据库中就会有一个表，这个表有三列：id，name和age，id是自动生成的，这样就可以理解映射了吧。所以我们使用LitePal的时候不用去指定每一列是什么，只需要给他一个Bean类，自动就会生成了。

# 配置LitePal
LitePal使用之前需要先配置一下，一共分为两步：
1. 添加依赖库：在app/build.gradle中添加如下内容：
```java
dependencies {
    implementation 'org.litepal.android:java:3.0.0'
}
```
其中3.0.0是版本号，写这个文章的时候是3.0，他更新也是很快的，读者可以自行到文末进入官网查询最新的版本号。添加完之后sync一下就行了。
2. 修改AndroidManifest中的代码：添加一句android:name="org.litepal.LitePalApplication"：
```java
<application
        android:name="org.litepal.LitePalApplication"
        ...
<application
  
```

添加这句的意思是让启动app的时候会自动实例化LitePalApplication这个类供给LitePal这个框架使用。如果有自己写了一个android：name的，那么只需要添加这一句LitePal.initialize(context);就可以了。其中的context参数为全局app的context。例如：
```java
public class myApplication extends Application {
    private static Context context;

    @Override
    public void onCreate() {
        super.onCreate();
        context = getApplicationContext();
        LitePal.initialize(context);
    }
     
}
```
3. 在main目录下创建一个Directory：assets。然后再assets目录下再创建一个litepal.xml，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190921221538303.png)
4. 编辑litepal.xml中的内容：
```java
<?xml version="1.0" encoding="utf-8"?>
<litepal>
    <dbname value="bookStore"/>
    <version value="1"/>

    <list>     
    </list>

</litepal>
```
dbname就是数据库的名字，version是数据库的版本，list中是数据库中的表，可以在这里添加，怎么添加后面会讲到。

# CRUD操作
常规增删查改操作，但是在这个框架下都显得特别的简单。
## 增加表和数据
1. 例如我们现在要在数据库中创建一个学生的表，首先要创建一个学生的类,再让他继承LitePalSupport类，至于为什么下面会讲到：
```java
public class Student extends LitePalSupport {
    private String name;
    private int age;
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```
2. 然后在刚才的litepal.xml中添加表：
```java
 <list>
        <mapping class="com.example.myapplication.Student"/>
 </list>

```
这里的class要是你的类的真实目录，视具体情况而定。
添加其他的表也是同样的道理。
3. 调用student对象的save()方法：
```java
Student student = new Student();
                student.setAge(12);
                student.setName("hha");
                student.save();
```
这里的save方法就是继承前面的LitePalSuppport类的，调用这个方法后就会自动添加到库中对应的表中的一行。
添加其他行数据也是同样的道理
## 更改表结构
更新表的列。例如前面的学生类是name和age，但是如果你想要增加一个studentId，可以很简单地实现。具体操作如下：
1. 首先更改你的bean类，想怎么改就怎么改
2. 在litepal.xml中更改版本号增加1.例如：
```java
<?xml version="1.0" encoding="utf-8"?>
<litepal>
    <dbname value="bookStore"/>
    <version value="2"/>
    <list>
            <mapping class="com.example.myapplication.Student"/>
    </list>

</litepal>
```
把他改成2就行了。
## 删除数据
删除数据也很简单，有两种删除方法，一种是指定行删除，一种给个约束条件删除。
1. 删除单行：
LitePal.delete(Student.class , id);
2. 约束条件：
LitePal.deleteAll(Student.class, "age > ?" , "12");
指定约束条件删除，？是占位符会把后面的12放进去。
如果只传入一个Student.class，那么就会把整个表的数据都删除了

## 查询数据
查询数据的接口都会返回一个List，每一行对应一个对象。所以是LitePal把数据解析都给我们做好了，我们直接拿对象使用就ok了。这里有几种方法接口都看一下：
1. LitePal.findAll(Student.class,id);查询对应表的对应行，如果没有传入id参数，就返回这个表的所有内容。同样findFirst是返回第一行，findLast是返回最后一行。
2. 查询的内容还可以进行筛选，这里就用到几个方法：
 - select（）对应查哪几列的内容
 - where（）查询的约束条件
 - order（）排序方式
 - limit（）指定查询的数量
 - offset（）指定结果的偏移量。这个可能比较难理解，举个例子：假设你查的id是1，但是你设置了偏移量是1，那么返回的就是第二行的数据。
 最后举一个综合例子演示一下：
 ```java
List<Song> songs = LitePal.where("name like ? and duration < ?", "song%", "200")
										.order("duration")
										.select("name")
										.limit(3)
										.offset(3)
										.find(Song.class);

```
这样就可以查询到对应的数据了。
# 异步操作
有时候如果我们的数据库中的内容很多，涉及到重量级的数据库操作往往是比较费时的，那么这个时候肯定时不能放在主线程去进行操作的，这样会造成系统卡死。那么我们就需要去把这个操作放在子线程中。LitePal早就为我们考虑到这个问题了，所以也增加了异步操作，轻松实现，来看看怎么用吧。
先看个例子：
```java
LitePal.findAllAsync(Song.class).listen(new FindMultiCallback<Song>() {
    @Override
    public void onFinish(List<Song> allSongs) {
    
    }
});
```
这是在官网中的例子，要注意的两个点
 - 用findAllAsync代替findAll方法
 - 添加listen方法，并新建匿名类FindMultiCallback<>()作为参数，重写里面的onFinish方法即可
这样获取完数据后就会执行onFinish方法了
轻松实现异步操作。同样这个可以结合上面的数据筛选。
# 创建多个数据库
如果你一个数据库不够用，想要创建多个数据库，当然也是可以的，看代码：
```java
LitePalDB litePalDB = new LitePalDB("demo2", 1);
litePalDB.addClassName(Singer.class.getName());
LitePal.use(litePalDB);
```
这里就创建了一个库叫做demo2，并增加了一个表：Singer。最后执行LitePal.use方法来启用这个库。这样的话就默认使用这个库了。对象的save方法都会执行到这个库中
如果想切回到litepal.xml中的那个库，可以用下面的方法：
LitePal.useDefault();

如果想删除一个库（删库跑路可能会被乱棒打死）
LitePal.deleteDatabase("demo2");
是不是很简单？
# 监听数据库创建或者升级
当数据库创建或者升级的时候都会调用下面的两个方法：
```java
LitePal.registerDatabaseListener(new DatabaseListener() {
    @Override
    public void onCreate() {
    	// fill some initial data
    }

    @Override
    public void onUpgrade(int oldVersion, int newVersion) {
    	// upgrade data in db
    }
});
```
可以在里面写要执行的逻辑。
# 总结
LitePal这个库确实是非常的强大，把很复杂的数据库操作都简化成了一个个的方法。但是更新很快，需要时刻看着他更新的内容，有可能会换API，所以建议大家多去官网学习。

### 参考资料
·
·
·
《第一行代码》 郭霖
LitePal官网：[LitePal项目地址](https://github.com/LitePalFramework/LitePal)

