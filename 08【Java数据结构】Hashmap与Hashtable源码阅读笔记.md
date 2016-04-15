***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51107039

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

----------

引言：这几天在捣鼓Hashmap跟Hashtable的源码，其中关注的 比较多的就是他两计算在Entry[]数组中index的方法到底有什么区别。

----------

 - Hashmap跟Hashtable的实现原理比较类似，借用一张其他地方偷来的图。
![这里写图片描述](http://dl.iteye.com/upload/attachment/177479/3f05dd61-955e-3eb2-bf8e-31da8a361148.jpg)

 - 可以看到，都是采用外拉链的方式来实现元素存储，底层是数组+链表实现，原理都不说了，学过数据结构中hash冲突解决的同学应该都能理解。


----------


***实现的关键在于如何通过key来计算对应value应该存放到数组中的位置，下面具体来看看。***

 - ***HashTable***的做法：[index = (hash & 0x7FFFFFFF) % tab.length;](1) 这相当于直接将hash值对数组长度取模（跟0x7FFFFFFF做&操作是为了保证hashcode的值为正数）。HashTable中数组的初始size为11，每次扩容都按照newsize = oldsize*2+1来计算。通过取模来计算index的值，从概率上来讲，保证了节点在数组上分配比较均匀（形成尽量短的拉链，有利于提高查询效率），但是取模操作的消耗是比较大的。
![这里写图片描述](http://img.blog.csdn.net/20160409202848452)


----------


  - ***HashMap***的做法则非常巧妙，[index = hash & (tab.length - 1)](1)。这有什么精妙之处呢，首先，Hashmap要求数组的size为2的幂乘，比如16，32，64，仔细看，当数组大小为16的时候，tab.length-1=15，在内存中的表示是00001111，将hash值与00001111做&（位与）操作后，会将hash值除了后四位全部抹为0，只保留了后四位，这样的方式完成了跟index = (hash & 0x7FFFFFFF) % tab.length一样的效果，就是取模。但是位运算的效率比取模操作高得多，也就是说HashMap的index计算方式要比Hashtable快得多。（Hashmap的初始size是16，每次扩容按照newsize = oldsize*2）
![这里写图片描述](http://img.blog.csdn.net/20160409203330084)


----------


  - 写Android的小伙伴可以在Android Studio中打开HashTable的源码看看，Google很明显的改动了Hashtable的index计算方式，改成跟Hashmap一样啦！
![这里写图片描述](http://img.blog.csdn.net/20160409201704697)





----------

***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51107039

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

