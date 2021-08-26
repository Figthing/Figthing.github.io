title: 适配器模式之对象适配器
author: Figthing
tags:
  - 设计模式
  - 适配器模式
categories:
  - 设计模式
date: 2018-02-22 09:59:00
---
问题导入：比如有A型螺母和B型螺母，那么用户可以再A型螺母上直接使用按着A型螺母生产的A型螺丝，同样也可以在B型螺母上直接使用按着B型螺母标准生产的B型螺丝。但是由于A型螺母和B型螺母的标准不一样，用户在A型螺母上不能直接使用B型的螺丝，反之也一样。该如何达到这个目的呢？

使用适配器就可以解决这个问题：生产一种“A型螺母适配器”，这种A型螺母适配器的前端符合A型螺母标准要求，可以拧在A型螺母上，后端又焊接了一个B型螺母。这样用户就可以借助A型螺母适配器在A型螺母上使用B型的螺丝了。

适配器模式又称为包装器，是用来将一个类的接口转换成客户希望的另外一个接口。这可以使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。适配器模式的关键是建立一个适配器，这个适配器实现了目标接口并且包含了被适配者的引用。

**适配器模式的三种角色：**
- 目标：目标是一个接口，该接口是客户想要使用的接口。
- 被适配者：被适配者是一个已经存在的接口或抽象类，这个接口接口或者抽象类需要适配。
- 适配器：适配器是一个类，该类实现了目标接口并且包含有被适配者的引用，即适配器的职责是对适配者接口或抽象类与目标接口进行适配。

以下通过一个简单的问题来描述适配器模式中所涉及的各个角色。

<!--more-->

#### 实例

用户已经有一个两厢的插座，但是最近用户又有了一个新的三厢插座。用户现有一台洗衣机和一台电视机，洗衣机是三厢插头，而电视机是两厢插头。现在用户想用心的三厢插座来使用洗衣机和电视机，即用心的三厢插座为洗衣机和电视机接通电流。
  
针对以上问题，使用适配器模式设计若干个类。

1. **目标**

 本问题是使用三厢插座来为电视机和洗衣机接通电流，所以目标是三厢插座。把三厢插座设置为一个接口：
  
 ```java
 package com.adatpe;

 //适配目标：三相插座
 public interface ThreeElectricOutlet {
 	void connectElectricCurrent();
 }
 ```

2. **被适配者**

 对于本问题，用户是想要用三厢插座为两厢插头的电视机接通电流，所以被适配者应该是两厢插座，也设置为一个接口：

 ```java
 package com.adatpe;

 //被适配者：两相插座
 public interface TwoElectricOutlet {
     void connectElectricCurrent();
 }
 ```

3. **适配器**

 该适配器实现了目标接口三厢插座ThreeElectricOutlet，同时又包含了两厢插座TwoElectricOutlet的引用：
  
 ```java
 package com.adatpe;

 //适配器：实现目标接口
 public class ThreeElectricAdapter implements ThreeElectricOutlet {
     //适配器包含被适配者的引用
     private TwoElectricOutlet outlet;
     public ThreeElectricAdapter(TwoElectricOutlet outlet) {
         this.outlet = outlet;
     }
     public void connectElectricCurrent() {
         outlet.connectElectricCurrent();
     }

 }
 ```

下列应用程序中，Application.java使用了适配器模式中所涉及的类，应用程序负责用Wash类创建一个对象来模拟一台洗衣机，使用TV类创建一个对象来模拟一台电视机

使用ThreeElectricOutlet接口变量调用Wash对象的connectElectricCurrent()方法，并借助适配器调用TV对象的connectElectricCurrent()方法，即用三厢插座分别为洗衣机和电视机接通电流。

```java
package com.adatpe; 
 
public class Application {
    public static void main(String[] args) {
        ThreeElectricOutlet outlet; //目标接口（三相插座）
        Wash wash = new Wash();     //洗衣机
        outlet = wash;              //洗衣机插在三相插座上
        System.out.println("使用三相插座接通电流");
        outlet.connectElectricCurrent();    //接通电流开始洗衣服
        TV tv = new TV();           //电视机
        ThreeElectricAdapter adapter = new ThreeElectricAdapter(tv); //把电视插在适配器上面
        outlet = adapter;           //再把适配器插在三厢插座上
        System.out.println("使用三厢插座接通电流");
        outlet.connectElectricCurrent();  //接通电流，开始播放电视节目
    }
}

//洗衣机使用三相插座
class Wash implements ThreeElectricOutlet{
    private String name;
    public Wash() {
        name = "黄河洗衣机";
    }
    public Wash(String name){
        this.name = name;
    }
    public void connectElectricCurrent() {
        turnOn();
    }
    public void turnOn(){
        System.out.println(name+"开始洗衣服了");
    }
}

//电视机使用两厢插座
class TV implements TwoElectricOutlet{
    private String name;
    public TV() {
        name = "长江电视机";
    }
    public TV(String name){
        this.name = name;
    }
    public void connectElectricCurrent() {
        turnOn();
    }
    public void turnOn(){
        System.out.println(name+"开始播放电视节目");
    }
}
```
 
**运行结果为：**

使用三相插座接通电流  
黄河洗衣机开始洗衣服了  
使用三厢插座接通电流  
长江电视机开始播放电视节目
  


#### 双向适配器

在适配器模式中，如果Adapter角色同时实现目标接口和被适配者接口，并包含目标接口和被适配接口的引用，那么该适配器就是一个双向适配器。使用双向适配器，用户既可以用新的接口又可以用已有的接口。在以上例子中，如果用户希望能有三厢插座来接通洗衣机和电视机的电流，有可以用两厢插座来接通洗衣机和电视机的电流，那么就必须使用一个双向适配器。具体代码如下：
  
  
```java
package com.adatpe;

public class ThreeAndTwoElectricAdapter implements ThreeElectricOutlet,
        TwoElectricOutlet {
    private ThreeElectricOutlet threeElectricOutlet;
    private TwoElectricOutlet twoElectricOutlet;
    public ThreeAndTwoElectricAdapter(ThreeElectricOutlet threeOutlet,TwoElectricOutlet twoOutlet) {
        threeElectricOutlet = threeOutlet;
        twoElectricOutlet = twoOutlet;
    }
    public ThreeAndTwoElectricAdapter(TwoElectricOutlet twoOutlet,ThreeElectricOutlet threeOutlet){
        threeElectricOutlet = threeOutlet;
        twoElectricOutlet = twoOutlet;
    }
    public void connectElectricCurrent() {
        if(this instanceof ThreeElectricOutlet){
            twoElectricOutlet.connectElectricCurrent();//twoElectricOutlet是被适配的接口
        }
        if(this instanceof TwoElectricOutlet){
            threeElectricOutlet.connectElectricCurrent(); //threeElectricOutlet是被适配的接口
        }
    }
    public static void main(String[] args) {
        ThreeElectricOutlet threeOutlet;
        TwoElectricOutlet twOutlet;
        Wash wash = new Wash();
        TV tv = new TV();
        ThreeAndTwoElectricAdapter adapter = new ThreeAndTwoElectricAdapter(wash,tv);
        threeOutlet = adapter;
        System.out.println("使用三厢插座接通电源");
        threeOutlet.connectElectricCurrent();
        twOutlet = adapter;
        System.out.println("使用两厢插座接通电源");
        twOutlet.connectElectricCurrent();
    }

}
```

**运行结果为：**

使用三厢插座接通电源  
长江电视机开始播放电视节目  
黄河洗衣机开始洗衣服了  
使用两厢插座接通电源  
长江电视机开始播放电视节目  
黄河洗衣机开始洗衣服了  

这样就实现了即可以用三厢插座又可以用两厢插座来为电视机和洗衣机接通电流了。


#### 优点
- 目标和被适配者是完全解耦的关系。
- 适配器模式满足“开--闭原则”，当添加一个实现了Adapter接口的新类时，不必修改Adapter，Adapter就能对这个新类的实例进行适配。