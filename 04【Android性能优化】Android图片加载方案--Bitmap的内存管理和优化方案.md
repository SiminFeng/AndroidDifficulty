***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51028953

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

----------

***写在前面：***笔者的上一篇博文有提到过，如果不恰当的使用Bitmap，很容易造成OOM。这篇博文就来谈谈应该如何正确的管理Bitmap的内存，以及优化策略。

***参考：*** Google官方教程 -- [《Android Training Ccourse》](http://developer.android.com/training/displaying-bitmaps/index.html)


----------


###一、加载按显示需要的比例缩小的图片

***1、先来说说屏幕密度***

 - 在Android中，Imageview控件的长宽单位一般设置为dp/dip，而不用px。这样做的原因，是因为dp/dip与屏幕像素密度无关，px与屏幕密度有关。在Android中，规定以160dpi为基准，1dip=1px，如果密度是320dpi，则1dip=2px。（所以，同一个imageview，在不同的设备上所显示的长宽的像素是不同的，我们需要根据像素不同，来按比例压缩大图）

 - 如果把一个大小为1024x1024像素的图片显示到大小为512x512像素的ImageView上吗，就没有必要加载整张原图到Bitmap中。

 - 为了告诉解码器去加载一个缩小比例是多少的图片到内存中，需要在BitmapFactory.Options 中设置 inSampleSize 的值。例如, 一个分辨率为2048x2048的图片，如果设置 inSampleSize 为4，那么会产出一个大约512x512大小的Bitmap。

 - 下面的代码是动态获取一个ImageView的长宽像素：
**注意：**返回的是像素值（px），而不是dp/dip
```
	int reqHeight = iv.getLayoutParams().height;
	int reqWidth = iv.getLayoutParams().width;
```



***2、压缩图片***

 - 一般来说，加载本地资源图片才需要压缩，加载网络图片，应该让服务器按需压缩，一方面节省流量，另一方面提高加载流畅度
 - 可以通过下面的代码计算inSampleSize的值，就是需要压缩的倍数：

```
//reqWidth和reqHeight是需要显示的Imageview的width和height
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // height和width图片长宽的像素
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```

 - **通过下面的方法可以获得压缩后的Bitmap：**
***注意：***可以通过设置 inJustDecodeBounds 属性为true可以在解码的时候避免内存的分配，它会返回一个null的Bitmap，但是可以获取到 outWidth, outHeight 与 outMimeType。确定好压缩比例后，再将inJustDecodeBounds设置为false。
```
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // 计算inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // 根据inSampleSize压缩图片
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```


----------


###二、Bitmap缓存
***1、缓存类LruCache介绍***

 - 在Android3.0之后，一般使用LruCache来缓存Bitmap，它使用一个强引用的LinkedHashMap保存最近引用的对象，并且在缓存超出设置大小的时候剔除（evict）最少使用到的对象。
 - LinkedHashMap会根据LRU算法来排列对象的顺序，新加入的对象添加到头部，刚被使用过的对象也被移动到头部，所以在链表尾部的对象是最久没有被使用过的，一旦链表满了，有新对象加入，就会删除链表尾部的对象。

***2、如何给LruCache设置一个合适的大小？***

 - LruCache太大的话，容易造成OOM或者分配给应用的剩余内存不够用，LruCache大小的设置，应该考虑下面的因素：
  - （1）应用允许的最大内存是多少？剩下了多少可用的内存?
  - （2）多少张图片会同时显示在屏幕上？
  - （3）设备的屏幕密度是多少？显示图片的像素是多少？
  - （4）可以根据访问频率给Bitmap分组，为不同的Bitmap组设置不同大小的LruCache对象。


***3、一个LruCache使用的完整例子***

 - 代码参考了一个大神的代码，并修真了一些写的不对和不好的地方。

```
public class PhotoAdapter extends BaseAdapter implements AbsListView.OnScrollListener {

    // 从network下载图片的线程集合
    private List<ImageDownloadTask> mDownloadTaskList;
    private LruCache<String, Bitmap> mLruCache;

    // 引用外部的变量
    private WeakReference<GridView> mGridView;
    private WeakReference<List<String>> urls;
    private WeakReference<Context> mContext;


    // 可见项的第一项的index
    private int mFirstVisibleIndex;

    // 可见项的个数
    private int mVisibleItemCount;

    // 是不是第一次打开Activity
    private boolean isFirstOpen = true;

    public PhotoAdapter(Context context, GridView mGridView, List<String> urls) {
        this.mContext = new WeakReference<Context>(context);
        this.urls = new WeakReference<List<String>>(urls);
        this.mGridView = new WeakReference<GridView>(mGridView);
        this.mGridView.get().setOnScrollListener(this);
        mDownloadTaskList = new ArrayList<>();
        // 初始化图片缓存池
        initCache();
    }

    private void initCache() {

        // 获取应用的max heap size
        final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

        // Android官方教学文档推荐LruCache的size为heap size的1/8
        int cacheSize = maxMemory / 8;

        mLruCache = new LruCache<String, Bitmap>(cacheSize) {
            @Override
            protected int sizeOf(String key, Bitmap bitmap) {
                if (bitmap != null) {
                    return bitmap.getByteCount() / 1024;
                }
                return 0;
            }
        };
    }

    @Override
    public int getCount() {
        return urls.get().size();
    }

    @Override
    public Object getItem(int position) {
        return urls.get().get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {

        viewHolder holder = null;

        if (convertView == null) {
            convertView = LayoutInflater.from(mContext.get()).inflate(R.layout.layout_item, parent, false);
            holder = new viewHolder();
            holder.mImageView = (ImageView) convertView.findViewById(R.id.imageView);
            holder.mTextView = (TextView) convertView.findViewById(R.id.textView);
            convertView.setTag(holder);
        } else {
            holder = (viewHolder) convertView.getTag();
        }

        String url = urls.get().get(position);
        //imageview与url绑定，防止错乱显示
        holder.mImageView.setTag(MD5Tools.decodeString(url));
        holder.mTextView.setText("第" + position + "项");

        if (!holder.mImageView.getTag().equals(url)) {
            showImageView(holder.mImageView, url);
        }

        return convertView;
    }

    /**
     * convertView复用
     */
    private class viewHolder {
        ImageView mImageView;
        TextView mTextView;
    }

    /**
     * 给ImageView设置Bitmap
     */
    private void showImageView(ImageView imageView, String url) {

        // 对url进行md5编码
        String key = MD5Tools.decodeString(url);
        // 先从cache中找bitmap缓存
        Bitmap bitmap = get(key);

        if (bitmap != null) {
            // 如果缓存命中
            imageView.setImageBitmap(bitmap);
        } else {
            // 如果cache miss
            imageView.setBackgroundResource(R.color.color_five);
        }
    }

    /**
     * 将Bitmap put 到 cache中
     */
    private void put(String key, Bitmap bitmap) {

        if (get(key) == null) {
            mLruCache.put(key, bitmap);
        }
    }

    /**
     * 在Cache中查找bitmap，如果miss则返回null
     */
    private Bitmap get(String key) {
        return mLruCache.get(key);
    }

    /**
     * 从网络下载图片
     */
    private Bitmap loadBitmap(String urlStr) {

        HttpURLConnection connection = null;
        Bitmap bitmap = null;
        try {
            URL url = new URL(urlStr);
            connection = (HttpURLConnection) url.openConnection();
            connection.setConnectTimeout(5000);
            connection.setReadTimeout(5000);
            connection.setDoInput(true);
            connection.connect();
            if (connection.getResponseCode() == HttpURLConnection.HTTP_OK) {
                InputStream mInputStream = connection.getInputStream();
                bitmap = BitmapFactory.decodeStream(mInputStream);
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                connection.disconnect();
            }
        }
        return bitmap;
    }

    /**
     * 取消所有的下载任务
     */
    public void cancelAllTask() {

        if (mDownloadTaskList != null) {
            for (int i = 0; i < mDownloadTaskList.size(); i++) {
                mDownloadTaskList.get(i).cancel(true);
            }
        }
    }

    /**
     * 加载可见项的图片
     */
    private void loadVisibleBitmap(int mFirstVisibleItem, int mVisibleItemCount) {

        for (int i = mFirstVisibleItem; i < mFirstVisibleItem + mVisibleItemCount; i++) {
            final String url = urls.get().get(i);
            String key = MD5Tools.decodeString(url);
            Bitmap bitmap = get(key);
            ImageView mImageView;
            if (bitmap != null) {
                //缓存中存在该图片的话就设置给ImageView
                mImageView = (ImageView) mGridView.get().findViewWithTag(MD5Tools.decodeString(url));
                if (mImageView != null) {
                    mImageView.setImageBitmap(bitmap);
                }
            } else {
                //不存在的话就开启一个异步线程去下载
                ImageDownloadTask task = new ImageDownloadTask(this);
                mDownloadTaskList.add(task);
                task.execute(url);
            }
        }
    }

    /**
     * 从网络下载图片的异步task
     */
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

    /**
     * 监听GridView的滑动状态
     */
    @Override
    public void onScrollStateChanged(AbsListView view, int scrollState) {

        //GridView停止滑动时，去加载可见项的图片
        if (scrollState == SCROLL_STATE_IDLE) {
            loadVisibleBitmap(mFirstVisibleIndex, mVisibleItemCount);
        } else {
            //GridView开始滑动时，取消所有加载任务
            cancelAllTask();
        }
    }

    /**
     * 监听并更新GridView滑动过程中的可见项
     */
    @Override
    public void onScroll(AbsListView view, int firstVisibleIndex, int visibleItemCount, int totalItemCount) {

        mFirstVisibleIndex = firstVisibleIndex;
        mVisibleItemCount = visibleItemCount;

        // 第一次打开，加载可见项
        if (isFirstOpen && visibleItemCount > 0) {
            loadVisibleBitmap(mFirstVisibleIndex, mVisibleItemCount);
            isFirstOpen = false;
        }
    }
}
```



***4、Andorid3.0之前的Bitmap管理方法（参考自[xiaanming](http://blog.csdn.net/xiaanming/article/details/41084843)的博客）***

 - 在Android3.0之前，Bitmap对象与数据是分开存储的，Bitmap对象存储在Java heap中，而像素数据存储在Native Memory中，Java虚拟机的垃圾回收机制不会主动回收Native Memory中的对象，需要在Bitmap不需要使用的时候，主动调用recycle()方法来释放，而在Android3.0之后，Bitmap的像素数据和Bitmap对象都存放在Java Heap中，所以不需要手动调用recycle()来释放，垃圾收集器会处理。
 - 应该在什么时候去调用recycle()方法呢？可以用引用计数算法，用一个变量来记录Bitmap显示情况，如果Bitmap绘制在View上面displayRefCount加一, 否则就减一, 只有在displayResCount为0且Bitmap不为空且Bitmap没有调用过recycle()的时候，才调用recycle()，下面用BitmapDrawable类来包装下Bitmap对象，代码如下：

```
public class RecycleBitmapDrawable extends BitmapDrawable {
	private int displayResCount = 0;
	private boolean mHasBeenDisplayed;

    public RecycleBitmapDrawable(Resources res, Bitmap bitmap) {
        super(res, bitmap);
    }
	
    
    /**
     * @param isDisplay
     */
	public void setIsDisplayed(boolean isDisplay){
		synchronized (this) {
			if(isDisplay){
				mHasBeenDisplayed = true;
				displayResCount ++;
			}else{
				displayResCount --;
			}
		}
		
		checkState();
	}
	
	/**
	 * 检查图片的一些状态，判断是否需要调用recycle
	 */
	private synchronized void checkState() {
	    if (displayResCount <= 0 && mHasBeenDisplayed
	            && hasValidBitmap()) {
	        getBitmap().recycle();
	    }
	}
	
	
	/**
	 * 判断Bitmap是否为空且是否调用过recycle()
	 * @return
	 */
	private synchronized boolean hasValidBitmap() {
	    Bitmap bitmap = getBitmap();
	    return bitmap != null && !bitmap.isRecycled();
	}

}
```

- 还需要一个自定义的ImageView，重写了setImageDrawable()方法，在这个方法中我们先获取ImageView上面的图片，然后通知之前显示在ImageView的Drawable不在显示了，Drawable会判断是否需要调用recycle()，代码如下：

```
public class RecycleImageView extends ImageView {
  
    public RecycleImageView(Context context) {  
        super(context);  
    }  
  
    public RecycleImageView(Context context, AttributeSet attrs) {  
        super(context, attrs);  
    }  
  
    public RecycleImageView(Context context, AttributeSet attrs, int defStyle) {  
        super(context, attrs, defStyle);  
    }  
  
    @Override  
    public void setImageDrawable(Drawable drawable) {  
        Drawable previousDrawable = getDrawable();  
        super.setImageDrawable(drawable);  
          
        //显示新的drawable  
        notifyDrawable(drawable, true);  
  
        //回收之前的图片  
        notifyDrawable(previousDrawable, false);  
    }  
  
    @Override  
    protected void onDetachedFromWindow() {  
        //当View从窗口脱离的时候,清除drawable  
        setImageDrawable(null);  
        super.onDetachedFromWindow();  
    }  
  
    /** 
     * 通知该drawable显示或者隐藏 
     *  
     * @param drawable 
     * @param isDisplayed 
     */  
    public static void notifyDrawable(Drawable drawable, boolean isDisplayed) {  
        if (drawable instanceof RecycleBitmapDrawable) {  
            ((RecycleBitmapDrawable) drawable).setIsDisplayed(isDisplayed);  
        } else if (drawable instanceof LayerDrawable) {  
            LayerDrawable layerDrawable = (LayerDrawable) drawable;  
            for (int i = 0, z = layerDrawable.getNumberOfLayers(); i < z; i++) {  
                notifyDrawable(layerDrawable.getDrawable(i), isDisplayed);  
            }  
        }  
    }  
  
}
```

 - 具体的使用方法如下
```
ImageView imageView = new ImageView(context);  
        imageView.setImageDrawable(new RecycleBitmapDrawable(context.getResource(), bitmap));
```

----------

***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51028953

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

