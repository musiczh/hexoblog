## with

with方法重载非常多，主要是为了接受不同的参数类型，最终都是调用了getRetriever这个方法。最终是返回了RequestManager对象。

> 没有注意到生命周期相关内容

**with方法的作用就是创建requestManager对象**

```java
public static RequestManager with(@NonNull Context context) {
  return getRetriever(context).get(context);
}

public static RequestManager with(@NonNull Activity activity) {
  return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull Fragment fragment) {
  return getRetriever(fragment.getContext()).get(fragment);
}

public static RequestManager with(@NonNull android.app.Fragment fragment) {
  return getRetriever(fragment.getActivity()).get(fragment);
}

public static RequestManager with(@NonNull View view) {
  return getRetriever(view.getContext()).get(view);
}
```



### getRetriever

**获取一个RequestManagerRetriever对象**

```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // context可为null的原因只有在错误的fragment生命周期
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
        + "returns null (which usually occurs when getActivity() is called before the Fragment "
        + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}
```

### Glide.get(context)

**创建一个Glide对象**

这一步会对Glide进行初始化，过程中会创建一个RequestManagerRetriever。

```java
public static Glide get(@NonNull Context context) {
    // 很明显的单例模式，采用双重检锁来实现
    // glide被设置为volatile
    if (glide == null) {
        GeneratedAppGlideModule annotationGeneratedModule =
            getAnnotationGeneratedGlideModules(context.getApplicationContext());
        synchronized (Glide.class) {
            if (glide == null) {
                checkAndInitializeGlide(context, annotationGeneratedModule);
            }
        }
    }
    return glide;
}
```

```java
Glide.get() 
-> Glide.checkAndInitializeGlide()
-> Glide.initializeGlide()
-> Glide.initializeGlide()
-> GlideBuilder.build()
->RequestManagerRetriever()

// 内部创建一个主线程handler
public RequestManagerRetriever(@Nullable RequestManagerFactory factory) {
    this.factory = factory != null ? factory : DEFAULT_FACTORY;
    handler = new Handler(Looper.getMainLooper(), this /* Callback */);
}
```



## load

load方法的作用是对图片的来源进行记录，最终返回RequestBuilder对象

```java
public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) {
  return asDrawable().load(bitmap);
}
public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {
  return asDrawable().load(drawable);
}
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
public RequestBuilder<Drawable> load(@Nullable Uri uri) {
  return asDrawable().load(uri);
}
public RequestBuilder<Drawable> load(@Nullable File file) {
  return asDrawable().load(file);
}
public RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
  return asDrawable().load(resourceId);
}
public RequestBuilder<Drawable> load(@Nullable URL url) {
  return asDrawable().load(url);
}
public RequestBuilder<Drawable> load(@Nullable byte[] model) {
  return asDrawable().load(model);
}
public RequestBuilder<Drawable> load(@Nullable Object model) {
  return asDrawable().load(model);
}
```

### asDrawable()

创建一个RequestBuilder<Drawable>

```java
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}
```

### RequestBuilder.load

```java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
// 所有的load方法最终都被定向到这个方法，可见load只是做个记录，并没有发起请求
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
}
```



## into

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);

  BaseRequestOptions<?> requestOptions = this;
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }

  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}
```