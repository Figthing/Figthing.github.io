title: JAVA动态代理机制
author: Figthing
tags:
  - java
  - proxy
categories:
  - java
date: 2018-05-04 17:24:00
---
#### JAVA动态代理机制以及使用场景

##### 什么是代理？
大道理上讲代理是一种软件设计模式，目的地希望能做到代码重用。具体上讲，代理这种设计模式是通过不直接访问被代理对象的方式，而访问被代理对象的方法。这个就好比 商户---->明星经纪人(代理)---->明星这种模式。我们可以不通过直接与明星对话的情况下，而通过明星经纪人(代理)与其产生间接对话。

##### 什么情况下使用代理？
- 设计模式中有一个设计原则是开闭原则，是说对修改关闭对扩展开放，我们在工作中有时会接手很多前人的代码，里面代码逻辑让人摸不着头脑(sometimes the code is really like shit)，这时就很难去下手修改代码，那么这时我们就可以通过代理对类进行增强。

- 我们在使用RPC框架的时候，框架本身并不能提前知道各个业务方要调用哪些接口的哪些方法 。那么这个时候，就可用通过动态代理的方式来建立一个中间人给客户端使用，也方便框架进行搭建逻辑，某种程度上也是客户端代码和框架松耦合的一种表现。

- Spring的AOP机制就是采用动态代理的机制来实现切面编程。

<!--more-->

##### 静态代理和动态代理
我们根据加载被代理类的时机不同，将代理分为静态代理和动态代理。如果我们在代码编译时就确定了被代理的类是哪一个，那么就可以直接使用静态代理；如果不能确定，那么可以使用类的动态加载机制，在代码运行期间加载被代理的类这就是动态代理，比如RPC框架和Spring AOP机制。

##### 静态代理
我们先创建一个接口，遗憾的是java api代理机制求被代理类必须要实现某个接口，对于静态代理方式代理类也要实现和被代理类相同的接口；对于动态代理代理类则不需要显示的实现被代理类所实现的接口。

接口文件：IUser.java
```java
public interface IUser {
	public String getUserName();
}
```

明星：User.java
```java
public class User implements IUser{

	@Override
	public void getUserName() {
		return "我叫Fighting";
	}
}
```

经纪人：UserProxyFactory.java
```java
public class UserProxyFactory implements IUser {
	
	private IUser user;
	
	public UserProxyFactory(IUser user){
		this.user = user;
	}

	@Override
	public void getUserName() {
		// TODO Auto-generated method stub
		System.out.println("代理开始");
		this.user.sayHello();
		System.out.println("代理结束");
	}

}
```

商户：app.java

```java
public class app {
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		//user为被代理的对象，某些情况下 我们不希望修改已有的代码，我们采用代理来间接访问
		User user = new User();
		//创建代理类对象
		UserProxyFactory proxy = new UserProxyFactory(user);
		//调用代理类对象的方法
		System.out.println(proxy.getUserName());
	}
}

```

静态代理看起来是比较简单的，没有什么问题只不过是在代理类中引入了被代理类的对象而已。
那么接下来我们看看动态代理。

##### 动态代理

IUser.java不变，User.java不变，UserProxyFactory.java发生改变

```java
public class UserProxyFactory {

	private Object target;

	public UserProxyFactory(Object target) {
		this.target = target;
	}

	public Object getProxyInstance(){
		return Proxy.newProxyInstance(
				target.getClass().getClassLoader(),
				target.getClass().getInterfaces(),
				new InvocationHandler() {
					@Override
					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
						System.out.println("代理开始");
						Object retVal = method.invoke(target, args);
						System.out.println("代理结束");
						return retVal;
					}
				}
		);
	}
}
```

其中最重要的一点，类似于Spring AOP切面，在`代理开始`,`代理结束`地方加上逻辑，就成了切面方式了！这也是为什么要用代理的一个重要原因之一！你不用修改任何已经编写好的代码，只要使用代理就可以灵活的加入任何东西，将来不喜欢了，不用也不会影响原来的代码。