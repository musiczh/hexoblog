---
title: 启动页白屏解决方案 #标题
date: 2020/4/21 00:00:00 #建立日期
updated: 2020/4/21 17:00:00 #更新日期
comments: true #开启评论
tags:  #标签
 - android 

categories:  #分类
 - Android

---

﻿当我们打开app的时候是不是会有一瞬间的白屏然后再进入主活动，虽然这并不会造成什么不好的后果，但是感觉用户体验就不是很好。像网易云音乐等等，打开一瞬间就显示了他们的loge，无缝衔接，没有白屏，怎么做到的呢？

一开始我的思路是这样的。可能是因为我们的主活动逻辑太多，所以加载会变慢，导致显示白屏。如果使用一个只显示一张本地图片的活动，那会不会就不会显示白屏了呢。话不多说我们尝试一下：

Activity中的代码：
```java
/**
 * 启动页，显示倾旅的logo，停顿2秒后跳转
 */
public class LunchActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_lunch);

		//开启子线程进行停顿。如果在主线程停顿的话，会造成主页面卡死，所以在子线程sleep两秒后跳转
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                start();
                LunchActivity.this.finish();
            }
        }).start();
    }
    //跳转到主页面
    private void start(){
        Intent intent = new Intent(LunchActivity.this,MainActivity.class);
        startActivity(intent);
    }
}
```
layout中的代码：
```java
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#e74b37"
    tools:context=".LunchActivity">

    <ImageView
        android:id="@+id/imageView5"
        android:layout_width="80dp"
        android:layout_height="80dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.31"
        app:srcCompat="@drawable/icon" />
</android.support.constraint.ConstraintLayout>
```
这里简单指定一个imageView来显示一张图片。并把背景设置为橘色

最后再把启动页活动设置为主活动：
```java
<activity android:name="com.example.qinglv.LunchActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

一切想的很好，完成后打开一看，还是会白屏，怎么回事？

活动的加载都是需要时间的，比较简单的活动时间会少点，但是以然会有一瞬间的白屏。那这个白屏到底是什么？就是每个活动的背景。当打开一个活动的时候，因为还没加载出内容，所以显示的就只是背景，所以我们只需要，改变这个背景，设置为我们需要的一个logo照片即可。怎么设置呢？

 - 背景是在主题中指定的，首先设置一个主题，把背景改成我们要的。一般和我们的启动页保持一致，这样的话就不会看起来像两个启动页一样。也可以像网易云音乐那样，背景设置成logo，但是启动页是放广告，但是这会影响用户体验（为了收入打点广告也是可以理解的）。看代码：
在res-value-styles：
 ```java
<style name="NewAppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="android:windowBackground">@color/colorPrimary</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
 ```
重点是这句`<item name="android:windowBackground">@color/colorPrimary</item>`
这里我指定的是一种颜色你们也可以指定一张图片

 - 再给启动页活动指定主题：
 在：AndroidManifest：
 ```java
 <activity android:name="com.example.qinglv.LunchActivity"
            android:theme="@style/NewAppTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
 ```
重点是这句`android:theme="@style/NewAppTheme"`


然后再打开的时候，就会发现不会了。原本显示的白屏变成了我们设置好的图片。
