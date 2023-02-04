---
title: Spring AOP系列（一）— 代理模式
tags:
  - aop
categories:
  - 总结
date: 2021-07-15 00:00:00
---

> AOP（Aspect Oriented Programming）并没有创造或使用新的技术，其底层就是基于代理模式实现。因此我们先来学习一下代理模式。
# 一、基本概念
## 1.1 定义
代理模式，为对象提供一种代理，以控制对这个对象的访问。
## 1.2 角色
代理模式也称为委托模式，一般有以下三个角色
![Alt 代理模式的通用类图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1bae9c1441b49038c8af093aec5d878~tplv-k3u1fbpfcp-zoom-1.image)
- **抽象主题角色**：抽象主题类可以是抽象类也可以是接口，是一个最普通的业务类型定义，无特殊要求。
- **具体主题角色**：也被称为被委托角色、被代理角色，是具体业务逻辑的实际执行者。
- **代理对象角色**：也被称为委托类，代理类。负责控制具体主题角色的访问和应用，并且可以在具体主题角色的业务逻辑执行前后进行预处理和后处理的逻辑。
  从“具体主题角色”的定义我们可以明确一点：代理模式中的被代理类和代理类的关系，并非是说代理类帮助被代理类完成了所有的工作。反而是最核心的工作仍然是由被代理类自己来完成。代理类在其中起的作用就是控制被代理类的访问、应用，以及在被代理类执行核心业务逻辑的前后进行其它的处理。
  下面的实践将会帮助我们更好地理解。

<!-- more -->

# 二、简单代理模式的基本实现
这里我们先按照基本概念中内容，完成一个简单代理模式的基本实现。设想这样一个场景：明星与经纪人。经纪人帮明星与外界洽谈合作，签订合同，计算佣金；明星完成演戏、唱歌等工作。
首先定义一个Actor.java
```java
package com.leng.proxy;

/**
 * @Classname Actor
 * @Date 2020/9/12 23:30
 * @Autor lengxuezhang
 */
public interface Actor {
    /**
     * 演戏
     */
    public void act();

    /**
     * 唱歌
     */
    public void sing();
}
```
艺人一般需要会唱歌和演戏。
接着分别实现明星类，实现Actor接口，并传入姓名。
```java
package com.leng.proxy;

/**
 * @Classname Star
 * @Date 2020/9/12 23:32
 * @Autor lengxuezhang
 */
public class Star implements Actor {

    private String name;

    public Star(String name) {
        this.name = name;
    }

    @Override
    public void act() {
        System.out.println(name + "在演戏");
    }

    @Override
    public void sing() {
        System.out.println(name + "在唱歌");
    }
}
```
最后实现经纪人
```java
package com.leng.proxy;

/**
 * @Classname Agent
 * @Date 2020/9/12 23:36
 * @Autor lengxuezhang
 */
public class Agent implements Actor {

    private Actor actor = null;

    // 要想实现经纪人代理明星，需要将被代理类对象传递给代理类
    public Agent(Actor actor) {
        this.actor = actor;
    }
    
    @Override
    public void act() {
        this.before();
        // 实际调用的是明星的act(),经纪人接活，但最后去演戏的肯定是明星自己
        actor.act();
        this.after();
    }

    @Override
    public void sing() {
        this.before();
        actor.sing();
        this.after();
    }

    // 预处理方法
    private void before() {
        System.out.println("谈好价钱，签好合同");
    }

    // 后处理方法
    private void after() {
        System.out.println("清算演出费，和明星分钱");
    }
}
```
客户端调用Client.java：
```java
package com.leng;

import com.leng.proxy.Agent;
import com.leng.proxy.Star;

/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
        Star star = new Star("刘德华");
        Agent agent = new Agent(star);
        agent.act();
        agent.sing();
    }
}
```
运行结果：
```shell
谈好价钱，签好合同
刘德华在演戏
清算演出费，和明星分钱
谈好价钱，签好合同
刘德华在唱歌
清算演出费，和明星分钱
```

**小结**：
- 代理模式实现的关键是，代理类持有被代理类对象，这样才能执行被代理类本身想要执行的业务逻辑。
- 代理类可以定义自己的预处理和后处理方法，使之在被代理类的业务逻辑前后执行。
  以上被称为基于代理模式概念的一个例子，实际上代理模式在不同场景或要求下会有一些新的特性。如接下来要介绍的普通代理和强制代理。
# 三、代理模式的扩展
## 3.1 普通代理
明星的工作一般比较忙，没有时间和外界有太多沟通和接触。导演想要找明星演戏，一般只能先去联系他的公司经纪人，是联系不到明星本人。“导演只能联系到经纪人，而不能联系明星”，这样的模式就被称为“普通代理”。换句话说：**普通代理模式下，客户端只能访问代理类，不能访问被代理类。**
上面的Client.java中，直接new了一个明星对象，这就算是直接访问了被代理类，不能称为普通代理。那如何才能实现普通代理呢。关键是让被代理类的创建只能由代理类来完成。修改代码如下：
只需要修改构造函数
```java
package com.leng.proxy;

/**
 * @Classname Star
 * @Date 2020/9/12 23:32
 * @Autor lengxuezhang
 */
public class Star implements Actor {

    private String name;

    public Star(Agent agent, String name) throws Exception {
        // 必须是经纪人才能创建明星
        if(agent == null) {
            throw new Exception("非经纪人，不能创造明星");
        } else {
            this.name = name;
        }
    }

    @Override
    public void act() {
        System.out.println(name + "在演戏");
    }

    @Override
    public void sing() {
        System.out.println(name + "在唱歌");
    }
}
```
```java
package com.leng.proxy;

/**
 * @Classname Agent
 * @Date 2020/9/12 23:36
 * @Autor lengxuezhang
 */
public class Agent implements Actor {

    private Actor actor = null;

    // 告诉agent，我需要联系哪一个"明星"
    public Agent(String name) {
        try {
            this.actor = new Star(this, name);
        } catch (Exception e) {
            System.out.println("创建agent异常");
        }
    }

    @Override
    public void act() {
        this.before();
        actor.act();
        this.after();
    }

    @Override
    public void sing() {
        this.before();
        actor.sing();
        this.after();
    }

    // 预处理方法
    private void before() {
        System.out.println("谈好价钱，签好合同");
    }

    // 后处理方法
    private void after() {
        System.out.println("清算演出费，和明星分钱");
    }
}
```
客户端类在使用时，也需要做修改，此时客户端可以不直接访问Star
```java
package com.leng;
import com.leng.proxy.Agent;
/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
        Agent agent = new Agent("刘德华");
        agent.act();
        agent.sing();
    }
}
```
运行结果：
```shell
谈好价钱，签好合同
刘德华在演戏
清算演出费，和明星分钱
谈好价钱，签好合同
刘德华在唱歌
清算演出费，和明星分钱
```
可以看到运行结果并没有发生改变，但确实已经实现了只能以访问代理类方式访问非代理类。假如客户端直接访问代理类的话，是会抛出异常的。
普通代理模式的优点：
- 调用者不关心被代理类的实现，被代理类的修改不会影响高层模块的调用。
- 被代理类的访问权限收口在代理类。
## 3.2 强制代理
先来看一个场景：
> 王晶想找华仔拍戏，两人老相识了，于是王晶直接打电话问华仔：“华仔啊，有咩时间？搵你拍片喔"”。
> 华仔回复：“你先找我的经纪人谈一下吧，看一下我的工作安排”。
> 然后华仔给王晶自己经纪人的联系方式，王晶和经纪人谈好之后，华仔就准备去拍王晶的电影了。
这个场景里有个点，首先王晶实际上是不能直接找华仔拍戏，即使有联系方式，华仔也不一定给你拍；其次是王晶必须与华仔指定的经纪人沟通具体的拍戏工作。
强制代理的要求是：
- 不能直接通过代理类访问被代理类。
- 不能直接访问被代理类。
- 只能听过被代理类指定的代理类访问被代理类。
  接下来逐个讲解如何修改之前的代码。首先抽象主题角色中需要增加一个获取自己代理类的方法，其它不变：
```java
public interface Actor {
	...
    public Actor getProxy();
}
```
明星类实现该方法，一方面要找到自己的代理，另一方面其它的唱歌、演戏方法也需要检查是否是自己指定的代理类访问的。
```java
package com.leng.proxy;

/**
 * @Classname Star
 * @Date 2020/9/12 23:32
 * @Autor lengxuezhang
 */
public class Star implements Actor {

    private String name;

    private Agent agent = null;

    public Star(String name) {
        this.name = name;
    }

    @Override
    public void act() {
        if (isProxy()) {
            System.out.println(name + "在演戏");
        } else {
            System.out.println("请使用指定的代理访问");
        }
    }

    @Override
    public void sing() {
        if (isProxy()) {
            System.out.println(name + "在唱歌");
        }else {
            System.out.println("请使用指定的代理访问");
        }
    }

    @Override
    public Actor getProxy() {
        this.agent = new Agent(this);
        return this.agent;
    }

    /**
     * 判断是否是指定的代理
     * @return
     */
    private boolean isProxy() {
        if(agent == null) {
            return false;
        }
        return true;
    }
}
```
经纪人类
```java
package com.leng.proxy;

/**
 * @Classname Agent
 * @Date 2020/9/12 23:36
 * @Autor lengxuezhang
 */
public class Agent implements Actor {

    private Actor actor = null;

    // 告诉agent，我需要联系哪一个"明星"
    public Agent(Actor actor) {
        this.actor = actor;
    }

    @Override
    public void act() {
        this.before();
        actor.act();
        this.after();
    }

    @Override
    public void sing() {
        this.before();
        actor.sing();
        this.after();
    }

    @Override
    public Actor getProxy() {
        //经纪人没有自己的经纪人，所以就返回自己
        return this;
    }

    // 预处理方法
    private void before() {
        System.out.println("谈好价钱，签好合同");
    }

    // 后处理方法
    private void after() {
        System.out.println("清算演出费，和明星分钱");
    }
}
```
现在来测试一下，首先看看如果直接访问被代理类，是否可行
```java
public class Client {
    public static void main(String[] args) {

        Star star = new Star("刘德华");
        star.act();
        star.sing();

    }
}
```
运行结果：
```java
请使用指定的代理访问
请使用指定的代理访问
```
显然访问失败了。接下来尝试一下如果通过代理类访问呢
```java
package com.leng;

import com.leng.proxy.Actor;
import com.leng.proxy.Agent;
import com.leng.proxy.Star;

/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
        Actor star = new Star("刘德华");
        Actor agent = new Agent(star);
        agent.act();
        agent.sing();
    }
}
```
```shell
谈好价钱，签好合同
请使用指定的代理访问
清算演出费，和明星分钱
谈好价钱，签好合同
请使用指定的代理访问
清算演出费，和明星分钱
```

可以看到中间业务逻辑部分还是错误的。ok，那我就按照强制代理的规定来：
```java
package com.leng;

import com.leng.proxy.Actor;
import com.leng.proxy.Star;

/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
        // 先定义一个明星
        Actor star = new Star("刘德华");
        // 由这个明星指定经纪人
        Actor agent = star.getProxy();
        agent.act();
        agent.sing();
    }
}
```
```shell
谈好价钱，签好合同
刘德华在演戏
清算演出费，和明星分钱
谈好价钱，签好合同
刘德华在唱歌
清算演出费，和明星分钱
```
运行结果显示成功。

**小结**：强制代理的核心就是通过被代理类指定的代理类，去访问被代理类的方法。
# 四、总结
本篇文章首先介绍了代理模式的基本概念和基本代码实现。然后进行了一些扩展，介绍了普通代理和强制代理的概念和实现。下一篇将会详细介绍动态代理的知识。

## 五、参考文献
《设计模式之禅》

**最后**
-  文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处
