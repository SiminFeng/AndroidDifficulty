***博客地址：***http://blog.csdn.net/mxm691292118/article/details/50996938

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

----------



##一、内存泄露
 - 垃圾回收器无法回收原本应该被回收的对象，这个对象就引发了内存泄露。
 - 内存泄露的危害：
  - （1）过多的内存泄露最终会导致内存溢出（OOM）
  - （2）内存泄露导致可用内存不足，会触发频繁GC，不管是Android2.2以前的单线程GC还是现在的CMS和G1，都有一部分的操作会导致用户线程停止（就是所谓的Stop the world），从而导致UI卡顿。


----------


##二、内存溢出（OOM）

 - Android为每个进程设置Dalvik Heap Size阈值，这个阈值在不同的设备上会因为RAM大小不同而各有差异。如果APP想要分配的内存超过这个阈值，就会发生OOM。

 - ActivityManager.getMemoryClass()可以查询当前APP的Heap Size阈值，单位是MB。

 - 在3.x以前，Bitmap分配在Native heap中，而在4.x之后，Bitmap分配在Dalvik或ART的Java heap中。

 - Android 2.x系统，当dalvik allocated + native allocated + 新分配的大小 >= dalvik heap 最大值时候就会发生OOM，也就是说在2.x系统中，考虑native heap对每个进程的内存限制。
 - Android 4.x系统，废除了native的计数器，类似bitmap的分配改到dalvik的java heap中申请，只要allocated + 新分配的内存 >= dalvik heap 最大值的时候就会发生OOM（art运行环境的统计规则还是和dalvik保持一致），也就是说在4.x系统中，不考虑native heap对每个进程的内存限制，native heap只会收到本机总内存（包括RAM以及SWAP区或分页文件）的限制。


----------

##三、如何避免内存泄漏

参考在MDCC 2015中国移动开发者大会上[胡凯](http://hukai.me)前辈的讲述，整理总结。

***1、使用轻量的数据结构***

 - 使用ArrayMap/SparseArray来代替HashMap，ArrayMap/SparseArray是专门为移动设备设计的高效的数据结构。

 - **HashMap实现原理**
  - HashMap内部使用一个默认容量为16的数组来存储数据，采用拉链法解决hash冲突（数组+链表），如下图：
![这里写图片描述](http://img.blog.csdn.net/20160331144232258)

  - Entry存储的内容有key、value、hash值、next指针，通过计算hash(key)%len找到Entry在数组中的位置。
  - ***缺点：***（1）就算没有数据，也需要分配默认16个元素的数组（2）一旦数据量达到Hashmap限定容量的75%，就将按两倍扩容

 - ***SparseArray***
  - 支持int类型，避免自动装箱，但是也只支持int类型的key
  - 内部通过两个数组来进行数据存储的，一个存储key，另外一个存储value
  - 因为key是int，在查找时，采用二分查找，效率高，SparseArray存储的元素都是按元素的key值从小到大排列好的。 （Hashmap通过遍历Entry数组来获取对象）
  - 默认初始size为0，每次增加元素，size++
  - SparseArray中put方法的源码如下：

 - ***ArrayMap***

  - 跟SparseArray一样，内部两个数组，但是第一个存key的hash值，一个存value，对象按照key的hash值排序，二分查找也是按照hash
  - 查找index时，传入key，计算出hash，通过二分查找hash数组，确定index


***2、不要使用Enum***

 - 这点在Google的Android官方培训课程提到过，具体可以参考胡凯前辈的[《 Android性能优化典范（三）》](http://www.csdn.net/article/2015-08-12/2825447-android-performance-patterns-season-3)。


***3、大胖子Bitmap的处理***

 - Bitmap压缩
 - Lru机制处理Bitmap（[下一节博客](http://write.blog.csdn.net/mdeditor#!postId=51028953)会详细讲解），也可以使用那些有名的图片缓存框架。


***4、不要使用String进行字符串拼接***

 - 严格的讲，String拼接只能归结到内存抖动中，因为产生的String副本能够被GC，不会造成内存泄露。

 - 频繁的字符串拼接，使用StringBuffer或者StringBuilder代替String，可以在一定程度上避免OOM和内存抖动。


***5、非静态内部类内存泄露***

 - 在Activity中创建非静态内部类，非静态内部类会持有Activity的隐式引用，若内部类生命周期长于Activity，会导致Activity实例无法被回收。（屏幕旋转后会重新创建Activity实例，如果内部类持有引用，将会导致旋转前的实例无法被回收）。

 - 解决方案：如果一定要使用内部类，就改用static内部类，在内部类中通过WeakReference的方式引用外界资源。

 - 正确的代码示例：

```
static class ImageDownloadTask extends AsyncTask<String, Void, Bitmap> {

        private String url;
        private WeakReference<PhotoAdapter> photoAdapter;

        public ImageDownloadTask(PhotoAdapter photoAdapter) {
            this.photoAdapter = new WeakReference<PhotoAdapter>(photoAdapter);
        }

        @Override
        protected Bitmap doInBackground(String... params) {
            //在后台开始下载图片
            url = params[0];
            Bitmap bitmap = photoAdapter.get().loadBitmap(url);
            if (bitmap != null) {
                //把下载好的图片放入LruCache中
                String key = MD5Tools.decodeString(url);
                photoAdapter.get().put(key, bitmap);
            }
            return bitmap;
        }

        @Override
        protected void onPostExecute(Bitmap bitmap) {
            super.onPostExecute(bitmap);
            //把下载好的图片显示出来
            ImageView mImageView = (ImageView) photoAdapter.get().mGridView.get().findViewWithTag(MD5Tools.decodeString(url));
            if (mImageView != null && bitmap != null) {
                mImageView.setImageBitmap(bitmap);
                photoAdapter.get().mDownloadTaskList.remove(this);//把下载好的任务移除
            }
        }
    }
```



***6、匿名内部类内存泄漏***

 - 跟非静态内部类一样，匿名内部类也会持有外部类的隐式引用，比较常见的情况有，耗时Handler，耗时Thread，都会造成内存泄漏，解决方式也是static+WeakReference，下面给出正确写法。

 - Handler的正确写法：

```
private static class MyHandler extends Handler {

	private final WeakReference<Context> context;

	private MyHandler(Context context) {
		this.context = new WeakReference<Context>(context);
	}

	@Override
	public void handleMessage(Message msg) {
		switch (msg.what) {

		}
	}
}

private final MyHandler mHandler = new MyHandler(this);

private static final Runnable sRunnable = new Runnable() {
	@Override
	public void run() {

	}
};

@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	setContentView(R.layout.activity_home);
	//  发送一个10分钟后执行的一个消息
	mHandler.postDelayed(sRunnable, 600000);
}
```


 - Thread的正确写法：

```
private static class MyThread extends Thread {

	@Override
	public void run() {
		while (true) {
			// TODO 耗时任务
		}
	}
}
    
new MyThread().start();
```


***7、Context持有导致内存泄漏***

 - Activity Context被传递到其他实例中，这可能导致自身被引用而发生泄漏。
 - ***解决：***对于大部分非必须使用Activity Context的情况（创建Dialog的Context必须是Activity Context），应该使用Application Context。


***8、记得注销监听器***

 - 注册监听器的时候会add Listener，不要忘记在不需要的时候remove掉Listener。


***9、资源文件需要选择合适的文件夹进行存放***

 - hdpi/xhdpi/xxhdpi等等不同dpi的文件夹下的图片在不同的设备上会经过scale的处理。例如我们只在hdpi的目录下放置了一张100100的图片，那么根据换算关系，xxhdpi的手机去引用那张图片就会被拉伸到200200。需要注意到在这种情况下，内存占用是会显著提高的。对于不希望被拉伸的图片，需要放到assets或者nodpi的目录下。

***10、谨慎使用static对象***

 - static对象的生命周期过长，应该谨慎使用

***11、谨慎使用单例中不合理的持有***

 - 单例中的对象生命周期与应用一致，注意不要在单例中进行不必要的外界引用持有。如果一定要引用外部变量，需要在外部变量生命周期结束的时候接触引用（赋为null）。

***12、一定要记得关闭无用连接***

 - 在onDestory中关闭Cursor，I/O，数据库，网络的连接用完记得关闭。

***注意：谨慎使用lager heap***

 - 不同的设备有不容的RAM，他们为应用程序设置了不同大小的Heap的阈值。虽然可以通过largeHeap=true的属性来为应用获得一个更大的heap空间，然后通过getLargeMemoryClass()来获取到这个更大的heap阈值。但是你要注意，largeHeap只是为了一些本来就需要大量内存的APP存在，比如图墙和图片编辑软件。所以，不要随意的使用large heap，否则会影响系统整体的用户体验，会使每次gc时间更长。


----------
##四、内存泄露检测
 - 这里介绍LeakCanary，一款非常好用的内存泄露检测工具，安装在手机上，能够通过Log的方式告诉你是哪块代码发生了内存泄露。

 - 使用方法，在Application中install LeakCanary（默认只能检测Activity内容的内存泄露）：

```
public class MyApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```

 - 想要检测更多，首先注册一个RefWatcher：

```
public class MyApplication extends Application {

    private static RefWatcher sRefWatcher;

    @Override
    public void onCreate() {
        super.onCreate();
        sRefWatcher = LeakCanary.install(this);
    }

    public static RefWatcher getRefWatcher() {
        return sRefWatcher;
    }
}
```

 - 然后对某个可能发生泄露的占用大内存的对象进行监测：
```
MyApplication.getRefWatcher().watch(sLeaky);
```

- 对Fragment、BroadcastReceiver、Service进行监测：
```
public class MyFragment extends Fragment {
    @Override
    public void onDestroy() {
        super.onDestroy();
        MyApplication.getRefWatcher().watch(this);
    }
}
```

 - 具体就不多说，大家去LeakCanary的Github上看就行了，地址是：https://github.com/square/leakcanary



----------
##参考文献
 - [Andorid内存优化之OOM](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0920/3478.html)
 - [Android性能优化典范（一）](http://www.csdn.net/article/2015-01-20/2823621-android-performance-patterns)
 - [Android性能优化典范（二）](http://www.csdn.net/article/2015-04-29/2824583-android-performance-patterns-season-2)
 - [Android性能优化典范（三）](http://www.csdn.net/article/2015-08-12/2825447-android-performance-patterns-season-3)
 - [Android性能优化之内存篇](http://hukai.me/android-performance-memory/)
 - [Android官方学习教程](http://hukai.me/android-training-course-in-chinese/graphics/displaying-bitmaps/manage-memory.html)


----------

***博客地址：***http://blog.csdn.net/mxm691292118/article/details/50996938

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

