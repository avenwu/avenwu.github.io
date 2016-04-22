1.静态工厂方法代替构造器
	优势：
	* 有名称（相比构造器类名，更具可读性）
	* 不必每次都创建新的实例对象（减少实例对象，如Boolean.valueOf(), Integer.valueOf()等会特别优化返回对象）
	* 返回类型可以是任意子类型（如面向接口，隐藏具体实现类）
	* 创建参数化类型时简化代码（带泛型参数<T, K<X, Y>很复杂时，实例化会很冗长）

	缺点：
	* 如果没有共有构造器，则无法子类化
	* 和其他静态方法没区别，不容易被标示出来可以实例化对象

自动装箱相关实现类（Integer，Boolean，Float。。。），collection集合实现

5.避免创造不必要的对象
	* 对象不可变时总是可以复用
	举例：
	String a = "Hello world"
	String b = "Hello world"
	a==b >true, 
	此种实例化为优化版本，并没有直接调用构造器，实际会复用内充缓存中已存在的string，java language specification 3.10.5细节阐述
	类似的还有一些基础类型的包裹类的工程方法；HashMap的entrySet, keySet等
	显示调用构造器总是会开辟新的内存，重新实例化

6.消除过期的对象应用
	内存泄露：
	* 自己管理内存的的类，数组，栈的可用范围缩小了，未及时清理过期引用；
	* 缓存，
	* 监听器/回调，显示取消监听

	借助Heap Profile
7. Object里的通用方法
	equal的正确重载：对称，传递，一致，同时重载hasCode
	float， double比较用Float。compare,Double。compare

8.hasCode重载
	通常针对关键与分别计算散列码，然后通过下面的运算夹河处理
	result = 17；//初始值17任选的，不选0是避免散列码为零是对散列值没影响
	result = 31 * result + c
	乘数选31是因为他是奇素数，VM会自动优化：31 * i = (i << 5) - i
	不同类型关键域求散列码如下：
	boolean =》 f?1:0
	byte,char,short,int => (int)f
	long => (int)(f^(f>>>32))
	float => Float.floatToIntBits(f)
	double =》 Double.doubleToLongBits(f), 在按long处理
	对象，递归调用hasCode
	数组 =》 Arrays.hashCode()

11.谨慎的重载clone
	递归拷贝和非递归形式（用迭代摸底递归，减少栈层次，避免栈溢出），例如HashTable的clone

26. Collections集合实现了解，concurrent并发包了解，nio包了解
69. Executor framework, Concurrent Collection, 同步器Synchronizer(CountDownLatch, Semaphore)
71. 延迟初始化
	静态域用static hodler模式，实例域用双重检查（加临时变量）