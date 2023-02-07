---
title: Spring AOP系列(四) — 动态代理之CGLib动态代理
tags:
  - aop
categories:
  - 总结
date: 2021-07-18 00:00:00
---

# 前言
JDK动态代理要求被代理的类必须实现接口，而生成的代理类也只能代理某个类接口定义的方法，这有很强的局限性。而CGLIB动态代理没有这个要求。简单来说，两者的区别有以下几点：
1. Java动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为Proxy，Java类继承机制不允许多重继承）；CGLIB能够代理普通类。
2. Java动态代理使用Java原生的反射API进行操作，在生成类上比较高效；CGLIB 使用ASM框架直接对字节码进行操作，在类的执行过程中比较高效。

<!-- more -->

# 一、举个栗子
## 1.1 创建一个没有实现任何接口的类
```java
package com.leng.proxy.dynamic.cglib;
/**
 * @Classname UserServiceImpl
 * @Date 2020/9/22 21:58
 * @Autor lengxuezhang
 */
public class UserServiceImpl {

    public void login() {
        System.out.println("用户登录......");
    }

    public void sayHello() {
        System.out.println("Hello World!!!");
    }
}
```
想实现对`UserServiceImpl`的动态代理，使用JDK动态代理是无法实现的，因为没有实现接口，`UserServiceImpl`就是一个普通类。可以通过CGLIB实现代理，步骤如下：
1. 首先实现一个`MethodInterceptor`，方法调用会被转发到该类的intercept()方法。
2. 然后在需要使用`UserServiceImpl`的时候，通过CGLIB动态代理获取代理对象。
## 1.2 实现`MethodInterceptor`接口
```java
package com.leng.proxy.dynamic.cglib;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;
import java.util.Date;
/**
 * @Classname LogInterceptor
 * @Date 2020/9/22 21:59
 * @Autor lengxuezhang
 */
public class LogInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(o, objects);
        after();
        return result;
    }

    // 预处理方法
    private void before() {
        System.out.println(String.format("log start time [%s] ", new Date()));
    }

    // 后处理方法
    private void after() {
        System.out.println(String.format("log end time [%s] ", new Date()));
    }
}
```
`intercept`方法四个参数的含义如下：
`obj`: 代理类对象。
`method`: 被代理的类中的方法。
`args`: 调用方法需要的参数。
`proxy`: 生成的代理类对方法的“代理引用”。----?这句是什么意思？


用户需要实现`MethodInterceptor`接口，实现对方法的拦截。这一点与JDK动态代理中用户需要实现`InvocationHandler`接口类似。

## 1.3 客户端中使用CGLIB
代码如下：
```java
package com.leng;
import com.leng.proxy.dynamic.cglib.LogInterceptor;
import com.leng.proxy.dynamic.cglib.UserServiceImpl;
import org.springframework.cglib.proxy.Enhancer;
/**
 * @Classname Client
 * @Date 2020/9/12 2:40
 * @Autor lengxuezhang
 */
public class Client {
    public static void main(String[] args) {
	    //CGLIB加强器，用于生成代理对象，类比JDK的Proxy
        Enhancer enhancer = new Enhancer();
        //设置加强器继承被代理类
        enhancer.setSuperclass(UserServiceImpl.class);
        //设置回调
        enhancer.setCallback(new LogInterceptor());
        //生成代理类对象
        UserServiceImpl userService = (UserServiceImpl) enhancer.create();
        //调用代理类对象中的方法时，会被我们实现的方法拦截器所拦截
        userService.login();
        userService.sayHello();
    }
}
```
**Enhancer:** `Enhancer`是CGLIB中最常用的一个类，和Java1.3动态代理中引入的Proxy类差不多。但和Proxy不同的是，<font color=red>Enhancer既能够代理普通的class，也能够代理接口。</font>Enhancer创建一个被代理对象的子类并且拦截所有的方法调用（包括从Object中继承的toString和hashCode方法）。<font color=red>Enhancer不能拦截final方法</font>，例如`Object.getClass()`方法，这是由于Java final方法语义决定的。基于同样的道理，Enhancer也不能对fianl类进行代理操作。

`Enhancer.setSuperclass`用来设置父类型，`Enhancer.create(Object…)`方法是用来创建增强对象的，其提供了很多不同参数的方法用来匹配被增强类的不同构造方法。（虽然类的构造方法只是Java字节码层面的函数，但是Enhancer却不能对其进行操作。Enhancer同样不能操作static或者final类）。我们也可以先使用`Enhancer.createClass()`来创建字节码(.class)，然后用字节码动态的生成增强后的对象。
# 二、源码分析
## 2.1 生成指定类的Class对象字节数组
指定类是哪个类？
如上客户端中的调用代码，首先创建`Enhancer`对象，设置`SuperClass`父类（被代理类），然后调用`Enhancer`对象的`create()`方法。
这里对`Enhancer`类需要特别提出一点：`Enhancer`类也是`AbstractClassGenerator`的子类，用处后面遇到会详细说。-
### 1. `create()`
```java
	/**
	 * Generate a new class if necessary and uses the specified
	 * callbacks (if any) to create a new object instance.
	 * Uses the no-arg constructor of the superclass.
	 * @return a new instance
	 */
	public Object create() {
		classOnly = false;
		argumentTypes = null;
		// 实际调用的方法
		return createHelper();
	}

	private Object createHelper() {
		//1.进行有效性验证
		//1.1 callBack不能为空，也就是说至少要有一个callBack(callBack与代理类紧密相关)
		//1.2 有多个callBack时必须有callBackFilter
		preValidate();
		//2.先根据KEY_FACTORY 以当前代理类的配置信息 生成一个组合Key，再利用这个组合Key，进行create
		Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
				ReflectUtils.getNames(interfaces),
				filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
				callbackTypes,
				useFactory,
				interceptDuringConstruction,
				serialVersionUID);
		this.currentKey = key;
		//根据生成的key创建代理对象
		Object result = super.create(key);
		return result;
	}
```
接下来来到`AbstractClassGenerator`类中的`create`方法，这一步就是为了得到动态类的Class对象（Class的对象，认作A），之后通过**反射**生成具体的类的对象（A的对象）
```java	
protected Object create(Object key) {
		try {
			//1.获取当前类加载器，应用类加载器
			ClassLoader loader = getClassLoader();
			//2.缓存，一级缓存的key是类加载器,value是ClassLoaderData
			//2.1 cache中有则直接获取
			Map<ClassLoader, ClassLoaderData> cache = CACHE;
			ClassLoaderData data = cache.get(loader);
			//2.2 cache中没有则生成
			if (data == null) {
				synchronized (AbstractClassGenerator.class) {
					cache = CACHE;
					data = cache.get(loader);
					if (data == null) {
						Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
						data = new ClassLoaderData(loader);
						newCache.put(loader, data);
						CACHE = newCache;
					}
				}
			}
			this.key = key;
			//3.调用 get方法获取字节码，如果没有字节码，则会创建字节码（Class对象）
			Object obj = data.get(this, getUseCache());
			//4.生成动态代理类
			if (obj instanceof Class) {
				return firstInstance((Class) obj);
			}
			return nextInstance(obj);
		}
		catch (RuntimeException | Error ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new CodeGenerationException(ex);
		}
	}
```
如代码中的注释，该方法的3、4是核心的步骤，首先详细分析下3，我们进到`get`方法中
```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
    //判断是否开启缓存，可直接设置：enhancer.setUseCache(false);默认为true
	if (!useCache) {
		return gen.generate(ClassLoaderData.this);
	}
	else {
		Object cachedValue = generatedClasses.get(gen);
		return gen.unwrapCachedValue(cachedValue);
	}
}
```
关于 `AbstractClassGenerator` 的详细解析，可参考：
> 死磕cglib系列之二 AbstractClassGenerator缓存解析：https://blog.csdn.net/zhang6622056/article/details/87783480

这里我们直接来看`generate`方法
```java
	protected Class generate(ClassLoaderData data) {
		Class gen;
		Object save = CURRENT.get();
		CURRENT.set(this);
		try {
			// 1.判断有无classLoader
			ClassLoader classLoader = data.getClassLoader();
			if (classLoader == null) {
				throw new IllegalStateException("ClassLoader is null while trying to define class " +
						getClassName() + ". It seems that the loader has been expired from a weak reference somehow. " +
						"Please file an issue at cglib's issue tracker.");
			}
			// 2.生成动态代理的类名
			synchronized (classLoader) {
				// 生成代理类名称
				String name = generateClassName(data.getUniqueNamePredicate());
				data.reserveName(name);
				this.setClassName(name);
			}
			if (attemptLoad) {
				try {
					gen = classLoader.loadClass(getClassName());
					return gen;
				}
				catch (ClassNotFoundException e) {
					// ignore
				}
			}
			// 3.生成动态代理类的字节码
			byte[] b = strategy.generate(this);
			String className = ClassNameReader.getClassName(new ClassReader(b));
			ProtectionDomain protectionDomain = getProtectionDomain();
			// 4.生成class文件
			synchronized (classLoader) { // just in case
				// SPRING PATCH BEGIN
				gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain, contextClass);
				// SPRING PATCH END
			}
			return gen;
		}
		catch (RuntimeException | Error ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new CodeGenerationException(ex);
		}
		finally {
			CURRENT.set(save);
		}
	}
```
注释中的3的方法是`DefaultGeneratorStrategy`中的方法如下：
```java
  public byte[] generate(ClassGenerator cg) throws Exception {
      DebuggingClassWriter cw = this.getClassVisitor();
      this.transform((ClassGenerator)cg).generateClass(cw);
      return this.transform((byte[])cw.toByteArray());
   }
```
注意，这里是一个访问者模式。`getClassVistor`调用了asm的接口，生成了一个`DebuggingClassWriter`对象。注意这里的cg就是之前的Enhancer实例，它overrided  generateClass方法。
于是此时查看generateClass方法，又回到Enhance类：
```java
public void generateClass(ClassVisitor v) throws Exception {
		Class sc = (superclass == null) ? Object.class : superclass;

		if (TypeUtils.isFinal(sc.getModifiers()))
			throw new IllegalArgumentException("Cannot subclass final class " + sc.getName());
		List constructors = new ArrayList(Arrays.asList(sc.getDeclaredConstructors()));
		filterConstructors(sc, constructors);

		// Order is very important: must add superclass, then
		// its superclass chain, then each interface and
		// its superinterfaces.
		List actualMethods = new ArrayList();
		List interfaceMethods = new ArrayList();
		final Set forcePublic = new HashSet();
		// 1.得到所有的方法，包括基类、接口
		getMethods(sc, interfaces, actualMethods, interfaceMethods, forcePublic);

		List methods = CollectionUtils.transform(actualMethods, new Transformer() {
			public Object transform(Object value) {
				Method method = (Method) value;
				int modifiers = Constants.ACC_FINAL
						| (method.getModifiers()
						& ~Constants.ACC_ABSTRACT
						& ~Constants.ACC_NATIVE
						& ~Constants.ACC_SYNCHRONIZED);
				if (forcePublic.contains(MethodWrapper.create(method))) {
					modifiers = (modifiers & ~Constants.ACC_PROTECTED) | Constants.ACC_PUBLIC;
				}
				return ReflectUtils.getMethodInfo(method, modifiers);
			}
		});

		// 2.生成字节码，参数还是之前的classWriter
		// 2.1 这里的className就是之前生成的className
		ClassEmitter e = new ClassEmitter(v);
		if (currentData == null) {
			e.begin_class(Constants.V1_2,
					Constants.ACC_PUBLIC,
					getClassName(),
					Type.getType(sc),
					(useFactory ?
							TypeUtils.add(TypeUtils.getTypes(interfaces), FACTORY) :
							TypeUtils.getTypes(interfaces)),
					Constants.SOURCE_FILE);
		}
		else {
			e.begin_class(Constants.V1_2,
					Constants.ACC_PUBLIC,
					getClassName(),
					null,
					new Type[]{FACTORY},
					Constants.SOURCE_FILE);
		}
		List constructorInfo = CollectionUtils.transform(constructors, MethodInfoTransformer.getInstance());

		// 2.2 这些都是字段，之后我们会在生成的文件中看到
		e.declare_field(Constants.ACC_PRIVATE, BOUND_FIELD, Type.BOOLEAN_TYPE, null);
		e.declare_field(Constants.ACC_PUBLIC | Constants.ACC_STATIC, FACTORY_DATA_FIELD, OBJECT_TYPE, null);
		if (!interceptDuringConstruction) {
			e.declare_field(Constants.ACC_PRIVATE, CONSTRUCTED_FIELD, Type.BOOLEAN_TYPE, null);
		}
		e.declare_field(Constants.PRIVATE_FINAL_STATIC, THREAD_CALLBACKS_FIELD, THREAD_LOCAL, null);
		e.declare_field(Constants.PRIVATE_FINAL_STATIC, STATIC_CALLBACKS_FIELD, CALLBACK_ARRAY, null);
		if (serialVersionUID != null) {
			e.declare_field(Constants.PRIVATE_FINAL_STATIC, Constants.SUID_FIELD_NAME, Type.LONG_TYPE, serialVersionUID);
		}

		// 2.4 这里就是生成的callback，命名就是CGLIB&CALLBACK_在数组中的序号
		for (int i = 0; i < callbackTypes.length; i++) {
			e.declare_field(Constants.ACC_PRIVATE, getCallbackField(i), callbackTypes[i], null);
		}
		// This is declared private to avoid "public field" pollution
		e.declare_field(Constants.ACC_PRIVATE | Constants.ACC_STATIC, CALLBACK_FILTER_FIELD, OBJECT_TYPE, null);

		if (currentData == null) {
			// 2.5 filter在这里发生作用
			emitMethods(e, methods, actualMethods);
			emitConstructors(e, constructorInfo);
		}
		else {
			emitDefaultConstructor(e);
		}
		emitSetThreadCallbacks(e);
		emitSetStaticCallbacks(e);
		emitBindCallbacks(e);

		if (useFactory || currentData != null) {
			int[] keys = getCallbackKeys();
			emitNewInstanceCallbacks(e);
			emitNewInstanceCallback(e);
			emitNewInstanceMultiarg(e, constructorInfo);
			emitGetCallback(e, keys);
			emitSetCallback(e, keys);
			emitGetCallbacks(e);
			emitSetCallbacks(e);
		}

		e.end_class();
	}
```
现在回到前面说到的“生成动态代理类对象”的方法：
```java
	//4.生成动态代理类
	if (obj instanceof Class) {
		return firstInstance((Class) obj);
	}
```
进到`firstInstance`方法，同样是Enhancer类中
```java
	/**
	 * This method should not be called in regular flow.
	 * Technically speaking {@link #wrapCachedClass(Class)} uses {@link EnhancerFactoryData} as a cache value,
	 * and the latter enables faster instantiation than plain old reflection lookup and invoke.
	 * This method is left intact for backward compatibility reasons: just in case it was ever used.
	 * @param type class to instantiate
	 * @return newly created proxy instance
	 * @throws Exception if something goes wrong
	 */
	protected Object firstInstance(Class type) throws Exception {
		if (classOnly) {
			return type;
		}
		else {
			// 从名字也能看出来是利用了反射来生成代理类对象的
			return createUsingReflection(type);
		}
	}
```
使用到了`ReflectUtils`反射工具类中的方法，完成了动态代理对象的生成。

# 三、参考文献：
- https://www.cnblogs.com/cruze/p/3865180.html
- https://blog.csdn.net/woshilijiuyi/article/details/83448407
- https://www.jianshu.com/p/20203286ccd9

**最后**
- 文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处

