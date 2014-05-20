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

>`synchronized`无论是修饰在方法上，还是方法内部的`同步块`，其本质都需要**锁定一个对象**。如果`synchronized`修饰在static方法或者同步块内锁定的是类.class，这两者是互斥的。

例如上面的：
```
	public static synchronized void staticSync() {
		System.out.println("static sync");
		
		System.out.println("static sync");
	}
	public void blockClassSync() {
		synchronized(LockTest.class) {
			System.out.println("block sync");
		}
	}
```
这两个方法由于都需要锁定LockTest这个类的`Class对象`，因此此处两者互斥。

而如果一个线程执行对象的synchronized方法，另一个请求类的synchronized方法，这两者是不互斥的。因为这两个方法要锁定的对象并不是同一个，可以一起执行。
例如：
```
	public static synchronized void staticSync() {
		System.out.println("static sync");
		
		System.out.println("static sync");
	}
	public synchronized void objSync() {
		System.out.println("obj sync");
		System.out.println("obj sync");
	}
```
那下面的两个方法是否可以并行执行呢？
```
	public void blockSync() {
		synchronized(o) {
			System.out.println("block sync");
		}
	}
	public synchronized void objSync() {
		System.out.println("obj sync");
		System.out.println("obj sync");
	}
```
当然也是可以的，blockSync要锁定的是`对象o`，而objSync方法要锁定的是LockTest的一个对象，这两者也不是同一个，因此可以同时执行。
