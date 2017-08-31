title: Java 知识点
date: 2017-03-26 17:01:59
tags: [Java,技术]
---

> 在本科时没有好好听老师讲课，现在只能重新再学学Java，😭。

## 我的java环境 

> java version "1.8.0_121"   
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)  
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)

## ClassLoader   

**bootstrap classloader**: 加载java的核心库的类。   
**ExtClassLoader**: 加载扩展的java库中的类。   
**AppClassLoader**: 加载应用中的类。

<!--more-->

## i++  

### 1. 前++和后++

{% codeblock lang:java %}

public class Main {
	static {
		//局部变量，不影响下面的x
		int x = 5;
	}
	//jvm会初始化全局变量x，y为0。局部变量则无初始值。
	static int x, y;

	public static void main(String[] args) {
		//x变为-1
		x--;
		
		myMethod();
		
		//1. 此时x=1,y=0
		//2. y++的++先不执行，而是y先参与运算，结果为2
		System.out.println(x + y++ + x); //2
	}

	public static void myMethod() {
		//1. x++的++先不运算，还是原始的x作为参数，所以此刻相当于x + ++x
		//2. ++x的++会执行，所以x此时变为0
		//3. 此时x为0，y = 0+0=0
		//4. x++在运算结束后开始执行，所以x为1
		y = x++ + ++x;
		
		System.out.println(x+" "+y); // x=1,y=0
	}
}

{% endcodeblock %}


### 2. ++的缓存机制

{% codeblock lang:java %}

public class Main {
	public static void main(String[] args) {
		int j = 0;
		for (int i = 0; i < 100; i++) {
			//执行顺序拆解为如下
			//temp=j
			//j++
			//j=temp
			j = j++;
		}

		System.out.println(j); //0
	}
}

{% endcodeblock %}

### 3. 各种++

{% codeblock lang:java %}

public class Main {
	public static void main(String[] args) {
		int i = 0;
		//1. 先执行i + ++i，++i为1，所以temp = 1 +1 =2
		//2. i++执行，变为2，但是没什么作用  
		//3. i=temp，结果是2
		i = i++ + ++i;
		
		int j = 0;
		//1. ++j 使得j为1，1号j++的++先不执行，所以temp= j + j= 1+1
		//2. 加法函数执行完了，所以1号j++执行，j=2
		//3. temp = temp + j++，2号j++同样先不执行，所以temp= 1+1+2
		//4. 2号j++执行，j=3
		//5. temp =temp +j++，3号j++同理，temp=1+1+2+3
		//6. 3号j++执行，得到j=4
		//7. j=temp，j最后变为7
		j = ++j + j++ + j++ + j++;
		
		
		int k = 0;
		//1. 1号k++先不执行，k为0，temp=0。
		//2. 1号k++执行，k=1
		//3. 到2号k++，该++不执行，所以temp=0+1
		//4. 2号k++执行，k=2
		//5. 到3号k++，temp=0+1+2
		//6. 3号k++执行，k=3，
		//7. ++k后，k=4
		//8. temp=0+1+2+4
		//9. k=temp=7
		k = k++ + k++ + k++ + ++k;
		
		int h = 0;
		
		//1. 第一个++h执行后，temp=1
		//2. 2号++h执行后temp=1+2=3
		//3. h=temp=3
		h = ++h + ++h;
		
		int p1 = 0, p2 = 0;
		int q1 = 0, q2 = 0;
		//++p1后p1=1，q1=p1=1
		q1 = ++p1;
		//temp=p2=0，p2++后p2=1
		//q2=temp=0
		q2 = p2++;
		System.out.println(" i " + i); //2
		System.out.println(" j " + j); //7
		System.out.println(" k " + k); //7
		System.out.println(" h " + h); //3
		System.out.println(" p1 " + p1); //1
		System.out.println(" p2 " + p2); //1
		System.out.println(" q1 " + q1); //1
		System.out.println(" q2 " + q2); //0
	}
}

{% endcodeblock %}

## 数据类型转换 

### 1. 基本数据类型的转换   
低级到高级：分别为（byte，short，char）< int < long < float < double。

下面的程序是合法的。  
{% codeblock lang:java %}
public class Main {
	public static void main(String[] args) {
		byte a = 1;
		char c = 'c';

		short s = a;
		// 不合法 s=c, c=s;
		int i = s;
		i = c;
		long l = i;
		l = c;
		float f = l;
		f = c;
		double d = f;
	}
}
{% endcodeblock %}

下面程序中，有一个错误：
{% codeblock lang:java %}

short s=1;
s+=1;
s=s+1; //不合法，s+1为int型

{% endcodeblock %}

## 位移操作   

int为32位，所以位移操作会先模运算32。

{% codeblock lang:java %}
int i = 32;
long l = 32;
System.out.println(i >> 32); //32
System.out.println(l >> 32); //0
{% endcodeblock %}

## 反射机制   

> 反射主要是指程序可以访问、检测和修改它本身的状态或行为的一种能力。  

在java中具体体现为一个类可以反射出一个class对象，通过class对象可以访问该类内部的成员变量和成员函数的详细信息。


## Object类中的方法

|类型|名称|解释|
|---|---|---|
|protected Object |clone()|Creates and returns a copy of this object.|
|boolean|equals(Object obj)|Indicates whether some other object is "equal to" this one.|
|protected void| finalize()|Called by the garbage collector on an object when garbage collection determines that there are no more references to the object.|
|Class<?> 	|getClass()|Returns the runtime class of this Object.|
|int 	|hashCode()|Returns a hash code value for the object.|
|void 	|notify()|Wakes up a single thread that is waiting on this object's monitor.
|void 	|notifyAll()|Wakes up all threads that are waiting on this object's monitor.
|String 	|toString()|Returns a string representation of the object.
|void 	|wait()|Causes the current thread to wait until another thread invokes the notify() method or the notifyAll() method for this object.
|void 	|wait(long timeout)|Causes the current thread to wait until either another thread invokes the notify() method or the notifyAll() method for this object, or a specified amount of time has elapsed.
|void 	|wait(long timeout, int nanos)|Causes the current thread to wait until another thread invokes the notify() method or the notifyAll() method for this object, or some other thread interrupts the current thread, or a certain amount of real time has elapsed.


## 接口中是否可以生命静态方法？

先说说`java.util.Collections`类，该类是一个工具类，它的构造函数是private的，所以你无法进行实例化，也无法被继承。将构造函数设置为private是java中防止class实例化的常用技巧，目的就是希望用户可以将该类作为一个工具使用，或者，通过该类提供的统一接口，如builder或者factory进行创建。

接口中是可以有静态方法的（与老版不同？），当然，该静态函数`必须`是有函数体的。同样，也可以有静态属性。仔细看，还会发现还可以有非静态的属性？其实这是一个默认静态的属性。

实现该接口的类也只需实现抽象方法，静态方法并没有被继承，也就是没办法通过类调用setMe。但是，却可以通过类调用me和sMe两个属性。

```java
//下面都是可以正确编译和执行的
//MeFace.java
public interface MeFace {
	//其实是默认静态的
	int me=1;
	
	static int sMe=2;
	
	public static void setMe() {
		System.out.println(sMe);
		System.out.println(me);
	}
	
	public int getMe();
}

//Main.java
public class Main{

	public static void main(String[] args) {
		MeFace.setMe();
		System.out.println(MeFace.me);
		System.out.println(MeFace.sMe);
	}

}
```

如果MeFace不是一个接口，而是一个abstract class，规则就有所不同。Abstract class它首先是个class，所以me就是一个常规的属性，sMe和setMe都会被继承下来，子类也可以正确使用这些函数和属性。



