@[toc]
# 关于序列化
这篇文章主要来讲一下在安卓中关于序列化的问题。首先了解一个问题：什么是序列化？为什么要用到序列化？
 - 什么是序列化：序列化就是就是把一个对象变成可传输的二进制流，可以进行传输。
 - 什么是反序列化：与序列化对应，反序列化就是把一个二进制流转化成对象。
 - 哪里用到序列化：上面说到序列化就是把对象变得可传输；例如在内存，或者网络中传输数据的时候，就得把一个对象变成二进制流可以进行传输。除此之外，在各种通信中，例如进程间通信，文件读取写入等等都要用到序列化。涉及到数据传输，就得使用序列化。因为只有二进制流才可以进行传输。
 - 我们在那些地方会遇到序列化：我们会发现，仅有基本数据类型可以自动进行序列化，但是我的自定义对象并不可以进行序列化。在哪里可以体现呢？当我们从一个Activity向另外一个Activity传递数据的时候，通过Intent，我们会发现只能放基本数据类型。
 - 怎么让自定义的对象可序列化：就是我们要讲的Serializable和Parcelable接口。这两个接口就可以让我们的自定义对象可序列化。
 
 那这两个接口怎么使用？他们有什么区别？这就是接下来我要讲的。
# Serializable
#### 简述原理
Serializable只要使用Java的ObjectOutputStream与ObjectInputStream来开启流。而这个接口主要就是用来标识这个对象可以被转换。关于IO流的相关知识，读者有兴趣可以去深入了解，这里不做深入探究。

#### 怎么使用
这个序列化接口是Java提供的，实现了这个接口的类，其实例就可以进行传输。那么这个接口怎么使用呢？
直接实现这个接口就行了。
因为这是一个空接口，所以使用方法极其简单，只需要实现这个接口即可。接下来看个例子：
```java
public class Test implements Serializable {
    private String string1;
    private String string2;
    private int num;
    private Son son;
    private List<Son> list = new ArrayList<>();

    public String getString1() {
        return string1;
    }
    public void setString1(String string1) {
        this.string1 = string1;
    }
    public String getString2() {
        return string2;
    }

    public void setString2(String string2) {
        this.string2 = string2;
    }

    public int getNum() {
        return num;
    }
    public void setNum(int num) {
        this.num = num;
    }
    public Son getSon() {
        return son;
    }
    public void setSon(Son son) {
        this.son = son;
    }
    public List<Son> getList() {
        return list;
    }
    public void setList(List<Son> list) {
        this.list = list;
    }
}
```
这样就可以把这个Test对象序列化了。
# Parcelable
同样是使对象可序列化，同样是继承接口，但是Parcelable相比Serializable就要复杂得多了。Paecelable需要在要序列化的类中利用Parcel重写序列化和反序列化的方法。
#### Parcelable的实现原理
Parcelable的实现原理主要就是在内存中开辟一个共享内存块，然后进行传输，看图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017234934209.png)
如图在AB两个进程中要进行传输，不能直接把一个对象传输过去，开辟一块内存，通过Parcel，也就是打包的方式，把数据存在了共享内存，再把共享内存中的数据用Parcel取出来，实现数据传输。通俗点来讲，就像送外卖一样，商家找了外卖这个中间人，把食物打包给了外卖小哥，然后你再找外卖小哥拿外卖，再拆包就可以拿到食物了。
通过原理可以看到，实现这个接口的重点就在：装包，拆包。如果将对象里面的数据，一个个装起来，是我们要在类中实现的方法。而装拆包公司就是Parcel。所以首先来了解一下这个包装公司：Parcel！

#### “包装公司”Parcel
Parcel的使用和Intent是差不多的，就是write（put）进去，再read（get）出来。简单来看看怎么使用：
###### 装包
直接上代码，相关方法说明看注释：
```java
		Parcel parcel;
		
		//打包字符串
		parcel.writeString(String string);
		
		//打包整型数
        parcel.writeInt(int num);
        
        //打包布尔变量，要和bite进行转换
        parcel.writeByte((byte) (boolean bl? 1 : 0));
        
        //打包字符串集合
        parcel.writeStringList(List<String> list);
        
         //ElemType表示一个自定义类
        //序列化对象的时候传入要序列化的对象和一个flag,这里的flag的意思是是否要将这个作为返回值返回
        //一般传入0，要作为返回值就写1。另外这个对象也一定要是可序列化的
        parcel.writeParcelable(ElemType elem, 0);
        
   		//这里写入自定义元素集合有两种方法，区别是是否写入类的信息
   		//第一种方法是不写入类信息，相对应的取出来的时候要用Parcel把元素里面的每一个元素拿出来
        parcel.writeTypedList(mFriends);
        //第二种方法是写入类的信息，取出来的时候就要用类加载器去加载
        parcel.writeList(mFriends);
        //同样也要保证集合的元素可序列化
```
这里可以看到整体和Intent的传入数据是十分像的。重点有三个地方不太一样：
 - 布尔变量。布尔变量不能直接放进去，要转换成bite
 - 可序列化对象。这个放进去的时候要保证这个对象是可序列化的，也就是必须实现序列化接口
 - 集合。集合分为两种。一种是把元素类的信息也放进去，一种是不把元素的类的信息放进去。下面将拆包会详细讲一下。
 


###### 拆包
直接上代码：（变量使用装包中的变量）
```java
Parcel source;//这个对象在反序列化的时候会作为参数传入供给使用

		//获取字符串
		String string= source.readString();
		
		//获取整型
        int i = source.readInt();

		//获取布尔数
        boolean b = source.readByte() != 0;//这里注意转化

		//获取自定义对象
		// 读取对象需要提供一个类加载器去读取,因为写入的时候写入了类的相关信息
		ElemType e = source.readParcelable(ElemType.class.getClassLoader()); 

		//获取字符串集合
        List<String> list = source.createStringArrayList();
        
        
        //这一类需要用相应的类加载器去获取
        //ElemType表示一个自定义类
        source.readList(List<Elemtype> list, ElemType.class.getClassLoader());
        //这一类需要使用自定义类的CREATOR,也就是使用Pacel的反序列方法去获取
        //source.readTypedList(ElemType e, ElemType.CREATOR); //对应writeTypeList
        //books = in.createTypedArrayList(ElemType.CREATOR); //对应writeTypeList
```
主要就是对于布尔变量以及集合的转换会比较不一样。
代码中的Friend.CREATOR是Parcelable接口中反序列的方法，实现原理也是用Parcel把对象里面的数据一个个转换出来。
 - 布尔变量要进行转换
 - 对象要用类加载器去加载。因为写入的时候写入了类信息
 - 集合看情况；写入了类信息就要用类加载器；否则用反序列的方法。

###### 小结
Parcel的使用方法和Intent很像，但是有几个点要注意一下：
 - Parcel中没有布尔变量，所以必须将布尔变量和bite进行转换
 - Parcel包装自定义对象时，这个对象对应的类必须也得是实现了序列化接口的才可以进行打包。装包的这个方法因为写入了类的信息，所以拆包的时候需要用到对应类的加载器来进行加载。
 - Parcel在包装集合的时候有两种方法：一种是写入类的信息，一种是不写入类的信息。但都有一个前提：集合中的元素必须是可序列化的。第一种方法需要用到类加载器去加载，第二种就需要用到自定义类中的Parcel的反序列化方法。
 - 对于类中有多个相同类型的数据，例如3个字符串，3个int等等，存入的顺序和取出来的顺序一定要一致。例如存int a，int b；那么取出来的时候也得是int a，int b。不能颠倒。
 
 #### Parcelable的使用
 看完了上面的Pacel可能还是有一点懵，那这个怎么Parcelable怎么用Pacel实现呢？Parcelable的主要实现思路就是把一个对象中的数据进行分解，分解出来的数据都是可以用Parcel打包的。Parcelable具体怎么使用呢？接下来就来看一下。先看看谷歌给出的示例（代码很重要，记得看注释）：
 
 ```java
 //首先要实现Parcelable接口
 public class MyParcelable implements Parcelable {
 	//类中的数据
     private int mData;
 
 	//接下来这两个方法是关于序列化的。一定要重写的两个方法
 	//第一个方法是内容描述。一般没什么要求都是返回0就行了
     public int describeContents() {
         return 0;
     }
     
     //第二个是序列化数据的方法。当我们序列化这个类的对象的时候就会调用到这个方法
     //两个参数：一个是Pacel“包装公司”上面我们讲的，一个是flags。第二个参数表示是否把数据返回。一般为0.
     //利用这个Parcel把类中所有的数据“打包”序列化
     public void writeToParcel(Parcel out, int flags) {
     	//这个就是上面我们介绍的Parcel的使用
          out.writeInt(mData);
    	}
 

	//接下来的这两个就是关于反序列化的了
	//CREATOR是Parcelable中的一个静态变量，这个名字不能改，因为他内部调用的时候就是用CREATOR来调用的改了名字就找不到了
	//观察我们上面的Pacel使用方法也可以发现，是传入ElemType.CREATOR。如果改了名字就找不到了。
	//Creator对象中有两个方法要重写：
      public static final Creator<MyParceable> CREATOR
              = new Creator<MyParceable>() {
              
           //这个就是反序列方法了。在Pacel中把数据取出来构建成一个MyParcelable对象
          public MyParcelable createFromParcel(Parcel in) {
          	//这里我们用到构造器来获取这个对象。具体构造器实现方法在下面
              return new MyParcelable(in);
          }
 
 		//这个是开辟数组供给使用。一般按照下面的格式写就好了。
          public MyParcelable[] newArray(int size) {
              return new MyParcelable[size];
          }
     };
    //这个是构造器。从Parcel中把数据取出来  
      private MyParcelable(Parcel in) {
      	//这里就用到了上面我们讲的Parcel的获取数据的方法了
          mData = in.readInt();
      }

```
通过上面的代码相信也可以很清晰的了解怎么实现了。主要就是分为两个部分：序列化和反序列化。
 - 序列化主要就是用一个Parcel把数据封装起来。怎么使用按照上面的Parcel使用方法。
 - 反序列化要用到一个静态变量，这也可以说是一个构造器，Creator本身就是创造的意思嘛。通过这个Creator吧Parcel中的数据取出来再构造成一个对象，对应拆包。具体我们使用构造器来实现。构造器里面用Parcel的获取数据方法来把数据取出来。需要注意的是这里的CREATOR名字不能改。
# 两者的区别
好了。到这里相信都了解怎么使用这两个接口了。主要是Parcelable比较复杂。现在有个问题了，为什么要造一个如此复杂的接口呢？来看看他们各自的优缺点。
##### Serializable
这个是java内部定义的序列化接口。
 - 优点：使用范围广，在网络，内存文件等均可以使用；使用方式简单，仅仅只需要继承一个接口就可以使用。
 - 缺点：性能差；序列化的时候需要创建很多的临时对象。

##### Parcelable
这个安卓中可以使用的接口，仅用于安卓。
- 优点：性能强大。开发者号称比Serialization强大10倍的性能。
- 缺点：实现过程复杂，需要把对象中的数据一个个拆开进行序列化；
			从实现原理可以看出来，这个只能用于内存中的序列化，不能用于网络和本地硬盘文件。

# 总结
序列化在安卓开发中还是很常用的，例如活动之间传递信息，跨进程传递数据等等都要用到。这两个接口一个简单粗暴，一个复杂强大，平常开发的话，尽量还是使用第二个接口，毕竟多一点代码可以加强这么多的性能何乐不为呢。Parcelable可能比较难理解，多看几遍就可以了。不理解或者笔者写错的可以评论区留言。
