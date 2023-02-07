---
title: Spring AOP系列(三) — 动态代理之JDK动态代理
tags:
  - aop
categories:
  - 总结
date: 2021-07-17 00:00:00
---
# JDK动态代理
JDK动态代理核心是两个类：`InvocationHandler`和 `Proxy`
## 举个栗子
为便于理解，首先看一个例子：
希望实现这样一个功能：使用 `UserService` 时，只需关注自己的核心业务逻辑的实现，对于日志功能的打印，由系统的公共服务完成。
首先定义一个业务类的接口：`UserService.java`

<!-- more -->

```java
package com.proxy;
/**
 * @author: create by lengzefu
 * @description: com.proxy
 * @date:2020-09-15
 */
public interface UserService {
    void login();
}
```
实现该接口：
```java
package com.proxy;

/**
 * @author: create by lengzefu
 * @description: com.proxy
 * @date:2020-09-15
 */
public class UserServiceImpl implements UserService {
    @Override
    public void login() {
        System.out.println("用户登录...");
    }
}
```
定义一个`InvocationHandler`实现类，该实现类有我们想要添加的公共日志
```java
package com.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Date;

/**
 * @author: create by lengzefu
 * @description: com.proxy
 * @date:2020-09-15
 */
public class LogHandler implements InvocationHandler {
	// 被代理的对象，这里指的是UserServiceImpl对象
    Object target;

	// 构造函数，将被代理对象传进来
    public LogHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(args);
        after();
        return result;
    }

    private void before() {
        System.out.println("方法调用开始时间:" + new Date());
    }

    private void after() {
        System.out.println("方法调用结束时间:" + new Date());
    }
}
```
接下来看看如何在客户端中调用
```java
/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
        // 设置变量可以保存动态代理类，默认名称以 $Proxy0 格式命名
        // System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 1. 创建被代理的对象，UserService接口的实现类
        UserService userService = new UserServiceImpl();
        // 2. 获取对应的 ClassLoader
        ClassLoader classLoader = userService.getClass().getClassLoader();
        // 3. 获取所有接口的Class，这里的UserServiceImpl只实现了一个接口UserService
        Class[] interfaces = userService.getClass().getInterfaces();
        // 4. 创建一个将传给代理类的调用请求处理器，处理所有的代理对象上的方法调用
        // 这里创建的是一个自定义的日志处理器，须传入实际的执行对象 userServiceImpl
        InvocationHandler logHandler = new LogHandler(userService);
        /*
		   5.根据上面提供的信息，创建代理对象 在这个过程中，
               a.JDK会通过根据传入的参数信息动态地在内存中创建和.class 文件等同的字节码
               b.然后根据相应的字节码转换成对应的class，
               c.然后调用newInstance()创建代理实例
		 */
        UserService proxy = (UserService) Proxy.newProxyInstance(classLoader, interfaces, logHandler);

        // 调用代理的方法
        proxy.login();
        // 保存JDK动态代理生成的代理类，类名保存为 UserServiceProxy
        //ProxyUtils.generateClassFile(userServiceImpl.getClass(), "UserServiceProxy");

    }
}
```
开始一步步分析源码
## 源码分析
### 1.Proxy.newProxyInstance( ClassLoader loader, Class[] interfaces, InvocationHandler h)
`Proxy.newProxyInstance( ClassLoader loader, Class[] interfaces, InvocationHandler h)`产生了代理对象，首先分析下方法的三个参数
- `ClassLoader loader`：类加载器，类加载器用于加载代理类。这里经过测试发现不论是传userService的类加载器，还是logHandler的类加载器，得到的结果都是一样。
- `Class[] interfaces`：被代理类实现的接口集合。有了它，代理类就可以实现被代理类的所有接口。
- `InvocationHandler `h：handler对象，不能为null。

接下来看一下方法的具体实现：
```java
 public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
	    //检验handler对象不能为空
        Objects.requireNonNull(h);
		//接口的类对象拷贝一份（接口也是一个对象，是一个类名为Class的对象）
        final Class<?>[] intfs = interfaces.clone();
        //安全检查，不是重点
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         * 查询（在缓存中已经有）或生成指定的代理类的class对象。
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         * 用指定的调用处理程序（什么程序？）调用其（谁的？）构造函数
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
			//返回constructorParams的公共构造函数
			//参数constructorParames为常量值：private static final Class<?>[] constructorParams = { InvocationHandler.class };
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            //这里生成了代理对象，关于这里的详解：https://www.cnblogs.com/ferryman/p/12089210.html
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
这段代码核心就是通过`getProxyClass0(loader, intfs)`得到代理类的Class对象，然后通过Class对象得到构造方法，进而创建代理对象。下一步看`getProxyClass0`这个方法。
### 2.getProxyClass0
```java
//此方法也是Proxy类下的方法
   private static Class<?> getProxyClass0(ClassLoader loader,
                                          Class<?>... interfaces) {
       if (interfaces.length > 65535) {
           throw new IllegalArgumentException("interface limit exceeded");
       }
 
       // If the proxy class defined by the given loader implementing
       // the given interfaces exists, this will simply return the cached copy;
       // otherwise, it will create the proxy class via the ProxyClassFactory
       //意思是：如果代理类被指定的类加载器loader定义了，并实现了给定的接口interfaces，
       //那么就返回缓存的代理类对象，否则使用ProxyClassFactory创建代理类。
       return proxyClassCache.get(loader, interfaces);
   }
```
### 3.proxyClassCache.get
这里看到 `proxyClassCache`，有Cache便知道是缓存的意思，正好呼应了前面Look up or generate the designated proxy class。查询（在缓存中已经有）或生成指定的代理类的class对象这段注释。
`proxyClassCache`是个`WeakCache`类的对象，调用`proxyClassCache.get(loader, interfaces)`; 可以得到缓存的代理类或创建代理类（没有缓存的情况）。先看下`WeakCache`类的定义（这里先只给出变量的定义和构造函数）：
```java
//K代表key的类型，P代表参数的类型，V代表value的类型。
// WeakCache<ClassLoader, Class<?>[], Class<?>>  proxyClassCache  说明proxyClassCache存的值是Class<?>对象，正是我们需要的代理类对象。
final class WeakCache<K, P, V> {
 
   private final ReferenceQueue<K> refQueue
       = new ReferenceQueue<>();
   // the key type is Object for supporting null key
   private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
       = new ConcurrentHashMap<>();
   private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
       = new ConcurrentHashMap<>();
   private final BiFunction<K, P, ?> subKeyFactory;
   private final BiFunction<K, P, V> valueFactory;
 
 
   public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                    BiFunction<K, P, V> valueFactory) {
       this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
       this.valueFactory = Objects.requireNonNull(valueFactory);
   }
```
其中map变量是实现缓存的核心变量，他是一个双重的Map结构: (key, sub-key) -> value。其中key是传进来的Classloader进行包装后的对象，sub-key是由WeakCache构造函数传入的KeyFactory()生成的。value就是产生代理类的对象，是由WeakCache构造函数传入的ProxyClassFactory()生成的。如下，回顾一下:
```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
       proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```
产生sub-key的KeyFactory代码如下，这个我们不去深究，只要知道他是根据传入的ClassLoader和接口类生成sub-key即可。
```java
private static final class KeyFactory
       implements BiFunction<ClassLoader, Class<?>[], Object>
   {
       @Override
       public Object apply(ClassLoader classLoader, Class<?>[] interfaces) {
           switch (interfaces.length) {
               case 1: return new Key1(interfaces[0]); // the most frequent
               case 2: return new Key2(interfaces[0], interfaces[1]);
               case 0: return key0;
               default: return new KeyX(interfaces);
           }
       }
   }
```
通过sub-key拿到一个`Supplier<Class<?>>`对象，然后调用这个对象的get方法，最终得到代理类的Class对象。

好，大体上说完 `WeakCache` 这个类的作用，我们回到刚才`proxyClassCache.get(loader, interfaces)`;这句代码。get是 `WeakCache` 里的方法。源码如下：
```java
//K和P就是WeakCache定义中的泛型，key是类加载器，parameter是接口类数组
public V get(K key, P parameter) {
       //检查parameter不为空
       Objects.requireNonNull(parameter);
        //清除无效的缓存
       expungeStaleEntries();
       // cacheKey就是(key, sub-key) -> value里的一级key，
       Object cacheKey = CacheKey.valueOf(key, refQueue);
 
       // lazily install the 2nd level valuesMap for the particular cacheKey
       //根据一级key得到 ConcurrentMap<Object, Supplier<V>>对象。如果之前不存在，则新建一个ConcurrentMap<Object, Supplier<V>>和cacheKey（一级key）一起放到map中。
        ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
       if (valuesMap == null) {
           ConcurrentMap<Object, Supplier<V>> oldValuesMap
               = map.putIfAbsent(cacheKey,
                                 valuesMap = new ConcurrentHashMap<>());
           if (oldValuesMap != null) {
               valuesMap = oldValuesMap;
           }
       }
 
       // create subKey and retrieve the possible Supplier<V> stored by that
       // subKey from valuesMap
       //这部分就是调用生成sub-key的代码，上面我们已经看过怎么生成的了
       Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
       //通过sub-key得到supplier
       Supplier<V> supplier = valuesMap.get(subKey);
       //supplier实际上就是这个factory
       Factory factory = null;
 
       while (true) {
           //如果缓存里有supplier ，那就直接通过get方法，得到代理类对象，返回，就结束了，一会儿分析get方法。
            if (supplier != null) {
               // supplier might be a Factory or a CacheValue<V> instance
               V value = supplier.get();
               if (value != null) {
                   return value;
               }
           }
           // else no supplier in cache
           // or a supplier that returned null (could be a cleared CacheValue
           // or a Factory that wasn't successful in installing the CacheValue)
           // lazily construct a Factory
           //下面的所有代码目的就是：如果缓存中没有supplier，则创建一个Factory对象，把factory对象在多线程的环境下安全的赋给supplier。
            //因为是在while（true）中，赋值成功后又回到上面去调get方法，返回才结束。
           if (factory == null) {
               factory = new Factory(key, parameter, subKey, valuesMap);
           }
			
			// 双重判空，实现线程安全？
           if (supplier == null) {
               supplier = valuesMap.putIfAbsent(subKey, factory);
               if (supplier == null) {
                   // successfully installed Factory
                   supplier = factory;
               }
               // else retry with winning supplier
           } else {
               if (valuesMap.replace(subKey, supplier, factory)) {
                   // successfully replaced
                   // cleared CacheEntry / unsuccessful Factory
                   // with our Factory
                   supplier = factory;
               } else {
                   // retry with current supplier
                   supplier = valuesMap.get(subKey);
               }
           }
       }
   }
```
### 4.Factory.get
supplier就是factory，所以接下来我们看看Factory类（Factory是WeakCache的内部类）的get方法。
```java
public synchronized V get() { // serialize access
    // re-check
    Supplier<V> supplier = valuesMap.get(subKey);
    /重新检查得到的supplier是不是当前对象
           if (supplier != this) {
               // something changed while we were waiting:
               // might be that we were replaced by a CacheValue
               // or were removed because of failure ->
               // return null to signal WeakCache.get() to retry
               // the loop
               return null;
           }
           // else still us (supplier == this)
 
           // create new value
           V value = null;
           try {
                //代理类就是在这个位置调用valueFactory生成的
                //valueFactory就是我们传入的 new ProxyClassFactory()
               //一会我们分析ProxyClassFactory()的apply方法
               value = Objects.requireNonNull(valueFactory.apply(key, parameter));
           } finally {
               if (value == null) { // remove us on failure
                   valuesMap.remove(subKey, this);
               }
           }
           // the only path to reach here is with non-null value
           assert value != null;
 
           // wrap value with CacheValue (WeakReference)
           //把value包装成弱引用
           CacheValue<V> cacheValue = new CacheValue<>(value);
 
           // put into reverseMap
           // reverseMap是用来实现缓存的有效性
           reverseMap.put(cacheValue, Boolean.TRUE);
 
           // try replacing us with CacheValue (this should always succeed)
           if (!valuesMap.replace(subKey, this, cacheValue)) {
               throw new AssertionError("Should not reach here");
           }
 
           // successfully replaced us with new CacheValue -> return the value
           // wrapped by it
           return value;
       }
   }
```
### 5.ProxyClassFactory.apply
最后来到 `ProxyClassFactory` 的 apply 方法，代理类就是在这里生成的。
```java
//这里的BiFunction<T, U, R>是个函数式接口，可以理解为用T，U两种类型做参数，得到R类型的返回值
private static final class ProxyClassFactory
       implements BiFunction<ClassLoader, Class<?>[], Class<?>>
   {
       // prefix for all proxy class names
       //所有代理类名字的前缀
       private static final String proxyClassNamePrefix = "$Proxy";
       
       // next number to use for generation of unique proxy class names
       //用于生成代理类名字的计数器
       private static final AtomicLong nextUniqueNumber = new AtomicLong();
 
       @Override
       public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
             
           Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            //验证代理接口，可不看
           for (Class<?> intf : interfaces) {
               /*
                * Verify that the class loader resolves the name of this
                * interface to the same Class object.
                */
               Class<?> interfaceClass = null;
               try {
                   interfaceClass = Class.forName(intf.getName(), false, loader);
               } catch (ClassNotFoundException e) {
               }
               if (interfaceClass != intf) {
                   throw new IllegalArgumentException(
                       intf + " is not visible from class loader");
               }
               /*
                * Verify that the Class object actually represents an
                * interface.
                */
               if (!interfaceClass.isInterface()) {
                   throw new IllegalArgumentException(
                       interfaceClass.getName() + " is not an interface");
               }
               /*
                * Verify that this interface is not a duplicate.
                */
               if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                   throw new IllegalArgumentException(
                       "repeated interface: " + interfaceClass.getName());
               }
           }
           //生成的代理类的包名 
           String proxyPkg = null;     // package to define proxy class in
           //代理类访问控制符: public ,final
           int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
 
           /*
            * Record the package of a non-public proxy interface so that the
            * proxy class will be defined in the same package.  Verify that
            * all non-public proxy interfaces are in the same package.
            */
           //验证所有非公共的接口在同一个包内；公共的就无需处理
           //生成包名和类名的逻辑，包名默认是com.sun.proxy，类名默认是$Proxy 加上一个自增的整数值
            //如果被代理类是 non-public proxy interface ，则用和被代理类接口一样的包名
           for (Class<?> intf : interfaces) {
               int flags = intf.getModifiers();
               if (!Modifier.isPublic(flags)) {
                   accessFlags = Modifier.FINAL;
                   String name = intf.getName();
                   int n = name.lastIndexOf('.');
                   String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                   if (proxyPkg == null) {
                       proxyPkg = pkg;
                   } else if (!pkg.equals(proxyPkg)) {
                       throw new IllegalArgumentException(
                           "non-public interfaces from different packages");
                   }
               }
           }
 
           if (proxyPkg == null) {
               // if no non-public proxy interfaces, use com.sun.proxy package
               proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
           }
 
           /*
            * Choose a name for the proxy class to generate.
            */
           long num = nextUniqueNumber.getAndIncrement();
           //代理类的完全限定名，如com.sun.proxy.$Proxy0.calss
           String proxyName = proxyPkg + proxyClassNamePrefix + num;
 
           /*
            * Generate the specified proxy class.
            */
           //核心部分，生成代理类的字节码
           byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
               proxyName, interfaces, accessFlags);
           try {
               //把代理类加载到JVM中，至此动态代理过程基本结束了
               return defineClass0(loader, proxyName,
                                   proxyClassFile, 0, proxyClassFile.length);
           } catch (ClassFormatError e) {
               /*
                * A ClassFormatError here means that (barring bugs in the
                * proxy class generation code) there was some other
                * invalid aspect of the arguments supplied to the proxy
                * class creation (such as virtual machine limitations
                * exceeded).
                */
               throw new IllegalArgumentException(e.toString());
           }
       }
   }
```
### 6.字节码文件分析
到这里其实已经分析完了，但是本着深究的态度，决定看看JDK生成的动态代理字节码是什么，于是将字节码保存到磁盘上的class文件中。代码如下：
```java
package com.sun.proxy;

import com.leng.proxy.dynamic.UserService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements UserService {

   private static Method m1;
   private static Method m2;
   private static Method m3;
   private static Method m0;

   // 代理类的构造函数，其参数就是InvovationHandler实例
   // Proxy.newInstance方法就是通过这个构造函数来创建代理实例的
   public $Proxy0(InvocationHandler var1) throws  {
      super(var1);
   }

   // 固定的三个类方法 equals, toString, hashCode
   public final boolean equals(Object var1) throws  {
      try {
         return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
      } catch (RuntimeException | Error var3) {
         throw var3;
      } catch (Throwable var4) {
         throw new UndeclaredThrowableException(var4);
      }
   }

   public final String toString() throws  {
      try {
         return (String)super.h.invoke(this, m2, (Object[])null);
      } catch (RuntimeException | Error var2) {
         throw var2;
      } catch (Throwable var3) {
         throw new UndeclaredThrowableException(var3);
      }
   }

   //接口代理方法
   public final void login() throws  {
      try {
         // invocation handler的invoke方法在这里被调用
         super.h.invoke(this, m3, (Object[])null);
      } catch (RuntimeException | Error var2) {
         throw var2;
      } catch (Throwable var3) {
         throw new UndeclaredThrowableException(var3);
      }
   }

   public final int hashCode() throws  {
      try {
         return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
      } catch (RuntimeException | Error var2) {
         throw var2;
      } catch (Throwable var3) {
         throw new UndeclaredThrowableException(var3);
      }
   }

   static {
      try {
         m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
         m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
         m3 = Class.forName("com.leng.proxy.dynamic.UserService").getMethod("login", new Class[0]);
         m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      } catch (NoSuchMethodException var2) {
         throw new NoSuchMethodError(var2.getMessage());
      } catch (ClassNotFoundException var3) {
         throw new NoClassDefFoundError(var3.getMessage());
      }
   }
}
```

**最后**
- 文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处
