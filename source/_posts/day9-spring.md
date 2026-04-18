---
title: 【学习打卡】Day 9 — Spring 框架核心知识点
date: 2026-04-14 12:00:00
tags:
  - Spring
  - Java后端
  - 框架
categories:
  - 学习打卡
---

# 【学习打卡】Day 9 — Spring 框架核心知识点

> 作者：初魁 | 学习周期：Java后端面试冲刺第9天 | 适合人群：Java后端求职者

---

## 一、Spring 框架

### 1. 单例 Bean 的线程安全问题

**结论**：Spring 框架中的单例 Bean **不是线程安全的**。

**原因**：
- Spring 框架没有对单例 Bean 进行任何多线程的封装处理
- 多用户并发请求时，容器给每个请求分配一个线程，并发执行对应业务逻辑（成员方法）
- 如果业务逻辑中有对单例 Bean 状态的修改（成员属性修改），则必须考虑线程同步问题

**实际情况**：项目中常用的 Service、DAO 类通常没有可变状态，因此在某种程度上是线程安全的。如果 Bean 有多种状态（如 ViewModel 对象），需要开发者自行保证线程安全。

**解决方案**：将多态 Bean 的作用域由 singleton 变更为 prototype。

---

### 2. AOP（面向切面编程）

**定义**：将与业务无关，但对多个对象产生影响的公共行为和逻辑，抽取并封装为一个可重用的模块，称为"切面"（Aspect）。

**目的**：减少重复代码，降低模块间耦合度，提高系统可维护性。

**常见应用场景**：记录操作日志、缓存处理、Spring 内置的事务处理。

**核心思路**：
- 使用 AOP 中的环绕通知（`@Around`）和切点表达式定位需要记录日志的方法
- 通过环绕通知的参数（`ProceedingJoinPoint`）获取请求方法的参数
- 获取请求的用户名、请求方式、访问地址、模块名称、登录 IP、操作时间等信息
- 将信息保存到数据库的日志表中

---

### 3. Spring 事务实现

**实现方式**：

- **编程式事务管理**：使用 `TransactionTemplate`，对业务代码有侵入性，项目中很少使用
- **声明式事务管理（常用）**：建立在 AOP 之上。通过 AOP 功能对方法前后进行拦截，在执行方法之前开启事务，执行完目标方法之后根据执行情况提交或回滚事务

**事务失效场景及解决**：

| 失效场景 | 原因 | 解决方案 |
|----------|------|----------|
| 异常被 catch | 事务通知只有捕获到目标抛出的异常才能回滚 | 在 catch 块中手动抛出运行时异常 `throw new RuntimeException(e)` |
| 抛出检查异常 | Spring 默认只回滚非检查异常（RuntimeException 子类）| 配置 `@Transactional(rollbackFor = Exception.class)` |
| 非 public 方法 | Spring 为方法创建代理、添加事务通知的前提是该方法是 public 的 | 将方法改为 public |

---

### 4. Bean 的生命周期

1. 通过 `BeanDefinition` 获取 Bean 的定义信息（封装了类名、属性值、作用域、初始化方法等）
2. 调用构造函数实例化 Bean
3. Bean 的依赖注入（属性赋值）
4. 处理 Aware 接口（如 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`）
5. Bean 的后置处理器（`BeanPostProcessor`）- 前置处理
6. 初始化方法（执行 `InitializingBean` 接口的 `afterPropertiesSet` 方法或自定义的 `init-method`）
7. Bean 的后置处理器（`BeanPostProcessor`）- 后置处理
8. 销毁 Bean（执行 `DisposableBean` 接口的 `destroy` 方法或自定义的 `destroy-method`）

---

### 5. 循环依赖与三级缓存

**循环依赖**：两个或两个以上的 Bean 互相持有对方，最终形成闭环（如 A 依赖 B，B 依赖 A）。

**三级缓存解决机制**：

| 缓存 | 源码名称 | 作用 |
|------|----------|------|
| 一级缓存 | SingletonObjects | 单例池，缓存已经历完整生命周期、初始化完成的 Bean 对象 |
| 二级缓存 | earlySingletonObjects | 缓存早期的 Bean 对象（生命周期还没走完）|
| 三级缓存 | SingletonFactories | 缓存的是 ObjectFactory 对象工厂，用来创建某个对象 |

**解决流程**（以 A、B 相互依赖为例）：

1. 实例化 A（半成品），将 A 的 ObjectFactory 放入三级缓存
2. 初始化 A，发现需要注入 B，但 B 不存在
3. 实例化 B（半成品），将 B 的 ObjectFactory 放入三级缓存
4. 初始化 B，发现需要注入 A，从三级缓存中获取 A 的 ObjectFactory，创建 A 的代理对象（半成品），注入给 B，并将 A 的代理对象放入二级缓存
5. B 创建成功，放入一级缓存
6. 将 B 注入给 A，A 创建成功，放入一级缓存

**构造方法循环依赖**：Spring 无法解决构造函数的循环依赖（因为构造函数是 Bean 生命周期的第一步）。解决方案：使用 `@Lazy` 注解进行懒加载。

---

## 二、SpringMVC 执行流程

### 版本1：视图阶段（JSP）

1. 用户发送请求到前端控制器 `DispatcherServlet`
2. `DispatcherServlet` 调用 `HandlerMapping`（处理器映射器）
3. `HandlerMapping` 找到具体的处理器，生成处理器对象及处理器拦截器（如果有），一起返回给 `DispatcherServlet`
4. `DispatcherServlet` 调用 `HandlerAdapter`（处理器适配器）
5. `HandlerAdapter` 经过适配调用具体的处理器（`Handler/Controller`）
6. `Controller` 执行完成返回 `ModelAndView` 对象
7. `HandlerAdapter` 将 Controller 执行结果 `ModelAndView` 返回给 `DispatcherServlet`
8. `DispatcherServlet` 将 `ModelAndView` 传给 `ViewResolver`（视图解析器）
9. `ViewResolver` 解析后返回具体的 `View`（视图）
10. `DispatcherServlet` 根据 `View` 进行渲染视图（将模型数据填充至视图中）
11. `DispatcherServlet` 响应用户

### 版本2：前后端分离阶段（接口开发，异步请求）

前 6 步相同，第 7 步起变为：

7. 如果方法上添加了 `@ResponseBody` 注解，则通过 `HttpMessageConverter` 将返回结果转换为 JSON 并响应给客户端

---

## 三、SpringBoot 自动配置原理

**核心注解**：`@SpringBootApplication`，封装了三个注解：

- `@SpringBootConfiguration`：声明当前类也是一个配置类（与 `@Configuration` 作用相同）
- `@ComponentScan`：组件扫描，默认扫描当前引导类所在包及其子包
- `@EnableAutoConfiguration`：SpringBoot 实现自动化配置的核心注解

**自动配置核心流程**：

`@EnableAutoConfiguration` 通过 `@Import` 注解导入了 `AutoConfigurationImportSelector` 类 → 读取项目及所引用 Jar 包的 classpath 路径下 `META-INF/spring.factories` 文件中配置的所有类的全类名 → 这些配置类中定义的 Bean 会根据条件注解（如 `@ConditionalOnClass`）所指定的条件来决定是否需要导入到 Spring 容器中

---

## 四、MyBatis

### 1. 执行流程

1. 读取 MyBatis 配置文件：`mybatis-config.xml`，加载运行环境和映射文件
2. 构造会话工厂：`SqlSessionFactory`
3. 创建会话：`SqlSessionFactory` 创建 `SqlSession` 对象（包含了执行 SQL 语句的所有方法）
4. 操作数据库：通过 `Executor` 执行器接口执行 SQL，同时负责查询缓存的维护
5. 封装映射信息：`Executor` 接口的执行方法中有一个 `MappedStatement` 类型的参数，封装了映射信息
6. 输入参数映射
7. 输出结果映射

### 2. 延迟加载

**定义**：在需要用到数据时才进行加载，不需要用到数据时就不加载数据。默认关闭。

**开启**：在 MyBatis 配置文件中配置 `lazyLoadingEnabled=true`。

**原理**：使用 CGLIB 创建目标对象的代理对象。当调用目标方法时，进入拦截器 `invoke` 方法，发现目标属性为 null，执行 SQL 查询关联数据，将结果设置到目标属性中。

### 3. 一级缓存与二级缓存

- **一级缓存**：作用域 `SqlSession` 级别，默认开启。当 Session 进行 flush 或 close 之后，该 Session 中的所有缓存将清空。
- **二级缓存**：作用域 `namespace` 和 mapper 的作用域，不依赖于 `SqlSession`。默认关闭（需在全局配置文件中开启 + 在 mapper 中使用 `<cache/>` 标签）。需要进行新增、修改、删除操作后，默认该作用域下所有 select 中的缓存将被清除。

---

> 说明：本笔记基于《04-框架篇.pptx》内容整理，涵盖 Spring、SpringMVC、SpringBoot、MyBatis 等主流 Java 框架的核心概念与运行机制。
