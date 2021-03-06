---
title: 动态代理模式
categories: [Java, DesignPattern]
tags: [design-pattern, java]
---
[装饰模式 vs (静态)代理模式](https://leon-wtf.github.io/java/2019/06/20/decorative-static-proxy-pattern/)中提到,在静态代理模式中，针对每一个需要被代理的类都要在编译前就提前写好一个代理类，这样做增加了类管理的复杂性，如果我们可以在运行期间动态的来生成这个代理类，就会方便很多，这就是动态代理模式的核心思想，也是Spring中AOP(Aspect Oriented Programming)的实现原理。动态代理有两种实现方法：jdk动态代理和cglib动态代理，下面分别来具体看一下：

### jdk动态代理 ###
我们知道，在java中如果想在运行期动态的生成一个类，就要借助反射机制。jdk动态代理就是通过*java.lang.reflect.Proxy*利用反射机制提供了一种原生的动态代理模式，它提供了一个静态方法：
```java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

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
它的三个参数分别是：
- loader: the class loader to define the proxy class
- interfaces: the list of interfaces for the proxy class to implement
- h: the invocation handler to dispatch method invocations to

其中*getProxyClass0*会调用Proxy的内部类*ProxyClassFactory*的*apply*方法，然后调用*ProxyGenerator*里的*generateProxyClass*生成Class字节码数组，再利用*defineClass0*将字节码数组转成Class对象返回。
```java
// In apply of apply
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);
try {
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
// In ProxyGenerator
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
	ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
	final byte[] var4 = var3.generateClassFile();
	if (saveGeneratedFiles) {
		AccessController.doPrivileged(new PrivilegedAction<Void>() {
			public Void run() {
				try {
					int var1 = var0.lastIndexOf(46);
					Path var2;
					if (var1 > 0) {
						Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
						Files.createDirectories(var3);
						var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
					} else {
						var2 = Paths.get(var0 + ".class");
					}

					Files.write(var2, var4, new OpenOption[0]);
					return null;
				} catch (IOException var4x) {
					throw new InternalError("I/O exception saving generated file: " + var4x);
				}
			}
		});
	}
	return var4;
}
```
然后利用这个Class对象传入InvocationHandler参数构建一个代理类实例。在运行当前main方法的路径下创建com/sun/proxy目录，并创建一个$*Proxy0*.class文件，然后设置*sun.misc.ProxyGenerator.saveGeneratedFiles*系统属性为true，反编$*Proxy0.class*文件可以看到：
```java
public final class $Proxy0 extends Proxy implements UserManager {
  private static Method m1;
  private static Method m3;
  private static Method m0;
  private static Method m2;
 
  public $Proxy0(InvocationHandler paramInvocationHandler) {
    super(paramInvocationHandler);
  }
 
  public final boolean equals(Object paramObject) {
    try {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
 
  public final void addUser(String paramString) {
    try {
      this.h.invoke(this, m3, new Object[] { paramString });
      return;
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
 
  public final int hashCode() {
    try {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
 
  public final String toString() {
    try {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
 
  static {
    try {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("com.leon.proxy.UserManager").getMethod("addUser", new Class[] { Class.forName("java.lang.String") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException) {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException) {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```
可以看到这个代理类里对于方法的调用都会去调用传入InvocationHandler的invoke方法。所以我们只需要实现这个接口的invoke方法，就可以实现任意被代理类方法的拦截和扩展。最后附上示例代码：
```java
public interface UserManager {
    void addUser(String userName);
}
public class UserManagerImpl implements UserManager {
    @Override
    public void addUser(String userName) {
        System.out.println("Add user: " + userName);
    }
}
public class LogHandler implements InvocationHandler {
    private Object targetObject;
    private Object newProxyInstance(Object targetObject){
        this.targetObject=targetObject;
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(),this);
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        System.out.println("=========start=========");
        Object ret=method.invoke(targetObject, args);
        System.out.println("=========end=========");
        return ret;
    }
    public static void main(String[] args){
        LogHandler logHandler=new LogHandler();
        UserManager userManager=(UserManager)logHandler.newProxyInstance(new UserManagerImpl());
        userManager.addUser("Leon");
    }
}
```
通过实现InvocationHandler接口，便可以对任意实现了接口的类进行代理，如果要对没有实现接口的类进行代理可以使用下面的方法。
### cglib动态代理 ###
通过cglib(Code Generation Library)第三方库来实现的动态代理，它的底层使用ASM([Java bytecode manipulation and analysis framework](https://asm.ow2.io/))利用继承的方法在内存中动态的生成被代理类的子类，解决了jdk动态代理要求被代理类必须实现接口的局限，且运行速度要远远快于jdk动态代理。下面假设UserManagerImpl没有实现接口：
```java
public class UserManagerImpl {
    public void addUser(String userName) {
        System.out.println("Add user: " + userName);
    }
}
```
首先实现一个MethodInterceptor接口，类似于InvocationHandler接口：
```java
class LogInterceptor implements MethodInterceptor{
  ...
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("=========start=========");
        Object ret = proxy.invokeSuper(obj, args);
        System.out.println("=========end=========");
        return ret;
    }
}
```
然后利用cglib的Enhancer来生成代理类：
```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserManagerImpl.class);
enhancer.setCallback(new LogInterceptor());
 
UserManagerImpl userManagerImpl = (UserManagerImpl)enhancer.create();
userManagerImpl.addUser("Leon");
```
这里需要注意，由于是通过继承来实现代理，所以不能代理final类，也不能代理final方法。如果反编译生成的代理类，可以看到：
```java
public class UserManagerImpl$$EnhancerByCGLIB$$e4856e83
  extends UserManagerImpl
  implements Factory
{
  ...
  private MethodInterceptor CGLIB$CALLBACK_0;
  ...
 
  public final String addUser(String userName)
  {
    ...
    MethodInterceptor var3= CGLIB$CALLBACK_0;
    if (var3 != null) {
      return (String)var3.intercept(this, CGLIB$addUser$0$Method, new Object[] {userName}, CGLIB$sayHello$0$Proxy);
    }
    return super.addUser(userName);
  }
  ...
}
```
代理类继承了被代理类并实现了net.sf.cglib.proxy.Factor接口，执行方法是如果有MethodInterceptor就调用其intercept方法，如有没有就调用父类也就是被代理类方法。最后附上MethodInterceptor接口的定义：
```java
/*
 * Copyright 2002,2003 The Apache Software Foundation
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package net.sf.cglib.proxy;
 
/*
 * General-purpose {@link Enhancer} callback which provides for "around advice".
 * @author Juozas Baliuka <a href="mailto:baliuka@mwm.lt">baliuka@mwm.lt</a>
 * @version $Id: MethodInterceptor.java,v 1.8 2004/06/24 21:15:20 herbyderby Exp $
 */
public interface MethodInterceptor
extends Callback
{
    /*
     * All generated proxied methods call this method instead of the original method.
     * The original method may either be invoked by normal reflection using the Method object,
     * or by using the MethodProxy (faster).
     * @param obj "this", the enhanced object
     * @param method intercepted Method
     * @param args argument array; primitive types are wrapped
     * @param proxy used to invoke super (non-intercepted method); may be called
     * as many times as needed
     * @throws Throwable any exception may be thrown; if so, super method will not be invoked
     * @return any value compatible with the signature of the proxied method. Method returning void will ignore this value.
     * @see MethodProxy
     */    
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,
                               MethodProxy proxy) throws Throwable;
}
```
