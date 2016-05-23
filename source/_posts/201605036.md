title: Android开发笔记——图片缓存、手势及OOM分析
tags:
- Android
- 图片缓存
- OOM分析
categories: Android开发笔记

---
把图片缓存、手势及OOM三个主题放在一起，是因为在Android应用开发过程中，这三个问题经常是联系在一起的。首先，预览大图需要支持手势缩放，旋转，平移等操作；其次，图片在本地需要进行缓存，避免频繁访问网络；最后，图片（Bitmap）是Android中占用内存的大户，涉及高清大图等处理时，内存占用非常大，稍不谨慎，系统就会报OOM错误。

庆幸的是，这三个主题在Android开发中属于比较普遍的问题，有很多针对于此的通用的开源解决方案。因此，本文主要说明笔者在开发过程中用到的一些第三方开源库。主要内容如下：

 1. Universal Image Loader、Picasso、Glide与Fresco的对比及使用
 2. PhotoView、GestureImageView的原理及使用
 3. leakcanry内存分析工具


----------
## 一、Universal Image Loader、Picasso、Glide与Fresco的对比及使用
Universal Image Loader（UIL）、Picasso、Glide与Fresco是Android中进行图片加载的常用第三方库，主要封装了内存缓存、磁盘缓存、网络请求缓存、线程池等方法，抽象了图片加载的流程，很大程度避免了加载图片引起的内存溢出，提高了图片加载的效率。下图是笔者近期从各个库的github页面查询到的信息：
![](/img/20160108-1.png)
 需要说明的是：

 - Imageloader是最早开源的图片缓存库，目前作者已停止维护（11.27）；
 - Picasso的实际作者是Square的Jake Wharton，Android领域的绝对大牛；
 - Glide是由Google员工开源的，在Google I/O 2014官方应用中推荐使用；
 - Fresco的图片加载不使用Java堆内存，而是匿名共享内存（Ashmem）。
 
附上各个库的github地址：
> Universal Image Loader：https://github.com/nostra13/Android-Universal-Image-Loader.git
Picasso：https://github.com/square/picasso.git
Glide：https://github.com/bumptech/glide.git
Fresco：https://github.com/facebook/fresco.git

这四个图片缓存库的基本使用（HelloWorld）都可以通过一句代码实现，分别如下：
**UIL：**
```
ImageLoader.getInstance().displayImage(url, imageView);
```
**Picasso：**
```
Picasso.with(context).load(url).into(imageView);
```
**Glide：**
```
Glide.with(context).load(url).into(imageView);
```
**Fresco：**
```
simpleDraweeView.setImageURI(uri);
```
细心的朋友可以看出，Picasso和Glide的API非常类似。事实上，这四个库在实现的核心思想上都比较相似，可以抽象为以下五个模块：

 1. RequestManager，主要负责请求生成和管理模块；
 2. Engine，主要负责创建任务以及执行调度；
 3. GetDataInterface，获取数据的接口，主要用于从内存缓存、磁盘缓存以及网络等获取图片数据；
 4. Displayer，主要用于显示图片，可能是对ImageView的封装或者其他虚拟的Displayer；
 5. Processor，主要负责处理图片，比如图片的旋转、压缩以及截取等操作。
 

> 说一句题外话，掌握了各种开源库的实现的核心思想后，会发现软件工程的一个共同点，就是通过将流程形式化、抽象化，从而提高效率。不论是业务的效率，还是开发的效率，这或许也是软件作为一门科学的核心思想。

**ImageLoader的设计及优点**
ImageLoader加载的流程如下图。（需要申明：下面三张流程图来自Trinea，尊重原作者版权）
![](/img/20160108-2.png)
ImageLoader收到加载及显示图片的任务，ImageLoaderEngine分发任务，获得图片数据后，BitmapDisplayer 在ImageAware中显示。
**ImageLoader的优点：**

> - 支持下载进度监听
- 可以在 View 滚动中暂停图片加载，通过 PauseOnScrollListener 接口可以在 View 滚动中暂停图片加载。
- 默认实现多种内存缓存算法，这几个图片缓存都可以配置缓存算法，不过 ImageLoader 默认实现了较多缓存算法，如 Size 最大先删除、使用最少先删除、最近最少使用、先进先删除、时间最长先删除等。
- 支持本地缓存文件名规则定义

**Picasso的设计及优点**
Picasso的加载流程如下图：
![](/img/20160108-3.png)
Picasso收到加载及显示图片的任务，Dispatcher 负责分发和处理，通过MemoryCache及Handler获取图片，通过PicassoDrawable显示到Target中。
**Picasso的优点：**

> - 自带统计监控功能，支持图片缓存使用的监控，包括缓存命中率、已使用内存大小、节省的流量等。
- 支持优先级处理，每次任务调度前会选择优先级高的任务，比如 App 页面中 Banner 的优先级高于 Icon 时就很适用。
- 支持延迟到图片尺寸计算完成加载，支持飞行模式、并发线程数根据网络类型而变，手机切换到飞行模式或网络类型变换时会自动调整线程池最大并发数，比如 wifi 最大并发为 4， 4g 为 3，3g 为 2。这里 Picasso 根据网络类型来决定最大并发数，而不是 CPU 核数。
- “无”本地缓存，不是说没有本地缓存，而是 Picasso 自己没有实现，交给了 Square 的另外一个网络库 okhttp 去实现，这样的好处是可以通过请求 Response Header 中的 Cache-Control 及 Expired 控制图片的过期时间。

**Glide的设计及优点**
Glide的加载流程如下图：
![](/img/20160108-4.png)
Glide 收到加载及显示资源的任务，Engine 处理请求，通过Fetcher获取数据，经Transformation 处理后交给Target显示。
**Glide的优点：**

> (1) 图片缓存->媒体缓存
Glide 不仅是一个图片缓存，它支持 Gif、WebP、缩略图。甚至是 Video，所以更该当做一个媒体缓存。
(2) 支持优先级处理
(3) 与 Activity/Fragment 生命周期一致，支持 trimMemory
Glide 对每个 context 都保持一个 RequestManager，通过 FragmentTransaction 保持与 Activity/Fragment 生命周期一致，并且有对应的 trimMemory 接口实现可供调用。
(4) 支持 okhttp、Volley
Glide 默认通过 UrlConnection 获取数据，可以配合 okhttp 或是 Volley 使用。实际 ImageLoader、Picasso 也都支持 okhttp、Volley。
(5) 内存友好
① Glide 的内存缓存有个 active 的设计 
从内存缓存中取数据时，不像一般的实现用 get，而是用 remove，再将这个缓存数据放到一个 value 为软引用的 activeResources map 中，并计数引用数，在图片加载完成后进行判断，如果引用计数为空则回收掉。
② 内存缓存更小图片 
Glide 以 url、viewwidth、viewheight、屏幕的分辨率等做为联合 key，将处理后的图片缓存在内存缓存中，而不是原始图片以节省大小
③ 与 Activity/Fragment 生命周期一致，支持 trimMemory
④ 图片默认使用默认 RGB565 而不是 ARGB888 
虽然清晰度差些，但图片更小，也可配置到 ARGB_888。
其他：Glide 可以通过 signature 或不使用本地缓存支持 url 过期

**关于Fresco**
Fresco库开源较晚，目前还没有正式的1.0版本。但其功能比前三个库都强大，比如：

 - 图片存储系统匿名共享内存Ashmem（Anonymous Shared Memory），并不分配Java堆内存，因此图片加载不会引起堆内存抖动；
 - JPEG图像流加载（先显示图像轮廓，再慢慢加载清晰图像）；
 - 更加完善的图像处理、显示方式；
 - JPEG图像本地（native）变换尺寸，避免OOM；
 - ......
关于系统匿名共享内存Ashmem，会在后续的一篇关于Android的内存使用的文章中详述，这里仅作简单介绍：

在Android系统里面，Ashmem这个区域的内存并不属于Java Heap，也不属于Native Heap。当Ashmem中的某个内存空间像要被释放时候，会通过系统调用unpin来告知。但实际上这块内存空间的数据并没有被真正的擦除。如果Android系统发现内存吃紧时，就会把unpin的内存空间利用起来去存储所需的数据。而被unpin的内存空间，是可以被重新pin的，如果此时的该内存空间还没有被其他人使用的话，就节省了重新往Ashmem重新写入数据的过程了。所以，Ashmem这个工作原理是一种延迟释放。

另外，学习Ashmem可以参考罗升阳大师的博客：

Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划
Android系统匿名共享内存Ashmem（Anonymous Shared Memory）驱动程序源代码分析
Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析

## 二、PhotoView、GestureImageView的原理及使用
需要使用上述第三方开源库进图片加载的一个典型场景是点击查看大图。大图支持手势缩放、旋转、平移等操作，ImageView的手势缩放，有很多种方法，绝大多数开源自定义缩放都是修改了ondraw函数来实现的。但是ImageView本身有scaleType属性，通过设置android:scaleType="matrix" 可以轻松实现缩放功能。缩放的优点是实现起来简单，同时因为没有反复调用ondraw函数，缩放过程中不会有闪烁现象。另外，需要注意的是，scaleType控制图片的缩放方式，该图片指的是资源而不是背景，换句话说，android:src="@drawable/ic_launcher"，而非android:background="@drawable/ic_launcher"。

在github上可以找到很多开源的实现，这里主要举两个例子进行简单说明。
 

> PhotoView地址：https://github.com/bm-x/PhotoView.git
GestureImageView地址：https://github.com/jasonpolites/gesture-imageview.git

PhotoView的介绍：

1.Gradle添加依赖（推荐）
```
dependencies {
    compile 'com.bm.photoview:library:1.3.6'
}
```
（或者也可以将项目下载下来，将Info.java和PhotoView.java两个文件拷贝到你的项目中，不推荐）——这种方式适用于Eclipse。

2.xml添加
```
<com.bm.library.PhotoView
     android:id="@+id/img"
     android:layout_width="match_parent"
     android:layout_height="match_parent"
     android:scaleType="centerInside"
     android:src="@drawable/bitmap1" />
```
3.java代码
```
PhotoView photoView = (PhotoView) findViewById(R.id.img);
// 启用图片缩放功能
photoView.enable();
// 禁用图片缩放功能 (默认为禁用，会跟普通的ImageView一样，缩放功能需手动调用enable()启用)
photoView.disenable();
// 获取图片信息
Info info = photoView.getInfo();
// 从一张图片信息变化到现在的图片，用于图片点击后放大浏览，具体使用可以参照demo的使用
photoView.animaFrom(info);
// 从现在的图片变化到所给定的图片信息，用于图片放大后点击缩小到原来的位置，具体使用可以参照demo的使用
photoView.animaTo(info,new Runnable() {
       @Override
       public void run() {
           //动画完成监听
       }
   });
// 获取动画持续时间
int d = PhotoView.getDefaultAnimaDuring();
```
PhotoView实现的基本原理是在继承于ImageView的PhotoView中采用了缩放Matrix及手势监听。
```
public class PhotoView extends ImageView {

    ……

    private Matrix mBaseMatrix = new Matrix();
    private Matrix mAnimaMatrix = new Matrix();
    private Matrix mSynthesisMatrix = new Matrix();
    private Matrix mTmpMatrix = new Matrix();

    private RotateGestureDetector mRotateDetector;
    private GestureDetector mDetector;
    private ScaleGestureDetector mScaleDetector;
    private OnClickListener mClickListener;

    private ScaleType mScaleType;
    
    ……
}
```
PhotoView的实现与上述原理基本一致，这里不再赘述。对于自定义控件的实现，后续文章会进行详细的分析。

GestureImageView的简介如下：
1.Configured as View in layout.xml
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:gesture-image="http://schemas.polites.com/android"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent">

    <com.polites.android.GestureImageView
        android:id="@+id/image"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:src="@drawable/image"
        gesture-image:min-scale="0.1"
        gesture-image:max-scale="10.0"
        gesture-image:strict="false"/>

</LinearLayout>
```
2.Configured Programmatically
```
public class SampleActivity extends Activity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        LayoutParams params = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);

        GestureImageView view = new GestureImageView(this);
        view.setImageResource(R.drawable.image);
        view.setLayoutParams(params);

        ViewGroup layout = (ViewGroup) findViewById(R.id.layout);

        layout.addView(view);
    }
}
```
原理基本同PhotoView一致，不再赘述。

## 三、OOM分析工具——LeakCanary
 LeakCanary的介绍：
 

> A memory leak detection library for Android and Java.

可见，LeakCanary主要用于检测各种内存不能被GC，从而导致泄露的情况。

> LeakCanary的地址   https://github.com/square/leakcanary.git
Demo地址：
https://github.com/liaohuqiu/leakcanary-demo.git（AS）
https://github.com/teffy/LeakcanarySample-Eclipse.git（Eclipse）

下面是demo中TestActivity中的TextView被静态变量引用导致无法回收引起的内存泄露的截图。
![](/img/20160108-5.png)
LeakCanary的使用较为简单，首先添加依赖工程：
```
dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
}
```
其次，在application的onCreate()方法中进行初始化。
```
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```
经过这两步之后就可以使用了。LeakCanary.install(this)会返回一个预定义的 RefWatcher，同时也会启用一个ActivityRefWatcher，用于自动监控调用 Activity.onDestroy() 之后泄露的 activity。如果需要监听fragment，则在fragment的onDestroy()方法进行注册：
```
public abstract class BaseFragment extends Fragment {

  @Override public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
  }
}
```
当然，需要对某个变量进行监听，直接对其进行watch即可。
```
RefWatcher refWatcher = {...};

// We expect schrodingerCat to be gone soon (or not), let's watch it.
refWatcher.watch(schrodingerCat);
```
需要注意的是，在eclipse中使用LeakCanary需要在AndroidManifest文件中对堆占用分析以及展示的Service进行申明：
```
<service
    android:name="com.squareup.leakcanary.internal.HeapAnalyzerService"
    android:enabled="false"
    android:process=":leakcanary" />
<service
    android:name="com.squareup.leakcanary.DisplayLeakService"
    android:enabled="false" />

<activity
    android:name="com.squareup.leakcanary.internal.DisplayLeakActivity"
    android:enabled="false"
    android:icon="@drawable/leak_canary_icon"
    android:label="@string/leak_canary_display_activity_label"
    android:taskAffinity="com.squareup.leakcanary"
    android:theme="@style/leak_canary_LeakCanary.Base" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```
注意：HeapAnalyzerService采用了多进程android:process=":leakcanary"。

> 上述开源工具的使用都较为简单，关于详细使用，请参考其github地址。

## 四、一些杂乱的总结
 **内存泄露的常见原因：**

 - 静态对象：监听器，广播，webview；
 - this$0：线程，定时器，Handler；
 - 系统：TextLine，输入法，音频

**兜底回收内存：**

Activity泄露会导致该Activity引用的Bitmap/DrawingCache等无法释放，兜底回收是指对已泄露的Activity，尝试回收其持有的资源。在onDestroy中从rootview开始，递归释放所有子VIew涉及的图片，背景，DrawingCache，监听器等资源。

**降低Runtime内存的方法：**

 1. 减少bitmap占用的内存：1）防止bitmap占用资源过大，2.x系统打开BitmapFactory.Options中的inNativeAlloc；4.x系统采用Facebook的fresco库，将图片资源放于native中。2）图片按需加载，图片的大小不应超过view的大小。3）统一的bitmap加载器：Picasso/Fresco。4）图片存在像素浪费。
 2. 自身内存占用监控：1）实现原理：通过Runtime获取maxMemory，而totalMemory-freeMemory即为当前真正使用的dalvik内存。2）操作方式：定期检查这个值，达到80%就去释放各种cache资源（bitmap的cache）。
 3. 使用多进程。对于webview，图库等，由于存在内存系统泄露，可以采用单独的进程。