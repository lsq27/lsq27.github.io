---
title: spring AOP 真正代理的是 TargetSource
date: 2023-07-18 17:56:33
tags: [spring, TargetSource]
categories: [读源码]
---

我们在使用 spring AOP 功能时，可能认为 spring 为 bean 生成代理对象，其实 spring 是为 TargetSource 生成代理对象，利用 TargetSource，可以方便的修改 AOP 真正执行的对象，偷天换日。

<!-- more -->

TargetSource 顾名思义就是 target 的来源，这里的 target 指的是 AOP 执行过程中被反射调用方法的目标。
在 spring 的 AOP 执行过程中，TargetSource 用于获取反射的执行目标 target。如果一个 TargetSource，每次返回同一个 target，在 spring 中称其为静态（static），如果每次返回的不是同一个 target，具体的使用场景有 pooling 和 hot swapping。

TargetSource 这一抽象让 AOP 变得更加灵活，如果没有它，AOP 只能代理一个对象，每次执行只能调用同一个对象的方法。
![TargetSource](images/TargetSource.png)

spring 中最常用的是 SingletonTargetSource，该类只对 target 进行简单的封装，没有特殊的逻辑。spring AOP 框架默认为需要进行增强的类创建 SingletonTargetSource 实例，所以在日常使用 spring 中我们并不能感觉到 TargetSource 的存在。手动使用 SingletonTargetSource 示例如下所示，可以看出使用 SingletonTargetSource 和直接代理 testBean 在效果上并无不同。

```java
@Test
void testSingletonTargetSource() {
    TestBean testBean = new TestBean("testBean");
    TargetSource targetSource = new SingletonTargetSource(testBean);
    TestBean proxy = (TestBean) ProxyFactory.getProxy(targetSource);
    assertThat(testBean.getName()).isSameAs(proxy.getName());
    assertThat(testBean).isNotEqualTo(proxy).isNotSameAs(proxy);
}
```

TargetSource 接口具体定义及如下

```java
public interface TargetSource extends TargetClassAware {

    /**
     * 返回 target 的类型
     */
    @Override
    @Nullable
    Class<?> getTargetClass();

    /**
     * 如果 getTarget() 方法每次返回相同的对象，则返回true。
     * 此时不必调用 releaseTarget(Object) 释放对象且 AOP 框架可以对 getTarget() 的结果进行缓存
     */
    boolean isStatic();

    /**
     * 返回 target，用于 AOP framework 反射调用 target 执行方法前
     */
    @Nullable
    Object getTarget() throws Exception;

    /**
     * 释放获取的 target
     */
    void releaseTarget(Object target) throws Exception;

}
```
