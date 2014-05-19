###JAVA并发中Monitor锁

对于Monitor锁，有几个相关的问题
```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class LockTest {

	private Object o = new Object();
	public static void main(String[] args) {
		
		ExecutorService exec = Executors.newFixedThreadPool(6);
		for(int i=0;i<3;i++) {
			exec.execute(new Runnable() {
				@Override
				public void run() {
					LockTest.staticSync();
				}});
		}
		final LockTest lt = new LockTest();
		for(int i=0;i<3;i++) {
			exec.execute(new Runnable() {
				@Override
				public void run() {
					lt.blockClassSync();
					lt.blockSync();
					lt.objSync();
				}});
		}
	}
	
	public static synchronized void staticSync() {
		System.out.println("static sync");
		
		System.out.println("static sync");
	}

	public synchronized void objSync() {
		System.out.println("obj sync");
		System.out.println("obj sync");
	}
	
	public void blockSync() {
		synchronized(o) {
			System.out.println("block sync");
		}
	}
	
	public void blockClassSync() {
		synchronized(LockTest.class) {
			System.out.println("block sync");
		}
	}
}

```

看上面这段代码，如果`blockClassSync`执行时，此时有另外的线程去请求`blockSync`，此时第二个线程是否会阻塞，此时第三个线程去请求`staticSync`方法，此时是否会阻塞，也就是说哪些情况下，方法的请求是**互斥**的呢？
