通过MAT来分析一处潜在的内存泄露的问题，来帮助你使用MAT工具做内存泄露的分析
=========================

阅读前准备
----------------------
内存分析是基于[Android-memory-leak-case](https://github.com/zhuchen1109/Android-memory-leak-case) 提供的demo，所以请先理解此case。

看完这个case后，你可能有个疑惑，这些都是分析结果，那如何去验证它就有内存问题呢。
下面我们一起来通过mat工具来分析内存问题！

<b>本文提供的是一种思路</b>
----------------------

二个问题！
----------------------

1、怎么去发现有内存泄露问题？

2、若发现有内存泄露问题，改如何去分析是哪里出问题了？


如何解决？
----------------------

1、通常有一个简单的办法，就是不断的操作你觉得可能导致内存泄露的操作，加重他的内存泄露大小，然后用DDMS提供的Update Heap查看操作前和操作后的内存有没有异常增加。
下面给出[Android-memory-leak-case](https://github.com/zhuchen1109/Android-memory-leak-case) 工程内存泄露的heap截图

操作前heap实例分析图：
![Android-memory-leak-case](https://raw.githubusercontent.com/zhuchen1109/Android-memory-leak-case/master/screenshot/%E6%AD%A3%E5%B8%B8%E6%83%85%E5%86%B5%E4%B9%8BHeap%E6%88%AA%E5%9B%BE.jpg)

操作后heap实例分析图：
![Android-memory-leak-case](https://raw.githubusercontent.com/zhuchen1109/Android-memory-leak-case/master/screenshot/%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E4%B9%8BHeap%E6%88%AA%E5%9B%BE.JPG)

通过比对，很明显发现，经过多次操作后，内存明显变大了很多。从这个得出结果，操作中导致了大量的内存泄露


2、现在我们知道这个工程是有内存问题的，那么我们来分析这第二个问题！

使用DDMS提供的Dump HPROF File获取.hprof文件，即当前应用程序的内存快照，并用eclipse提供的mat插件打开这个文件

打开之后，选择 "Histogram" 进入类列表直方图，这是显示所有类占用的内存情况，如下图：

![Android-memory-leak-case](https://raw.githubusercontent.com/zhuchen1109/Android-memory-leak-case/master/screenshot/MAT%E5%88%86%E6%9E%90%E5%9B%BE1.jpg)

从图中可以明显看到MainActivity占有的内存很大，约90MB。到这里你是不是觉得，哇！就是这出问题了！其实不是...这个是当前栈顶的MainActivity实例，可能是它有内存使用问题，但不能以此来断定它有内存泄露问题。
我们应该往下看，接下来有个醒目的实例MainActivity$1，它也占有了大量内存，这就不正常了，他是被finish的实例，内存还占用这么多，初步判断这地方出现内存泄露了。

下面我们看看MainActivity$1的被引用关系，如下图：

![Android-memory-leak-case](https://raw.githubusercontent.com/zhuchen1109/Android-memory-leak-case/master/screenshot/MAT%E5%88%86%E6%9E%90%E5%9B%BE2.jpg)

![Android-memory-leak-case](https://raw.githubusercontent.com/zhuchen1109/Android-memory-leak-case/master/screenshot/MAT%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E6%88%AA%E5%9B%BE.jpg)

通过截图，可以找到其引用关系是 AsynckTask->ViewRootImpl的静态变量RunQueue->...->MainActivity$1
最终使MainActivity$1无法被GC回收，导致内存泄露。

到这里，你基本上已能清楚的看到，为什么会出现[Android-memory-leak-case](https://github.com/zhuchen1109/Android-memory-leak-case) 的问题了。