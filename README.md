# Andorid难点整理
Andorid学习过程中的重难点整理，包括个人的一些读书笔记和博客

还会附上一些最近内推面试遇到的面试题（大三菜鸟一枚，刚拿到阿里巴巴和百词斩的内推offer）

如果大家需要修正错误或补充，可以pull request或issues

如果你觉得对你有帮助的话，希望可以star活follow一下哟，我会持续保持更新


#联系方式
我的CSDN：http://blog.csdn.net/mxm691292118?viewmode=list

我的微信：mxm691292118


#目录

[00【学习清单】我的Android学习清单](https://github.com/miomin/AndroidDifficulty/blob/master/00%E3%80%90%E5%AD%A6%E4%B9%A0%E6%B8%85%E5%8D%95%E3%80%91%E6%88%91%E7%9A%84Android%E5%AD%A6%E4%B9%A0%E6%B8%85%E5%8D%95.md)

[01【深入理解JVM】JVM内存分块和垃圾收集算法（HotSpot）](https://github.com/miomin/AndroidDifficulty/blob/master/01%E3%80%90%E7%90%86%E8%A7%A3JVM%E3%80%91JVM%E5%86%85%E5%AD%98%E5%88%86%E5%9D%97%E5%92%8C%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E7%AE%97%E6%B3%95%EF%BC%88HotSpot%EF%BC%89.md)

[02【理解JVM】深入浅出JVM垃圾收集（HotSpot）](https://github.com/miomin/AndroidDifficulty/blob/master/02%E3%80%90%E7%90%86%E8%A7%A3JVM%E3%80%91%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAJVM%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%EF%BC%88HotSpot%EF%BC%89.md)

[03【Android性能优化】内存泄露和内存溢出（OOM）的引发原因及优化方案](https://github.com/miomin/AndroidDifficulty/blob/master/03%E3%80%90Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E3%80%91%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E5%92%8C%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%EF%BC%88OOM%EF%BC%89%E7%9A%84%E5%BC%95%E5%8F%91%E5%8E%9F%E5%9B%A0%E5%8F%8A%E4%BC%98%E5%8C%96%E6%96%B9%E6%A1%88.md)

[04【Android性能优化】Android图片加载方案--Bitmap的内存管理和优化方案](https://github.com/miomin/AndroidDifficulty/blob/master/04%E3%80%90Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E3%80%91Android%E5%9B%BE%E7%89%87%E5%8A%A0%E8%BD%BD%E6%96%B9%E6%A1%88--Bitmap%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%92%8C%E4%BC%98%E5%8C%96%E6%96%B9%E6%A1%88.md)

[05【Thinking in Java读书笔记】如何安全的终止线程](https://github.com/miomin/AndroidDifficulty/blob/master/05%E3%80%90Thinking%20in%20Java%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0%E3%80%91%E5%A6%82%E4%BD%95%E5%AE%89%E5%85%A8%E7%9A%84%E7%BB%88%E6%AD%A2%E7%BA%BF%E7%A8%8B.md)

[06【Java并发编程】对比synchronized和Lock](https://github.com/miomin/AndroidDifficulty/blob/master/06%E3%80%90Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E3%80%91%E5%AF%B9%E6%AF%94synchronized%E5%92%8CLock.md)

[07【Android性能优化】布局的性能优化](https://github.com/miomin/AndroidDifficulty/blob/master/07%E3%80%90Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E3%80%91%E5%B8%83%E5%B1%80%E7%9A%84%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96.md)

[08【Java数据结构】Hashmap与Hashtable源码阅读笔记](https://github.com/miomin/AndroidDifficulty/blob/master/08%E3%80%90Java%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E3%80%91Hashmap%E4%B8%8EHashtable%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0.md)

[09【码上杂谈】关于之前对Server上允许的最大TCP连接数理解错误的更正](https://github.com/miomin/AndroidDifficulty/blob/master/09%E3%80%90%E7%A0%81%E4%B8%8A%E6%9D%82%E8%B0%88%E3%80%91%E5%85%B3%E4%BA%8E%E4%B9%8B%E5%89%8D%E5%AF%B9Server%E4%B8%8A%E5%85%81%E8%AE%B8%E7%9A%84%E6%9C%80%E5%A4%A7TCP%E8%BF%9E%E6%8E%A5%E6%95%B0%E7%90%86%E8%A7%A3%E9%94%99%E8%AF%AF%E7%9A%84%E6%9B%B4%E6%AD%A3.md)




