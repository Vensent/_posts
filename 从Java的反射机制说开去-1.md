title: "从Java的反射机制说开去<1>"
date: 2014-10-10 11:09:28
tags: [java, 反射机制]
categories: Java
---
最近在写Android的时候用到了一些关于Java反射机制的内容。Java的反射机制也是在面试过程中常考的，这里就好好地将它们回顾一下。

## 什么是Java的反射机制
Java比起其他的语言，最大的特点就是“面向对象”。而怎么合理地理解“面向对象”的概念则是需要大量的编程练习的积累。            
通俗地说，反射机制就是可以把一个类，类的成员（函数、变量），当成一个对象来操作，希望读者能理解。也就是说，类、类的成员，在运行需要的时候可以动态地去操作它们。         
下面就用一个例子来具体地说明。

## 反射机制的Demo
在这里只就类的函数的反射来说明一下。          
先是一个自己写的Student类，无须赘述：        
```java
public class Student {

	private String name;
	private int age;
	
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public int getAge() {
		return age;
	}
	public void setAge(int age) {
		this.age = age;
	}
	
	public Student()
	{
	}
	
	public Student(String name, int age)
	{
		this.name = name;
		this.age = age;
	}
	
	public void print()
	{
		System.out.println("This Student is: " + name + ",");
		System.out.println("This Student is " + age + " years old.");
	}
}
```
接下来就是关于反射的使用：             
```java
public class Test {

	/**
	 * @param args
	 */
	public static void main(String[] args) {

		try {
		    // 第一步：new一个Student，这一步不可少
			Student demoStudent = new Student();
			// 第二步：通过Class类来实现的反射机制，所以需要获得Student的Class
			Class<Student> studentClass = Student.class;
			// 或者通过包名forName来获得：
			// Class<?> studentClass = Class.forName("com.test.inflect.Student");
			// 第三步：构造需要调用的方法的参数
			Class nameArea = String.class;
			Class ageArea = int.class;
			// 第四步：使用getDeclaredMethod，这步是反射机制的精髓。该方法的参数分别是要获得的方法名，以及要获得的方法的参数类型，前者是String类型，后者是Class<?>类型
			Method setNameMethod = studentClass.getDeclaredMethod("setName", nameArea);
			Method setAgeMethod = studentClass.getDeclaredMethod("setAge", ageArea);
			Method printMethod = studentClass.getDeclaredMethod("print", (Class[]) null);
			// setAccessible(true)并不一定需要，因为默认的就是true.
			printMethod.setAccessible(true);
			setAgeMethod.setAccessible(true);
			setNameMethod.setAccessible(true);
			
			// 第五步：使用invoke调用方法。第一个参数是调用方法的发出者，后面的参数则依次是调用的方法的参数
			setNameMethod.invoke(demoStudent, "Vensent");
			setAgeMethod.invoke(demoStudent, 23);
			printMethod.invoke(demoStudent, (Object[]) null);
		} catch (Exception e) {
			e.printStackTrace();
		} 		
	}

}
```

## 疑惑
细心的读者可能会发现在上述的例子中，如果我们直接使用`demoStudent.setName(String)`、`demoStudent.setAge(int)`、`demoStudent.print()`不就完事了么，为什么还要去用什么反射机制？劳神费力不说，getDeclaredMethod、invoke还这么复杂，稍不留神就可能用错。          
笔者一开始也是这样想的，而且这样的想法也持续了很长的时间。到底反射机制是在什么样的情况下可以使用的？               
而在最近写Android的代码的时候却多次碰到了这种情况。通过下面一个例子来具体说明。        
请继续阅读：[从Java的反射机制说开去<2>](http://vensent.github.io/2014/10/10/%E4%BB%8EJava%E7%9A%84%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6%E8%AF%B4%E5%BC%80%E5%8E%BB-2/)。

