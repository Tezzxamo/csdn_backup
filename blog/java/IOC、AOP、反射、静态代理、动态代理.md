<!-- TOC -->
* [概要](#概要)
* [IOC：](#ioc)
  * [反射：](#反射)
* [AOP：](#aop)
  * [静态代理：](#静态代理)
  * [动态代理：](#动态代理)
<!-- TOC -->

# 概要

- IOC的意思是控制反转，是一种思想，也是Spring容器的重要概念。

- AOP的意思是面向切面编程，是面向OOP的一个重要补充，主要弥补了OOP在纵向的扩展性不够强，不能很好的解决横向扩展的问题。

# IOC：

```text
创建对象本来是依靠我们手动去创建，但是现在我们可以通过IOC容器来帮我们创建并管理对象，这个过程，就叫做控制反转。

IOC容器负责创建对象、管理对象、依赖注入。

其实现原理是：通过工厂模式结合反射机制和配置解析去实现的。
```

## 反射：

- 核心概念：反射就是在`程序运行时`能够获取一个类的属性和方法，对于一个对象能够调用它的属性和方法的机制。
- 通俗讲解：程序运行的时候，能够获取一个类的属性和方法的机制。
- 示例：
  ```java
  // 运行时得到任意一个类的class对象，并通过这个对象查看这个类的信息。
  Class clz = Class.forName("com.zzx.demo.Student");
  // 得到任意一个类的实例，并访问该对象的成员变量
  Student student = (Student) clz.newInstance();
  System.out.println(student.getName());
  // 生成一个类的动态代理类（后面介绍）
  ```

# AOP：

```text
实现原理：动态代理。
         通过动态代理得到目标对象的代理类，然后就可以通过这个动态代理类，实现对目标对象的增强。
         
常见场景：异常处理、日志记录、事务控制、安全控制等场景。
```

## 静态代理：

`代理模式`

- 代理模式的应用
- 概念：程序运行前，代理类就已经存在代理类的代理方式，是静态代理。
- 描述：静态代理就是实现与目标对象相同的接口，但是成员变量是目标对象，也就是需要将代理的对象通过构造函数注入进来，实现与目标对象相同的接口，然后就可以通过代理对象实现对目标对象的增强、环绕等功能。
- 应用场景：需要对原功能进行扩展的时候，不适合对原有代码进行直接修改，可以通过增加代理类进行功能增加。
- 示例：

```text
接口：UserService:{work()}
实现类：UserServiceImpl:{work()}
代理类：ProxyUserServiceImpl:{成员变量：Person，代理方法：work()}

ProxyUserServiceImpl()方法中，调用成员变量的work()，然后在work()方法中添加增强功能，
```

```java
// 使用示例：
public class ProxyRunTest {
    public static void main(String[] args) {
        UserService userService = new ProxyUserServiceImpl(new UserServiceImpl());
        userService.work();
    }
}
```

## 动态代理：

`代理模式`

- 概念：程序运行时，代理类才被创建出来的代理方式，是动态代理。
- 描述：实现`InvocationHandler`接口，并且实现`invoke()`方法，且在动态代理类中写一个方法`getInstance()`用于获取代理类，
  `getInstance()`的入参是目标类，在动态代理类中通过成员变量来接收这个目标对象。（`getInstance()`非必要）
- 示例：

```java
import lombok.AllArgsConstructor;

import java.lang.reflect.InvocationHandler;

@AllArgsConstructor
public class DynamicProxy implements InvocationHandler {
    private final Object target;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // do something
        System.out.println("do something");
        // 调用目标方法并返回执行结果。
        Object returnValue = method.invoke(target, args);
        // do something
        System.out.println("do something");
        return returnValue;
    }

    public static <T> T getInstance(T targetObj) {
        // 获取目标对象-类加载器
        // 获取目标对象-接口列表
        // 创建InvocationHandler对象（动态代理的执行逻辑）
        return (T) Proxy.newProxyInstance(targetObj.getClass().getClassLoader(), targetObj.getClass().getInterfaces(), new PerformanceInvocationHandler(targetObj));
    }
}
```

```java
public class ProxyRunTest {
    public static void main(String[] args) {
        // 被代理对象=目标对象
        UserServiceImpl target = new UserServiceImpl();
        // 创建一个代理对象（动态代理对象）
        UserService userServiceProxy = DynamicProxy.getInstance(target);
        // 调用方法
        userServiceProxy.work();
    }
}
```













