title: Android开发笔记——ListView模块、缓存及性能
tags:
- ListView
- notifyDataSetChanged
- 图片缓存
categories: 日志

---

ListView是Android开发中最常用的组件之一。本文将重点说明如何正确使用ListView，以及使用过程中可能遇到的问题。

 1. ListView开发模块
 2. 图片缓存
 3. 可能遇到的问题

----------
## 一、ListView开发模块 ##

从项目实践的角度来看，ListView适合“自底向上”的开发模式，即从每个条目的显示组件，到对其进行控制的数据结构，最后通过Activity等进行使用。主要包括以下模块：

 1. 首先是item组件，即用于每项布局输出的xml文件。Android SDK中有simple_list_item_1、simple_list_item_2可用，当需要比较丰富的显示效果时，一般通过自定义xml实现。本文以论坛的格式进行说明，主要包括发帖人头像、用户名，帖子的标题、内容、最后回复时间、编辑、收藏、回复等内容，布局文件比较简单，这里截取其中一项显示，用以说明：
![](/img/20151023-1.jpg)
 2. 其次是父对象layout文件，即用于Activity或者Fragment的布局输出文件，一般在此输出文件中包含ListView。当然，如果采用ListFragment或ListActivity，并不需要再显示的定义ListView组件。本文中采用Fragment默认的输出文件，当然，也可以采用自定义的布局文件。
```
<ListView
    android:id="@+id/topic_list"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:cacheColorHint="@android:color/transparent"
    android:divider="@color/topic_divider_color"
    android:dividerHeight="1px"
    android:listSelector="@android:color/transparent" >
</ListView>
```
 3. 定义数据结构（容器），即用于持有单个Item的数据，可以是简单的String，也可以通过抽象Items所需字段组成一个类，抽象的原则是与Item中的组件对应。本文中上图涉及多个字段，因此通过抽象组件形成BBSTopicItem类。
 4. 列表适配器。决定每行Item中具体显示什么内容，以怎样的样式显示等，通常通过继承ArrayAdapter、SimpleAdapter等实现。本文定义BBSTopicAdapter，继承于ArrayAdapter<BBSTopicItem>。
 5. 最后，需要定义一个Activity或Fragment来使用上述模块。需要说明的是，ListView可以直接被ListActivity或者ListFragment使用。

以上五个模块就是使用ListView的**基本逻辑框架**，开发过程中，需要时刻理清它们之间的关系。 

----------

## 二、图片缓存及相关 ##
在ListView中显示图片是比较常见的应用场景，但加载图片一般需要通过缓存来进行处理。由于虚拟机的heapsize默认为16M：
```
AndroidRuntime.cpp

int AndroidRuntime::startVM(JavaVM** pJavaVM, JNIEnv** pEnv) {
    ……
    property_get("dalvik.vm.heapsize", heapsizeOptsBuf+4, "16M");
    ……
}
```
（厂商一般会修改为32M，后面会说到这个数值）
在操作大尺寸图片时无法分配所需内存，就会引起OOM。因此，使用LruCache来缓存图片是常见的做法。但在其使用过程中，也需要注意一些问题，比如使用线程池下载图片，使用SD卡缓存，ListView滑动流畅性，图片显示错乱等，下面对这些问题进行说明。

**1. 首先是LruCache，该类在android-support-v4的包中提供**
主要算法原理是把最近使用的对象用强引用存储在LinkedHashMap中，并且把最近最少使用的对象在缓存值达到预设定值之前从内存中移除。

以前经常会使用一种非常流行的内存缓存技术的实现，即软引用或弱引用 (SoftReference or WeakReference)。但是现在已经不再推荐使用这种方式了，因为从 Android 2.3 (API Level 9)开始，垃圾回收器会更倾向于回收持有软引用或弱引用的对象，这让软引用和弱引用变得不再可靠。另外，Android 3.0 (API Level 11)中，图片的数据会存储在本地的内存当中，因而无法用一种可预见的方式将其释放，这就有潜在的风险造成应用程序的内存溢出并崩溃。

那么，怎样确定一个合适的缓存大小给LruCache呢？有以下多个因素应该放入考虑范围内，例如：

 - 设备可以为每个应用程序分配多大的内存？
 - 设备屏幕上一次最多能显示多少张图片？有多少图片需要进行预加载？
 - 设备的屏幕大小和分辨率分别是多少？一个超高分辨率的设备（例如 Galaxy Nexus)比起一个较低分辨率的设备（例如 Nexus S），在持有相同数量图片的时候，需要更大的缓存空间。
 - 图片的尺寸和大小，还有每张图片会占据多少内存空间？
 - 图片被访问的频率有多高？是否有一些图片的访问频率比其它图片要高？如果有的话，应该让一些图片常驻在内存当中，或者使用多个LruCache 对象来区分不同组的图片。
 - 是否能维持好数量和质量之间的平衡吗？有些时候，存储多个低像素的图片，而在后台去开线程加载高像素的图片会更加的有效。

**并没有一个指定的缓存大小可以满足所有的应用程序**，通常需要分析程序内存的使用情况，然后制定出一个合适的解决方案。缓存太小，有可能造成图片频繁地被释放和重新加载；而缓存太大，则有可能还是会引起 java.lang.OutOfMemory 异常。
不过，读者可能会在不同的地方遇到类似下面这段定义LruCache的代码：
```
public ImageDownLoader(Context context){
    int maxMemory = (int) Runtime.getRuntime().maxMemory();  
    int mCacheSize = maxMemory / 8;
    mMemoryCache = new LruCache<String, Bitmap>(mCacheSize){

        @Override
        protected int sizeOf(String key, Bitmap value) {
            return value.getRowBytes() * value.getHeight();
        }
    };
}
```

**2. 使用线程池管理图片下载任务**
线程池自然是为了限制系统中执行线程的数量，通常的一种比较低效的做法是为每一张图片下载开启一个新线程（new thread），线程的创建和销毁将造成极大的性能损耗，而对服务器来讲，维护过多的线程将造成内存消耗过大。总的来讲，使用线程池是执行此类任务的一个基本做法。Java中线程池的顶级接口是Executor，但是严格意义上讲Executor并不是一个线程池，而只是一个执行线程的工具，真正的线程池接口是ExecutorService。配置线程池是略显复杂，尤其是对于线程池的原理不是很清楚的情况下，很有可能配置的线程池不是最优的。在Executors类里提供了一些静态工厂，生成一些常用的线程池。

 - newSingleThreadExecutor创建一个单线程的线程池。这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。
 - newFixedThreadPool 创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。
 - newCachedThreadPool 创建一个可缓存的线程池。如果线程池的大小超过了处理任务所需要的线程， 
那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。
 - newScheduledThreadPool创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

通过下面的代码创建固定大小的线程池：
```
private ExecutorService getThreadPools(){
    if(mImageThreadPool == null){
        synchronized(ExecutorService.class){
            if(mImageThreadPool == null){
                mImageThreadPool = Executors.newFixedThreadPool(4);
            }
        }
    }
    return mImageThreadPool;
}

(本段代码来自互联网)
```
对于固定大小的线程池，关键是需要根据实际应用场景设置线程数量，既快速有效的执行下载任务，又不造成资源浪费。

**3. SD卡存储配合LruCache**
使用SD卡存储下载的图片有多方面的好处，提高图片加载速度（从本地加载肯定比网络要快）、节约用户流量等，因此，除了LruCache，一般还会将图片存储在本地SD卡。因此，加载图片的顺序应该是

　　a. 首先从LruCache中获取图片； 
　　b. 如果a的返回值为null，则检查SD卡是否存在图片； 
　　c. 如果a、b的返回值都为null，则通过网络进行下载。

　　用代码表述上述逻辑为：
```
Bitmap bitmap;
if (getBitmapFromMemCache(url) != null) {
    bitmap = getBitmapFromMemCache(url);
} else if (fileUtils.isFileExists(url) && fileUtils.getFileSize(url) != 0) {
    bitmap = fileUtils.getBitmapFromSD(url);
} else {
    bitmap = getBitmapFormUrl(url);
}
return bitmap;
```
Tip：如果通过HttpURLConnection下载图片，需要注意一个小问题，如果设置HttpURLConnection对象的DoOutput属性为true（con.setDoOutput(true)），在Android4.0以后，会解析为post请求，导致filenotfound异常（获取图片应该是get请求）。

**4. ListView滑动停止下载**
滑动时停止下载也是提高用户体验的方式之一，因为如果在ListView滑动过程中执行下载任务，将会使得ListView出现卡顿。监听滑动状态改变的方法是**onScrollStateChanged(AbsListView view, int scrollState)**，该方法在**OnScrollListener**接口中定义的，而OnScrollListener是**AbsListView**中为了在列表或网格滚动时执行回调函数而定义的接口。（强烈建议做ListView相关应用的读者熟悉一下AbsListView的源码）

为了实现下载任务与滑动状态的关联，在自定义列表适配器中实现了OnScrollListener接口，在onScrollStateChanged方法中根据scrollState执行相应的下载任务操作。
```
@Override
public void onScrollStateChanged(AbsListView view, int scrollState) {
    this.scrollState = scrollState;
    if (scrollState == AbsListView.OnScrollListener.SCROLL_STATE_IDLE) {
        showImage(mFirstVisibleItem, mVisibleItemCount);
    } else {
        cancelTask();
    }
}

（本段代码来自互联网）
```

**5. 图片显示错乱**
这是一个比较老生常谈的问题，在百度搜索一下“listview图片错位”会见到一大片帖子在讨论这个问题，这里不再赘述，推荐几个比较靠谱链接：

http://www.cnblogs.com/lesliefang/p/3619223.html 
http://www.trinea.cn/android/android-listview-display-error-image-when-scroll/ 
http://blog.csdn.net/shineflowers/article/details/41744477

----------
总的来讲，图片缓存是Android开发中比较有意思的一个话题，常用的图片缓存开源库有ImageLoader、Picasso、Glide等，最近由Facebook开源了Fresco（http://www.fresco-cn.org/） ，根据介绍，它能够从网络、本地存储和本地资源中加载图片。同时，为了节省数据和CPU，它拥有三级缓存。此外，Fresco在显示方面是用了Drawees，可以显示占位符，直到图片加载完成。而当图片从屏幕上消失时，会自动释放图片所占的内存。这里推荐一个关于Android三大图片缓存原理、特性对比的链接：

http://www.csdn.net/article/2015-10-21/2825984#rd


## 三、可能遇到的问题

 **1. notifyDataSetChanged与局部更新**
 首先举一个栗子：QQ空间或者朋友圈的点赞功能，点赞之后页面会马上刷新，但不会影响本条目以外的其他条目的显示。换句话说，它使用了局部更新，而非notifyDataSetChanged。在了解notifyDataSetChanged与局部更新区别时，需要先对以下问题作出解释：

 - notifyDataSetChanged如何刷新界面？
 - 什么场景需要使用notifyDataSetChanged？什么场景需要使用局部更新？
 - 局部更新如何实现？

回到代码中，notifyDataSetChanged是在BaseAdapter中定义的，首先初始了一个DataSetObservable类的final实例mDataSetObservable：
```
private final DataSetObservable mDataSetObservable = new DataSetObservable();
```
notifyDataSetChanged就是通过操作mDataSetObservable实现的，DataSetObservable是观察者模式的一个实现（Android源码中有很多类似设计模式的实现）。
```
public void registerDataSetObserver(DataSetObserver observer) {
    mDataSetObservable.registerObserver(observer);
}

public void unregisterDataSetObserver(DataSetObserver observer) {
    mDataSetObservable.unregisterObserver(observer);
}

public void notifyDataSetChanged() {
    mDataSetObservable.notifyChanged();
}

public void notifyDataSetInvalidated() {
    mDataSetObservable.notifyInvalidated();
}
```
notifyDataSetChanged调用了notifyChanged方法，回到DataSetObservable中：
```
public void notifyChanged() {
    synchronized(mObservers) {
        // since onChanged() is implemented by the app, it could do anything, including
        // removing itself from {@link mObservers} - and that could cause problems if
        // an iterator is used on the ArrayList {@link mObservers}.
        // to avoid such problems, just march thru the list in the reverse order.
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged();
        }
    }
}

public void notifyInvalidated() {
    synchronized (mObservers) {
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onInvalidated();
        }
    }
}
```
notifyChanged也只是调用了其绑定的接口，并没有具体的实现，那么这个接口是什么时候绑定的呢？回忆ListView与adapter的关联是何时开始的呢？setAdapter！是的，从setAdapter的代码中可以看到这种关联。
```
@Override
public void setAdapter(ListAdapter adapter) {
    if (mAdapter != null && mDataSetObserver != null) {
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
    }

    resetList();
    mRecycler.clear();

    if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
        mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
    } else {
        mAdapter = adapter;
    }

    mOldSelectedPosition = INVALID_POSITION;
    mOldSelectedRowId = INVALID_ROW_ID;

    // AbsListView#setAdapter will update choice mode states.
    super.setAdapter(adapter);

    if (mAdapter != null) {
        mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
        mOldItemCount = mItemCount;
        mItemCount = mAdapter.getCount();
        checkFocus();
        // 原来是在这里绑定了数据改变的观察者对象
        mDataSetObserver = new AdapterDataSetObserver();
        mAdapter.registerDataSetObserver(mDataSetObserver);

        mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

        int position;
        if (mStackFromBottom) {
            position = lookForSelectablePosition(mItemCount - 1, false);
        } else {
            position = lookForSelectablePosition(0, true);
        }
        setSelectedPositionInt(position);
        setNextSelectedPositionInt(position);

        if (mItemCount == 0) {
            // Nothing selected
            checkSelectionChanged();
        }
    } else {
        mAreAllItemsSelectable = true;
        checkFocus();
        // Nothing selected
        checkSelectionChanged();
    }

    requestLayout();
}
```
前面提到，观察者对象调用的onChanged方法，可以确定，上述绑定的AdapterDataSetObserver中必然有onChanged方法的实现。
```
public void onChanged() { 
    mDataChanged = true;
    mOldItemCount = mItemCount;
    mItemCount = getAdapter().getCount();

    if ((getAdapter().hasStableIds()) && 
        (mInstanceState != null) && 
        (mOldItemCount == 0) && 
        (mItemCount > 0)) {
        onRestoreInstanceState(mInstanceState);
        mInstanceState = null;
    } else {
        rememberSyncState();
    }
    checkFocus();
    requestLayout();
}
```
很明显，在onChanged的末尾调用了requestLayout方法，而requestLayout方法是用来绘制界面的，定义在View中。
```
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        // Only trigger request-during-layout logic if this is the view requesting it,
        // not the views in its parent hierarchy
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }

    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```
根据上述解释，会发现notifyDataSetChanged会通知View刷新所有与其绑定的数据列表，而某些局部操作明显不需要全部刷新，全局刷新会造成极大的资源浪费。在这种情况下，就需要进行局部更新。

局部更新的实现定义在是适配器（adapter）中，根据指定的index（即item在listview中的位置），实现指定条目内容的更新：
```
/** 
 * 局部刷新
 * @param index item在listview中的位置 
 */  
public void updateItem(int index) {  
    if (listView == null) {  
        return;  
    }
    // 停止滑动时才更新界面
    if (scrollState == AbsListView.OnScrollListener.SCROLL_STATE_IDLE) {
        // 指定更新的位置在可见范围之内
        if (index >= listView.getFirstVisiblePosition() && 
            index <= listView.getLastVisiblePosition()) {
            // 获取当前可以看到的item位置  
            int visiblePosition = listView.getFirstVisiblePosition();
            View view = listView.getChildAt(index - visiblePosition); 
            //在这里对view中的组件进行设置，数据可以通过getItem(index)获取//
        }
    }
}
```
（一个小问题：ListView的getCount()与getChildCount()有什么差别呢？）

使用notifyDataSetChanged时，一个常见的问题就是调用了notifyDataSetChanged，但界面并没有刷新。很常见的原因是list的指向改变了，换句话说，list指向了与初始化时不同的堆地址。这种情况比较常见，给一个说明的链接：http://www.tuicool.com/articles/aiiYzeR。

一般的经验是在声明变量时对list进行初始化，当涉及数据改变时，通过add或者remove实现。

**2. listview的item内部组件的事件响应**
在具体的工程中，item组件的响应会根据对其使用的Activity（Fragment）的不同而变化，因此，不宜在其内部设定响应事件的具体实现。推荐在adapter中定义接口，将接口暴露给具体的Activity（Fragment），Activity（Fragment）根据具体的业务逻辑进行配置。以前面提到的论坛帖子为例，其包含以下操作：
```
/**
 * 处理Item中控件的点击事件接口
 */
public interface ITopicItemOperation {
    public void topicItemEdit(BBSTopicItem item);
    public void topicItemCollect(BBSTopicItem item);
    public void topicItemReply(BBSTopicItem item);
}
```
然后， 在Activity（Fragment）中实现上述接口：
```
/**
 * 编辑主题贴
 */
@Override
public void topicItemEdit(BBSTopicItem item) {
    // 业务逻辑
}

/**
 * 收藏主题贴
 */
@Override
public void topicItemCollect(BBSTopicItem item) {
    // 业务逻辑
}

/**
 * 回复主题帖
 */
@Override
public void topicItemReply(BBSTopicItem item) {
    // 业务逻辑
}
```
将该实现通过adapter的构造器进行传递，以响应点击事件为例：
```
/**
 * 处理ListView中控件的点击事件
 */
private class TopicItemOnClickListener implements OnClickListener {

    private BBSTopicItem item;
        
    public TopicItemOnClickListener(BBSTopicItem item) {
        this.item = item;
    }
        
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
        case R.id.bbs_topic_edit:
            topicItemOperation.topicItemEdit(item);
            break;
        case R.id.bbs_topic_collect:
            topicItemOperation.topicItemCollect(item);
            break;
        case R.id.bbs_topic_reply:
            topicItemOperation.topicItemReply(item);
            break;
        }
    }
}
```
在点击按钮时，添加TopicItemOnClickListener对象，即可实现不同的Activity（Fragment）对该项功能的复用。

**3.图文混合显示**
在涉及论坛帖子的时候，图文混合显示一种很常见的场景。Android中没有原生支持图文混合显示的控件，github上有一些自定义控件能实现这种需求，百度一下也能发现很多。但此类个性化的需求需要很据项目实际来灵活运用，这里描述一种通过正则来处理的方法。比如，服务端返回的帖子内容如下：

“全新宝马7系上市了，是不是很有气势？ http://img2.tuohuangzu.com/THZ/UserBlog/0/15/2015061810510550085.jpg 不过相比于7系，我还是更喜欢3系的操控，转向非常精确，而且过弯姿势的建立也是非常恰到好处，过弯姿势建立的过早过晚都不好。过早会导致操控措手不及，无法感觉方向打多少，在匝道，有时打多了要在回，回多了又要打。过晚会导致路感缺失，侧倾明显，虚的慌 http://res3.auto.ifeng.com/s/6606/0/3/13309355849880_3.jpghttp://img1.cheshi-img.com/product/1_1024/887/4b25a8df6e8ec.jpg 明天天气不错，去自驾游如何？”

帖子内容为纯文本格式，显示时需要从中提取出图片的链接。这时正则就派上用场了：
```
Pattern p = Pattern.compile("http://[^\\u4e00-\\u9fa5]*?[.]jpg");
Matcher m = p.matcher(text);
            
int lastTextIndex = 0;
while (m.find()) {
    // 设置文本显示
    String textFrag = text.substring(lastTextIndex, m.start());
    if (!textFrag.isEmpty()) {
        layout.addView(getTextView(context, textFrag));
    }
    // 更新最后文本下标
    lastTextIndex = m.end();
                
    // 设置图片显示
    String imageUrl = m.group();
    ImageView imageView = getImageView(context);
    setImageViewDisplay(imageView, imageUrl);
    layout.addView(imageView);
}
if (lastTextIndex < text.length()) {
    String textFrag = text.substring(lastTextIndex, text.length());
    layout.addView(getTextView(context, textFrag));
}
```
这段代码比较简单，只有一处需要说明。在上述帖子内容汇总，后面两张图片的链接是连续的，当正则表达式中包含能接受重复的限定符时，通常的行为是（在使整个表达式能得到匹配的前提下）匹配尽可能多的字符。以这个表达式为例：a.\*b，它将会匹配最长的以a开始，以b结束的字符串。如果用它来搜索aabab的话，它会匹配整个字符串aabab。这被称为贪婪匹配。有时，我们更需要懒惰匹配，也就是匹配尽可能少的字符。前面给出的限定符都可以被转化为懒惰匹配模式，只要在它后面加上一个问号?。这样.*?就意味着匹配任意数量的重复，但是在能使整个匹配成功的前提下使用最少的重复。因此，在上述场景中，想要将连续的两个URL匹配成功，则需要进行懒惰匹配。

**4.ScrollView与ListView的冲突**
如果在ScrollView中嵌套了ListView（原则上应尽量避免这种情况），那么很不幸，可能会遇到以下问题：

 - ListView只显示一行
 - 页面默认不从顶端开始显示
 - ListView滑动事件无法监听

这几个问题都是比较常见的问题了，这里不再赘述其原理，给出比较通用的解决方案：

 1. listview需要手动设置高度，这里给出一个链接：http://www.cnblogs.com/zhwl/p/3333585.html
 2. listview需要设置listview.setFocusable(false);
 3. 重载listview的onInterceptTouchEvent方法，在ACTION_DOWN时通过ScrollView的requestDisallowInterceptTouchEvent方法设置交出ontouch权限，ACTION_CANCEL时再恢复ontouch权限。

再次强调，应尽量**避免ScrollView中嵌套了ListView**。