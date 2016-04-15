***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51120048

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

----------


参考资料：[《Thinging in Java》](1)

引言：在较早的Java代码中，是使用suspend（）和resume（）来终止和唤醒线程，但是现在已经被废止了，因为可能导致死锁。stop（）强制终止线程的方式也已经不推荐使用，因为它不释放线程获得的锁，还会产生一些不可预料的后果。下面看看《Thinging in Java》中建议我们如何来终止线程。


----------
##一、线程进入阻塞态的原因

 - 在阻塞态中终止线程，需要做一些处理，让线程跳出阻塞态，所以有必要先来了解一些一个任务进入阻塞态的原因：

  - （1）通过调用sleep（）使线程进入阻塞态（休眠）。
  - （2）通过wait（）使线程挂起进入阻塞态，直到线程收到notify（）的消息，线程才会重新进入就绪态。
  - （3）等待I/O操作的完成
  - （4）等待锁

 - 进入阻塞态的线程，调度器会忽略线程，不会分配任何CPU时间。


----------
##二、通过标识符终止线程

 - 通过设置flag为false的方式，让线程跳出while循环，从而终止线程。（如果线程已经进入while循环，且处于阻塞状态，这种方式将无法立即解除线程阻塞）

```
	static class MyThread extends Thread {

		private volatile boolean isRunning = true;

		@Override
		public void run() {
			while (isRunning) {
				System.out.println("Running");
			}
		}

		public void stopMyThread() {
			isRunning = false;
		}
	}
```

----------
##三、调用interrupt（）

 - 很显然，这是一种更优雅的终止线程方式，配合Thread.interrupted()的检测，在阻塞中通过抛出异常来安全退出线程。

```
	static class MyThread extends Thread {

		@Override
		public void run() {
			try {
				while (!Thread.interrupted()) {
					Thread.sleep(100);
					System.out.println("Running");
				}
			} catch (InterruptedException e) {
				// TODO 结束线程
			} finally {
				// TODO 释放资源
			}
		}
	}

	MyThread myThread = new MyThread();
	myThread.start();
	// 终止线程
	myThread.interrupt();
```

----------

***博客地址：***http://blog.csdn.net/mxm691292118/article/details/51120048

如果对你有帮助，欢迎star和follow哟，保持关注，持续更新。

