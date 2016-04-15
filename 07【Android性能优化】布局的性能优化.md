***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51131912

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

----------

【参考资料】
1、http://developer.android.com/training/improving-layouts/optimizing-layout.html
2、何红辉前辈 -- [《Android开发进阶----从小工到专家》](1)
3、Herve Guihot -- [《Pro Android Apps Performance Optimization》](1)


----------

##一、优化Layout层级
　　APP的每个View和Layout都需要经过Measure、Layout和Draw三个过程，如果布局层级过深，这个过程就会非常耗时。所以减少Layout的层级是优化布局性能的一个重要手段。本文将介绍通过include、merge、ViewStub三种手段来进行Android布局层级优化。

推荐一个布局检测工具：
　　***Hierarchy Viewer ：***能够在程序运行时分析 Layout，可以用这个工具找到 Layout 的性能瓶颈。
　　


----------


##二、include布局重用

　　在复杂的UI界面中，经常会遇到多个界面含有一些相同子布局的情况，如果我们分别定义这些公共元素，如果需要修改，就必须到每个布局文件中分别修改，增加了大量的维护成本。
　　include标签可以帮助我们把公共元素提取出来，作为一个子布局，在每一个需要共用这个子布局的布局中通过include引入即可。

```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <include
        android:id="@+id/sub_layout"
        layout="@layout/layout_item" />

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"></RelativeLayout>

</FrameLayout>

```

　　include标签实现的原理很简单，就是在解析xml的时候，如果检测到include标签，就直接把include引用的布局的RootView添加到include所在的父布局中。

 - 如果要查找被include进来的布局中的某个控件：

  - 如果被include的布局的根元素是merge：只能通过单层findViewById查找
  - 如果被include的布局的根元素不是merge：可以通过单层的findViewById查找；也可以先找到include的布局，在通过被include的布局findViewById。（双层）


----------
##三、merge标签

　　如果子布局的根元素跟父容器的是同一类型，比如都是FrameLayout，那在布局树种将会出现两个连续的每层只有一个元素的FrameLayout，这时候应该使用merge作为子布局的根元素，用于布局合并，减少视图层级。

 - 1、merge只能作为一个布局文件的根元素
 - 2、merge的布局将与父布局共用同一Rootview，如果在一个RelativeLayout中include一个布局文件，被include的布局中的元素将按照RelativeLayout原则来排列元素。
 - 3、由于Activity的视图的父容器是FrameLayout，所以如果通过setContentView设置的布局文件是FrameLayout的单根文件，为了避免布局树种存在两个FrameLayout，被setContentView设置进来的布局文件的根元素应该设置为merge。


----------

##四、ViewStub延迟加载

　　ViewStub是一个不可见的，可以在运行时延迟加载目标布局的，宽高都为0的View，默认不可见不加载，不占用布局空间和系统资源。可以在运行时通过代码控制其可见并加载目标布局。

　　在默认情况下，ViewStub出现在视图书中，起到占位作用，但是不消耗资源，也不绘制，在调用inflate（）后，加载ViewStub引用的布局，然后将ViewStub从布局树中替换出去。

　　***注意：ViewStub引用的布局的根元素不能是merge***

 - **布局的代码如下：**

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <include
        android:id="@+id/layout_content_main"
        layout="@layout/content_main" />

    <ViewStub
        android:id="@+id/comment_stub"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@id/layout_content_main"
        android:layout="@layout/layout_stub" />

</RelativeLayout>

```


 - **延迟加载布局：**

```
final ViewStub viewStub = (ViewStub) findViewById(R.id.comment_stub);

findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
	@Override
	public void onClick(View v) {
		viewStub.inflate();
	}
});
```


----------

***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51131912

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

