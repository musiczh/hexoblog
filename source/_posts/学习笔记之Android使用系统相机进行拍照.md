@[toc]
# 前言
我们在日常的开发中有时候会遇到需要用到相机的需求，而相机也是很常用的东西，例如扫二维码啊拍照上传啊等等。这里我不讲像qq那样自定义很强的拍照功能（事实上我也不会），讲个最简单的调用系统相机拍照并储存
# 调用系统相机步骤
这里我通过一个简单的例子来讲这个内容。
我自己写了一个demo，布局很简单：
```java
<Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:text="take phone"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.281"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <ImageView
        android:id="@+id/imageView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="29dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/button"
        app:srcCompat="@mipmap/ic_launcher_round" />
```
就是一个按钮点击弹起相机，然后一个imageView显示拍到的照片。

·

接下来我想一下调用的整个过程我们需要做什么：
首先弹起相机肯定要跳到相机这个应用，那么就必须通过隐性启动相机的活动。
然后当我们返回应用的时候，还要将照片显示，所以这里就要用到startActivityForResult这个方法。
其次，我们拍照之后肯定要进行储存的，那么就涉及到文件的操作。
涉及到内存的操作就肯定要和权限打交道，所有还有权限相关的内容。
最后还有一个问题就是，相机拍完照是要储存照片的，所以我们要给他一个地址uri，但是可不可以直接把地址当成参数发过去呢？这里就要用到特殊的内容提供器FileProvider。
上面就是调用相机要用到的内容，虽然用的都很浅，但是都会涉及到。接下来看看具体怎么实现。看看Activity中的onCreate的代码：
·

```java
    private Uri imageUri;
	private ImageView imageView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        imageView = findViewById(R.id.imageView);

        Button button = findViewById(R.id.button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
            
            	//创建一个File对象。getExternalCacheDir()获取应用的缓存目录，outputImage.jpg是照片名称
                File outputImage = new File(getExternalCacheDir(),"outputImage.jpg");
                try{                
                //创建一个空文件            
                    outputImage.createNewFile();
                }catch (IOException e){
                    e.printStackTrace();
                }
                
                //不同的安卓版本对用不同的获取Uri的方法
                if (Build.VERSION.SDK_INT>=24){
                    imageUri =FileProvider.getUriForFile(MainActivity.this,"huan",outputImage);
                }else{
                    imageUri = Uri.fromFile(outputImage);
                }
                
				//启动相机的对应Activity
                Intent intent = new Intent("android.media.action.IMAGE_CAPTURE");
                intent.putExtra(MediaStore.EXTRA_OUTPUT,imageUri);
                startActivityForResult(intent,1);
            }
        });
```
我们来看看这里的代码：前面的代码很简单就是控件的初始化。
我们知道照片是要放文件夹得，所以这里要创建一个File对象，指定文件的路径以及名字。这里路径为什么要用getExternalCacheDir()呢？因为每个应用都有对应得缓存目录，访问这些目录得时候不用访问内存权限，这样得话就可以省去需求权限得步骤啦。这个目录在/scare/Android/data/<package name>/cache。

然后我们再创建一个空的文件夹。这里如果已经有照片了的话例如我们第二次拍照的时候，那么就不会创建新的空文件夹了。直到储存的时候才会被替换掉。

然后我们刚才讲到，拍到的图片要在我们的应用中展示，那么就必须用到内容提供器。这里用到FileProvider来获取uri，关于provider我在下文有讲到可以[点击跳转](#jump)
如果是低于4.4版本的安卓就用Uri.fromFile(outputImage);方法可以获取到uri

再通过隐式启动相机activity可以打开相机了。这里系统相机的action是android.media.action.IMAGE_CAPTURE，相机储存路径的参数名字是MediaStore.EXTRA_OUTPUT，并把uri传输进去。

好了这样就完成了拍照并把照片储存的步骤了。接下来还差什么？对了，把照片显示出来。现在在内存中已经有这个照片了，而且uri也知道，所以就很容易了，看代码：
```java
    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        if (requestCode == 1) {
            if (resultCode == RESULT_OK)
                try {
                    Bitmap bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(imageUri));
                    imageView.setImageBitmap(bitmap);
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                }
        }
    }
```
我们刚才使用startActivityForResult来启动活动的，所以就要重写这个方法来显示图片了。这里首先判断是哪个启动命令，然后再判断是否成功启动，再BitmapFactory.decodeStream这个方法来获取bitmap，再把bitmap显示出来就行了。BitmapFactory.decodeStream这个方法需要一个流，可以通过getContentResolver().openInputStream这个方法来开启一个流。

到此整个流程就解决了。

# <span id = "jump">FileProvider</span>
FileProvider是一个特殊的内容提供器，可以把一个file开头的uri改成content开头的，例如：file://uri -> content://uri。那为什么要这么做呢？这里简单讲一下：
这个是因为在Android 7.0之后，官方禁止直接把一个真实路径的uri传输到别的应用，而我们要把地址送给相机，所以就会出现问题了。详细可以查阅这篇博客：[Android 7.0 行为变更 通过FileProvider在应用间共享文件吧](https://blog.csdn.net/lmj623565791/article/details/72859156)

既然是内容提供器那么肯定是要进行注册的：
```java
<provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="huan"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
```
这里的authorities参数必须和前面的getUriForFile方法的第二个参数保持一致，同个内容提供器的authorities肯定要一样啦。grantUriPermissions参数一定要是true，这个的大概意思就是给他的所有元素授权可以被访问，在FileProvider中这个参数必须是true（这也是为什么在4.4一下版本的安卓无法使用的原因之一，有兴趣可以去了解一下）export这个参数表示可不可以给其他的应用共享，这里要设置为false。<meta-data这个是配置我们可以访问的文件路径，@xml/file_paths这个就是表示什么文件可以被访问，当然要建一个这个文件。看代码：
```java
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path
        name="my_image"
        path="/"/>
</paths>
```
<external-path这个就是表示可以被访问的路径，name是后面映射用到的，可以自己随便起，我这里用一横杆表示整个目录可以被访问。

# 小结
调用系统相机的功能虽然不难，代码也不多，但是其中的零碎知识很多，零零散散，还是要注意的。特别是关于低高配的安卓版本问题还是要特别注意一下。

·
·
·
·
### 参考资料
《第一行代码》郭霖
[Android 7.0 行为变更 通过FileProvider在应用间共享文件吧](https://blog.csdn.net/lmj623565791/article/details/72859156)


