###SingletonAndLazyInit

经常会遇到需要写一个`单例模式`的问题。这个时候，我们一拍脑门，啊哈，太容易了，于是出现了下面的代码：
```
class Single {
	private int id;
	private String name;
	private String address;
	private Single(int id,String name,String address) {
		this.id = id;
		this.name = name;
		this.address = address;
	}
	
	private static Single s = new Single(1,"ABC","Peiking");
	
	public static Single getInstance() {
		return s;
	}
}
```
搞定，我们拍拍手。

这时，老板说
> 这个太简单了，如果有时候我这个`Single`是像**数据库连接**这类比较重的资源。并不一定会使用到，我不希望一上来直接就初始化了。

于是，我们想，这也容易：
```
class Single {
	private int id;
	private String name;
	private String address;
	private Single(int id,String name,String address) {
		this.id = id;
		this.name = name;
		this.address = address;
	}
	
	private static Single s;
	
	public static Single getInstance() {
	  if(s == null) {
	    s = new Single(1,"ABC","Peiking");
	  }
		return s;
	}
}
```
注意此处，我们加了判断
```
if(s == null) {
	 **   s = new Single(1,"ABC","Peiking");**
	  }
```

一切搞定。

-------------------
过了几天，有人测试说
> 我们按照这个写了个**多线程**应用，跑了一下发现，还是会创建多个Single实例啊。
怎么回事？

于是，我们卷起袖子，打开IDE，DEBUG了一把，发现当一个线程在执行`if(s == null)`时，此时如果s确实为空，会执行下面的`new Single`操作，假设这个new操作比较耗时，甚至比较巧的是，在一个线程在执行new的时候，另一个执行到了判断s是否为空的地方，此时由于s仍然为null,所以第二个线程又一次执行了new操作。

了解了这个之后，我们下定决心搞定他，于是代码中相应的`getInstance`方法改成了这样：
```
	public static synchronized Single getInstance() {
	  if(s == null) {
	    s = new Single(1,"ABC","Peiking");
	  }
		return s;
	}
```
注意，我们加了`synchronized`哦，此时，所有要调用该方法的线程之间是**互斥**的，一个线程执行时，另一个线程只能阻塞。“这下应该没什么问题了”，我们心想。
下班路上，忽然**良心发现**，为了在多线程环境下使用，我竟然让多个线程互斥，这种对效率影响太大了。
于是，第二天上班，又使用了一种很聪明的实现方式：
```
	public static Single getInstance() {
	  if(s == null) {
	    synchronized(Single.class) {
	      if(s == null) {
	        s = new Single(1,"ABC","Peiking"); 
	      }
	    }
	  }
		return s;
	}
```
O了，这招牛X。我**完美的检查了两次**，万无一失。

-------------

某天在技术论坛闲逛，发现一个名为`DCL 相关问题`的讨论。这DCL不就是咱使用的万无一失的**两次检查**吗？于是认真看了下，不看不知道，一看还真吓一跳。**两次检查也是有问题的。**

具体是由于CPU会对这些字节码指令重排序，比如我们上面创建Single对象的过程，可能会包含以下几步：
* 创建Single对象
* 为Single对象各属性赋值
* 将Single对象指向其引用s

 
这个时候，Java虚拟机只要保证最终结果正确即可，并不保证中间是按上面的顺序执行，所以，如果此时的执行顺序是`1、3、2`，那这个时候，多线程环境下，第二个线程将看到第一个线程**创建了一半的对象**，即Single的各个属性可能并没有赋值，此时就被使用了，可能引起其它的问题。

啊。这该如何是好？
庆幸的是JDK1.5及之后，Java内存模型(JMM)得到修正，使用`volatile`关键字可以声明不允许CPU重排序指令，从而解决上面**重排序**的问题。
所以，完整的线程安全代码是这样的：
```
	private volatile static Single s;

	public static Single getInstance() {
	  if(s == null) {
	    synchronized(Single.class) {
	      if(s == null) {
	        s = new Single(1,"ABC","Peiking"); 
	      }
	    }
	  }
		return s;
	}
```
> 也就是我们声明Single对象为volatile的，此时即可解决DCL的问题。

很不错，这下放心了。

不过，如果不采用锁的形式，我们依然可以实现延迟初始化，例如：
```
// Correct lazy initialization in Java
@ThreadSafe
class Foo {
    private static class HelperHolder {
       public static Helper helper = new Helper();
    }
 
    public static Helper getHelper() {
        return HelperHolder.helper;
    }
}
```
即使用内部类的方式，内部类的对象只有在使用时才会初始化，从而可以在无锁的情况下实现**延迟初始化形式的单例**。

```
public class Foo {
    private static class FooHolder { 
        static final Foo foo = new Foo(); 
    } 
    public static Foo getFoo() {
        return FooHolder.foo;
    } 
}
```
