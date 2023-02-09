---
title: Spring AOP系列（五）— 反射
tags:
  - aop
categories:
  - 总结
date: 2021-08-15 00:00:00
---

# 前言
前面我们进行了代理模式、静态代理、动态代理的学习。而动态代理就是利用Java的反射技术(Java Reflection)，在运行时创建一个实现某些给定接口的新类（也称“动态代理类”）及其实例（对象）。所以接下来我们有必要学习一下Java中的反射。
# 一、基础知识
## 1.1 反射是什么？
在讲反射之前，先提一个问题：假如现在有一个类`User`，我想创建一个`User`对象并且获取到其`name`属性，我该怎么做呢？
`User.java`
```java
package com.reflect;

/**
 * @author: create by lengzefu
 * @description: com.reflect
 * @date:2020-09-29
 */
public class User {
    private String name = "小明";
    
    Integer age = 18;
    
    public User(){
        
    }
    
    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```
方式很简单：
```java
import com.reflect.User;

public class Main {

    public static void main(String[] args) {
        User user = new User();
        System.out.println(user.getName());
    }
}
```
这种方式是我们日常在写代码时经常用到的一种方式。这是因为我们在使用某个对象时，总是**预先**知道自己需要使用到哪个类，因此可以使用直接 new 的方式获取类的对象，进而调用类的方法。
那假设我们预先并不知道自己要使用的类是什么呢？这种场景其实很常见，比如动态代理，我们事先并不知道代理类是什么，代理类是在运行时才生成的。这种情况我们就要用到今天的主角：**反射**
### 1.1.1 反射的定义
JAVA反射机制是指在<font color=red>**运行状态**</font>中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。
注意：这里特别强调的是<font color=red>**运行状态**</font>。
## 1.2 反射能做什么？
定义已经给了我们答案。反射可以使得程序在运行时可以获取到任意类的任意属性和方法并调用。
## 1.3 反射为什么能做到？
这里我们需要讲一个“类对象”的概念。java中“面向对象”的理念贯彻的比较彻底，它强调“万事万物皆对象”。那么“类”是不是也可以认为是一个对象呢？java中有一种特殊的类：`Class`，它的对象是“类”，比如“String”类，“Thread”类都是它的对象。
> java.lang.Class是访问类型元数据的接口，也是实现反射的关键数据。通过Class提供的接口，可以访问一个类型的方法、字段等信息。

以上已经解答了“反射为什么能做到可以使得程序在运行时可以获取到任意类的任意属性和方法并调用”的问题，它是依赖.class字节码文件做到的。那么首先我们需要解决的问题是如何获取字节码文件对象（Class对象）。
### 1.3.1 获取Class对象
对于一个类，如上文的`User`，我想获取`User`的相关信息（由于`Users`属于`Class`类的对象，所以一般称该行为为“获取类对象”），该怎么做呢？
有以下三种方式
```java
import com.reflect.User;

public class Main {

    public static void main(String[] args) throws ClassNotFoundException {
        // 1.已实例化的对象可调用 getClass() 方法来获取,通常应用在：传过来一个 Object类型的对象，不知道具体是什么类，用这种方法
        User user = new User();
        Class clz1 = user.getClass();

        // 2.类名.class 的方式得到,该方法最为安全可靠，程序性能更高，这说明任何一个类都有一个隐含的静态成员变量 class
        Class clz2 = User.class;

        // 通过类的全路径名获取Class对象，用的最多，如果根据类路径找不到这个类那么就会抛出 ClassNotFoundException异常。
        Class clz3 = Class.forName("com.reflect.User");

        // 一个类在 JVM 中只会有一个 Class 实例,即我们对上面获取的 clz1,clz2,clz3进行 equals 比较，发现都是true。
        System.out.println(clz1.equals(clz2));
        System.out.println(clz2.equals(clz3));
    }
}
```
### 1.3.2 Class API
```java
获取公共构造器 getConstructors()
获取所有构造器 getDeclaredConstructors()
获取该类对象 newInstance()
获取类名包含包路径 getName()
获取类名不包含包路径 getSimpleName()
获取类公共类型的所有属性 getFields()
获取类的所有属性　getDeclaredFields()
获取类公共类型的指定属性 getField(String name)
获取类全部类型的指定属性 getDeclaredField(String name)
获取类公共类型的方法 getMethods()
获取类的所有方法 getDeclaredMethods()
获得类的特定公共类型方法: getMethod(String name, Class[] parameterTypes)
获取内部类 getDeclaredClasses()
获取外部类 getDeclaringClass()
获取修饰符 getModifiers()
获取所在包 getPackage()
获取所实现的接口 getInterfaces()
```

具体如何使用不再赘述

# 二、反射原理解析
## 2.1 反射与类加载的关系
java类的执行需要经历以下过程，

**编译**：java文件编译后生成.class字节码文件
**加载**：类加载器负责根据一个类的全限定名来读取此类的二进制字节流到 JVM 内部，并存储在运行时内存区的方法区，然后将其转换为一个与目标类型对应的 `java.lang.Class` 对象实例
**链接**：
- 验证：格式（class文件规范） 语义（final类是否有子类） 操作
- 准备：静态变量赋初值和内存空间，final修饰的内存空间直接赋原值，此处不是用户指定的初值。
- 解析：符号引用转化为直接引用，分配地址

**初始化**：有父类先初始化父类，然后初始化自己；将static修饰代码执行一遍，如果是静态变量，则用用户指定值覆盖原有初值；如果是代码块，则执行一遍操作。

Java的反射就是利用上面<font color=blue>第二步加载到jvm中的.class文件来进行操作的。.</font>第二步加载到jvm中的.class文件来进行操作的。.class文件中包含java类的所有信息，当你不知道某个类具体信息时，可以使用反射获取class，然后进行各种操作。

首先我们来看看如何使用反射来实现方法的调用的：
```java
public class Main {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        // 1.获取类对象
        Class clz3 = Class.forName("com.reflect.User");
        // 2.获取类的构造函数
        Constructor constructor = clz3.getConstructor(String.class, Integer.class);
        // 3.创建一个对象
        User user = (User)constructor.newInstance("璐璐", 17);
        // 4.获取方法getName
        Method method = clz3.getMethod("getName");
        // 5.调用该方法
        String name = (String) method.invoke(user);

        System.out.println(name);
    }
}
```
接下来主要解析4，5两个过程：获取Method对象和Methode.invoke
## 2.2 获取 Method 对象
### 2.2.1 获取 Method 的API介绍
Class API中关于获取Method对象的方法有如下几个：
`getMethod/getMethods` 和 `getDeclaredMethod/getDeclaredMethod`
**后缀有无“s”的区别**
有“s”表示获取的所有的，无“s”表示获取的是特定的（由方法参数指定）。
**`getMethod`和`getDeclaredMethod`的区别**
Method对应的是`Member.PUBLIC`，DeclaredMethod对应的是`Member.DECLARED`
两者定义如下：
```java
public
interface Member {

    /**
     * Identifies the set of all public members of a class or interface,
     * including inherited members.
     * 标识类或接口的所有公共成员的集合，包括父类的公共成员。
     */
    public static final int PUBLIC = 0;

    /**
     * Identifies the set of declared members of a class or interface.
     * Inherited members are not included.
     * 标识类或接口所有声明的成员的集合（public、protected,private），但是不包括父类成员
     */
    public static final int DECLARED = 1;
}
```
其实不管是`getMethod`还是`getDeclaredMethod`，底层都调用了同一个方法：`privateGetDeclaredMethods`，因此我们只分析其中一个方法即可。

### 2.2.2 getMethod 方法源码分析
#### seq1
```java
// 4.获取方法getName
Method method = clz3.getMethod("getName");
```
客户端调用`Class.getMethod()`方法。
#### seq2
```java
	// 参数“name”为方法名，参数“parameterTypes”为方法的参数，由于参数可能有多个且类型不同，所以这里使用到了泛型及可变参数的设定
    public Method getMethod(String name, Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        // 权限安全检查，无权限则抛出 SecurityException
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        Method method = getMethod0(name, parameterTypes, true);
        // 获取到的method为空，抛出 NoSuchMethodException 异常
        if (method == null) {
            throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
        }
        return method;
    }
```
该方法的核心方法是 `getMethod0`。
#### seq3
```java
private Method getMethod0(String name, Class<?>[] parameterTypes, boolean includeStaticMethods) {
		// 保存接口中的方法，最多只有1个，但MethodArray初始化大小至少为2
        MethodArray interfaceCandidates = new MethodArray(2);
        // 递归获取方法，之所以递归，正是因为getMethod是需要获取父类中的方法，与前面关于getMethod的介绍对应
        Method res =  privateGetMethodRecursive(name, parameterTypes, includeStaticMethods, interfaceCandidates);
        // 获取到本类或父类中的方法，直接返回结果
        if (res != null)
            return res;

        // Not found on class or superclass directly
        // 在本类或父类中没有找到对应的方法，则尝试去接口中的方法寻找
        // removeLessSpecifics：移除更不具体的方法，保留具有更具体实现的方法
        interfaceCandidates.removeLessSpecifics();
        return interfaceCandidates.getFirst(); // may be null
    }
```
接下来分析`privateGetMethodRecursive`
#### seq4
```java
 private Method privateGetMethodRecursive(String name,
            Class<?>[] parameterTypes,
            boolean includeStaticMethods,
            MethodArray allInterfaceCandidates) {
        // Must _not_ return root methods
        Method res;
        // Search declared public methods 搜索本来中声明的公共方法
        if ((res = searchMethods(privateGetDeclaredMethods(true),
                                 name,
                                 parameterTypes)) != null) {
            if (includeStaticMethods || !Modifier.isStatic(res.getModifiers()))
            // res 不为空，返回
                return res;
        }
        // Search superclass's methods res为空，继续向父类搜索
        if (!isInterface()) { // 接口必然无父类
            Class<? super T> c = getSuperclass();
            if (c != null) {
	            // 递归调用getMethod0，获取父类的方法实现
                if ((res = c.getMethod0(name, parameterTypes, true)) != null) {
                    return res;
                }
            }
        }
        // Search superinterfaces' methods res仍然为空，继续向接口搜索
        Class<?>[] interfaces = getInterfaces();
        for (Class<?> c : interfaces)
		    // 递归调用getMethod0，获取接口的方法实现
            if ((res = c.getMethod0(name, parameterTypes, false)) != null)
                allInterfaceCandidates.add(res);
        // Not found
        return null;
    }
```
该方法原英文注释翻译之后的大概意思为：
> 注意：该例程（作用类似于函数，含义比函数更广）使用的搜索算法的目的是等效于`privateGetPublicMethods`方法的施加顺序。它仅获取每个类的已声明公共方法，但是，以减少在要查询的类中声明了所请求方法，常见情况下必须创建的Method对象的数量。 由于使用默认方法，除非在超类上找到方法，否则需要考虑在任何超级接口中声明的方法。 收集所有InterfaceCandidates的超级接口中声明的所有候选对象，如果未在超类上找到匹配项，则选择最具体的候选者。

原文的英语注释中各种从句真的很多，自己翻译完感觉还是有点问题。简单来说，我觉得比较重要的一点应该是还是能理解到：
<font color=green>获取该类已声明的方法，如果使用的是默认方法，则从父类中寻找该方法。否则去接口中寻找实现最具体的候选方法</font>

接下来分析`searchMethods`
#### seq5
```java
    private static Method searchMethods(Method[] methods,
                                        String name,
                                        Class<?>[] parameterTypes)
    {
        Method res = null;
        String internedName = name.intern();
        for (int i = 0; i < methods.length; i++) {
            Method m = methods[i];
            if (m.getName() == internedName // 比较方法名字
	            // 比较方法参数
                && arrayContentsEq(parameterTypes, m.getParameterTypes())
                // 比较方法返回值
                && (res == null
                    || res.getReturnType().isAssignableFrom(m.getReturnType())))
                res = m;
        }

        return (res == null ? res : getReflectionFactory().copyMethod(res));
    }
```
`searchMethods`的实现逻辑比较简单，详细如注释。这里关键是方法参数`Method[] methods`是怎么得到的，我们回到`searchMethods`的方法调用处：
```java
searchMethods(privateGetDeclaredMethods(true),
                                 name,
                                 parameterTypes)) != null)
```
`methods`通过方法`privateGetDeclaredMethods(true)`得到
#### seq6
```java
private Method[] privateGetDeclaredMethods(boolean publicOnly) {
        checkInitted();
        Method[] res;
        // 1.ReflectionData 存储反射数据的缓存结构
        ReflectionData<T> rd = reflectionData();
        if (rd != null) {
	        // 2.先从缓存中获取methods
            res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
            if (res != null) return res;
        }
        // No cached value available; request value from VM
        // 3.没有缓存，通过 JVM 获取
        res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
        if (rd != null) {
            if (publicOnly) {
                rd.declaredPublicMethods = res;
            } else {
                rd.declaredMethods = res;
            }
        }
        return res;
    }
```
先看看`ReflectionData<T>`
```java
 // reflection data that might get invalidated when JVM TI RedefineClasses() is called
    private static class ReflectionData<T> {
        volatile Field[] declaredFields;
        volatile Field[] publicFields;
        volatile Method[] declaredMethods;
        volatile Method[] publicMethods;
        volatile Constructor<T>[] declaredConstructors;
        volatile Constructor<T>[] publicConstructors;
        // Intermediate results for getFields and getMethods
        volatile Field[] declaredPublicFields;
        volatile Method[] declaredPublicMethods;
        volatile Class<?>[] interfaces;

        // Value of classRedefinedCount when we created this ReflectionData instance
        final int redefinedCount;

        ReflectionData(int redefinedCount) {
            this.redefinedCount = redefinedCount;
        }
    }
```
`ReflectionData<T>`是类`Class`的静态内部类，`<T>`表示泛型，为具体的类对象。该缓存数据结构中存储了类的所有信息。`redefinedCount`是类的重定义次数，可以理解为缓存的版本号。
注意最上面的一行注释：reflection data that might get invalidated when JVM TI RedefineClasses() is called。意思是 当 JVM TI（工具接口）`RedefineClasses()`被调用时，缓存数据可能会失效。

通过以上分析，我们知道，每一个类对象理论上都会有（被垃圾回收或从来没被加载过就没没有）一个`ReflectionData<T>`的缓存，那么如何获取它呢？

这就要用到 `reflectionData`
```java
    // Lazily create and cache ReflectionData
    private ReflectionData<T> reflectionData() {
	    // 获取当前 reflectionData 缓存
        SoftReference<ReflectionData<T>> reflectionData = this.reflectionData;
        // 当前缓存版本号
        int classRedefinedCount = this.classRedefinedCount;
        ReflectionData<T> rd;
        // 可以使用缓存&&缓存不为空&&缓存中版本号与类中记录的版本号一致则直接返回缓存
        if (useCaches &&
            reflectionData != null &&
            (rd = reflectionData.get()) != null &&
            rd.redefinedCount == classRedefinedCount) {
            return rd;
        }
        // else no SoftReference or cleared SoftReference or stale ReflectionData
        // -> create and replace new instance
        // 创建新的缓存数据
        return newReflectionData(reflectionData, classRedefinedCount);
    }
```
看看`newReflectionData`的实现
```java
    private ReflectionData<T> newReflectionData(SoftReference<ReflectionData<T>> oldReflectionData,
                                                int classRedefinedCount) {
        // 不使用缓存则直接返回null
        if (!useCaches) return null;

		// 使用while+CAS方式更新数据，创建一个新的ReflectionData，如果更新成功直接返回
        while (true) {
            ReflectionData<T> rd = new ReflectionData<>(classRedefinedCount);
            // try to CAS it...
            if (Atomic.casReflectionData(this, oldReflectionData, new SoftReference<>(rd))) {
                return rd;
            }
            // else retry
            // 获取到旧的reflectionData和classRedefinedCount的值，如果旧的值不为null, 并且缓存未失效，说明其他线程更新成功了，直接返回 
            oldReflectionData = this.reflectionData;
            classRedefinedCount = this.classRedefinedCount;
            if (oldReflectionData != null &&
                (rd = oldReflectionData.get()) != null &&
                rd.redefinedCount == classRedefinedCount) {
                return rd;
            }
        }
    }
```
现在我们回到`privateGetDeclaredMethods`方法的实现，对于第3步：
```java
  // 3.没有缓存，通过 JVM 获取
        res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
```
调用的是native方法，此处不再赘述。
在获取到对应方法以后，并不会直接返回，如下：
```java
 return (res == null ? res : getReflectionFactory().copyMethod(res));
```
##### seq7
通过单步调试可发现`getReflectionFactory().copyMethod(res)`最终调用的是`Method#copy`
```java
    Method copy() {
	    // 1.该对象的root为null时，表明是 基本方法对象
        if (this.root != null)
	        // 2.只能拷贝基本方法对象即root为null的对象
            throw new IllegalArgumentException("Can not copy a non-root Method");
		// 3.新建一个与基本方法对象具有相同性质的方法对象
        Method res = new Method(clazz, name, parameterTypes, returnType,
                                exceptionTypes, modifiers, slot, signature,
                                annotations, parameterAnnotations, annotationDefault);
        // 4.res作为this的拷贝，其root属性必须指向this
        res.root = this;
        // Might as well eagerly propagate this if already present
        // 5.所有的Method的拷贝都会使用同一份methodAccessor
        res.methodAccessor = methodAccessor;
        return res;
    }
```
注意以下几点：
- **root 属性**：可以理解为每一个 java方法都有唯一的一个Method对象，这个对象就是root，它相当于根对象，对用户不可见。这个root是不会暴露给用户的，当我们通过反射获取Method对象时，新创建Method对象把root包装起来再给用户，我们代码中获得的Method对象都相当于它的副本（或引用）。
- **methodAccessor**：root 对象持有一个 MethodAccessor 对象，所以所有获取到的 Method对象都共享这一个 MethodAccessor 对象，因此必须保证它在内存中的可见性。
- `res.root = this`：res 作为 this 的拷贝，其 root 属性必须指向 this。
#### 小结
getMethod方法时序图
![Alt](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c17005e9ed274677ba02723348fd6889~tplv-k3u1fbpfcp-zoom-1.image)

## 2.3 Method.invoke
```java
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
	        // 1.检查权限
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        // 2.获取 MethodAccessor
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
	        // 2.1为空时创建MethodAccessor
            ma = acquireMethodAccessor();
        }
        // 3.调用 MethodAccessor.invoke
        return ma.invoke(obj, args);
    }
```
### 2.3.1 检查权限
这里对 override 变量进行判断，如果 override == true，就跳过检查 我们通常在 Method#invoke 之前，会调用 Method#setAccessible(true)，就是设置 override 值为 true。
### 2.3.2 获取 MethodAccessor
在上面获取 Method 的时候我们讲到过，Method#copy 会给 Method 的 methodAccessor 赋值。所以这里的 methodAccessor 就是拷贝时使用的 MethodAccessor。如果 ma 为空，就去创建 MethodAccessor。

```java
    /*
     注意这里没有使用synchronization。 为给定方法生成一个以上的MethodAccessor是正确的（尽管效率不高）。 但是，避免同步可能会使实现更具可伸缩性。
     */
    private MethodAccessor acquireMethodAccessor() {
        // First check to see if one has been created yet, and take it
        // if so
        MethodAccessor tmp = null;
        if (root != null) tmp = root.getMethodAccessor();
        if (tmp != null) {
            methodAccessor = tmp;
        } else {
            // Otherwise fabricate one and propagate it up to the root
            tmp = reflectionFactory.newMethodAccessor(this);
            setMethodAccessor(tmp);
        }

        return tmp;
    }
```
这里会先查找 root 的 MethodAccessor，这里的 root 在上面 Method#copy 中设置过。如果还是没有找到，就去创建 MethodAccessor。
```java
class ReflectionFactory {
    public MethodAccessor newMethodAccessor(Method method) {
        // 其中会对 noInflation 进行赋值
        checkInitted();
        // ...
        if (noInflation && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            // 生成的是 MethodAccessorImpl
            return new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        } else {
            NativeMethodAccessorImpl acc =
                new NativeMethodAccessorImpl(method);
            DelegatingMethodAccessorImpl res =
                new DelegatingMethodAccessorImpl(acc);
            acc.setParent(res);
            return res;
        }
    }
}
```
这里可以看到，一共有三种 MethodAccessor。MethodAccessorImpl，NativeMethodAccessorImpl，DelegatingMethodAccessorImpl。采用哪种 MethodAccessor 根据 noInflation 进行判断，noInflation 默认值为 false，只有指定了 sun.reflect.noInflation 属性为 true，才会 采用 MethodAccessorImpl。所以默认会调用 NativeMethodAccessorImpl。

MethodAccessorImpl 是通过动态生成字节码来进行方法调用的，是 Java 版本的 MethodAccessor，字节码生成比较复杂，这里不放代码了。大家感兴趣可以看这里的 generate 方法。

DelegatingMethodAccessorImpl 就是单纯的代理，真正的实现还是 NativeMethodAccessorImpl。
```java
class DelegatingMethodAccessorImpl extends MethodAccessorImpl {
    private MethodAccessorImpl delegate;
 
    DelegatingMethodAccessorImpl(MethodAccessorImpl delegate) {
        setDelegate(delegate);
    }
 
    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        return delegate.invoke(obj, args);
    }
 
    void setDelegate(MethodAccessorImpl delegate) {
        this.delegate = delegate;
    }
}
```
NativeMethodAccessorImpl 是 Native 版本的 MethodAccessor 实现。
```java
class NativeMethodAccessorImpl extends MethodAccessorImpl {
    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        // We can't inflate methods belonging to vm-anonymous classes because
        // that kind of class can't be referred to by name, hence can't be
        // found from the generated bytecode.
        if (++numInvocations > ReflectionFactory.inflationThreshold()
                && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            // Java 版本的 MethodAccessor
            MethodAccessorImpl acc = (MethodAccessorImpl)
                new MethodAccessorGenerator().
                    generateMethod(method.getDeclaringClass(),
                                   method.getName(),
                                   method.getParameterTypes(),
                                   method.getReturnType(),
                                   method.getExceptionTypes(),
                                   method.getModifiers());
            parent.setDelegate(acc);
        }
        // Native 版本调用
        return invoke0(method, obj, args);
    }
 
    private static native Object invoke0(Method m, Object obj, Object[] args);
}
```
在 NativeMethodAccessorImpl 的实现中，我们可以看到，有一个 numInvocations 阀值控制，numInvocations 表示调用次数。如果 numInvocations 大于 15（默认阀值是 15），那么就使用 Java 版本的 MethodAccessorImpl。
为什么采用这个策略呢，可以 JDK 中的注释：
>     // "Inflation" mechanism. Loading bytecodes to implement
    // Method.invoke() and Constructor.newInstance() currently costs
    // 3-4x more than an invocation via native code for the first
    // invocation (though subsequent invocations have been benchmarked
    // to be over 20x faster). Unfortunately this cost increases
    // startup time for certain applications that use reflection
    // intensively (but only once per class) to bootstrap themselves.
    // To avoid this penalty we reuse the existing JVM entry points
    // for the first few invocations of Methods and Constructors and
    // then switch to the bytecode-based implementations.
    //
    // Package-private to be accessible to NativeMethodAccessorImpl
    // and NativeConstructorAccessorImpl
    private static boolean noInflation        = false;
Java 版本的 MethodAccessorImpl 调用效率比 Native 版本要快 20 倍以上，但是 Java 版本加载时要比 Native 多消耗 3-4 倍资源，所以默认会调用 Native 版本，如果调用次数超过 15 次以后，就会选择运行效率更高的 Java 版本。那为什么 Native 版本运行效率会没有 Java 版本高呢？从 R 大博客来看，是因为 这是HotSpot的优化方式带来的性能特性，同时也是许多虚拟机的共同点：跨越native边界会对优化有阻碍作用，它就像个黑箱一样让虚拟机难以分析也将其内联，于是运行时间长了之后反而是托管版本的代码更快些。
### 2.3.3 调用 MethodAccessor#invoke 实现方法的调用
在生成 MethodAccessor 以后，就调用其 invoke 方法进行最终的反射调用。这里我们对 Java 版本的 MethodAccessorImpl 做个简单的分析，Native 版本暂时不做分析。在前面我们提到过 MethodAccessorImpl 是通过 MethodAccessorGenerator#generate 生成动态字节码然后动态加载到 JVM 中的。
到此，基本上 Java 方法反射的原理就介绍完了。

# 三、反射为什么慢？
## 3.1 为什么慢？
> Java实现的版本在初始化时需要较多时间，但长久来说性能较好；native版本正好相反，启动时相对较快，但运行时间长了之后速度就比不过Java版了。这是HotSpot的优化方式带来的性能特性，同时也是许多虚拟机的共同点：跨越native边界会对优化有阻碍作用，它就像个黑箱一样让虚拟机难以分析也将其内联，于是运行时间长了之后反而是托管版本的代码更快些。 为了权衡两个版本的性能，Sun的JDK使用了“inflation”的技巧：让Java方法在被反射调用时，开头若干次使用native版，等反射调用次数超过阈值时则生成一个专用的MethodAccessor实现类，生成其中的invoke()方法的字节码，以后对该Java方法的反射调用就会使用Java版。 当该反射调用成为热点时，它甚至可以被内联到靠近Method.invoke()的一侧，大大降低了反射调用的开销。而native版的反射调用则无法被有效内联，因而调用开销无法随程序的运行而降低。

总结来说，原因如下：
1. **jit 无法优化反射** 。 JIT编译器无法对反射有效做优化，引用一段java doc中的解释：
> 由于反射涉及动态解析的类型，导致无法执行某些Java虚拟机优化。所以，反射操作的性能比非反射操作慢，因此应避免在对性能敏感的应用程序中使用反射

2. **参数的封装/解封和方法的校验**。invoke方法是传Object类型的，如果是简单类型如long，在接口处必须封装成object，从而生成大量的Long的Object，导致了额外的不必要的内存浪费，甚至有可能导致GC；需要进行参数校验和方法的可见性校验。
3. **难以内联**

## 3.2 反射慢为什么还要用它？
反射这种技术被广泛应用于框架的设计中，但反射的确带来了一定的性能损耗，既然如此为什么还要用反射呢？
- 绝大部分系统的性能瓶颈还远远没有到需要考虑反射这里，逻辑层和数据层上的优化对性能的提升比优化反射高n个数量级。
- 框架的设计是性能、标准和开发效率等多个方面的权衡。
- 反射多少会有性能损耗，但一般可以忽略，而java对javabean方面的反射支持，java底层都有PropertyDescriptor和MethodDescriptor支持，可以一定程度的减少反射消耗。 AOP方面，cglib是通过类的字节码生成其子类去操作的，一旦子类生成就是纯粹的反射调用，不再操作字节码了，而一般AOP调用是在单例上，不会频繁的去用cglib生成子类。

关于反射性能的具体测试数据，可参考：https://www.jianshu.com/p/4e2b49fa8ba1
其实通过以上数据可以看出，当量非常大的时候，反射确实是会影响性能。但一般的应用，即使不借助高性能工具包也不会是程序掣肘。当然，这也不是意味着可以随意使用，还是要结合实际的应用来。


# 四、反射优缺点
## 4.1 反射的优点
- 赋予程序在运行时可以操作对象的能力。
- 解耦，提高程序的可扩展性。
## 4.2 反射的缺点
**性能开销**
反射涉及类型动态解析，所以JVM无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
**安全限制**
使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如Applet，那么这就是个问题了。
**内部曝光**
由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用－－代码有功能上的错误，降低可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

## 4.3 原则
如果使用常规方法能够实现，那么就不要用反射。

# 五、参考文献
https://www.jianshu.com/p/607ff4e79a13
https://www.cnblogs.com/chanshuyi/p/head_first_of_reflection.html
https://blog.csdn.net/zhenghongcs/article/details/103143144

**最后**
- 文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处
