# Spring

## spring概述

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring
    * spring boot
        * spring cloud
@endmindmap
]]>
</code-block>

Spring框架 生态 扩展性

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring
    * IOC(控制反转)
        * DI(依赖注入)
    * AOP
@endmindmap
]]>
</code-block>

IOC是一种思想，DI是手段

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* IOC容器
    * 对象bean
@endmindmap
]]>
</code-block>

```Java
 <bean id="" class="" scope depen init-method....>-->
```

用的时候很简单
```Java
ApplicationContext context = new ClassPathXmlApplicationContext("tx.xml");
System.out.println("Spring context loaded");
A a = (A) context.getBean("a");
a.getApplicationContext(); // ApplicationContext是当前容器，获取不到
```

context是如何拿到bean的？

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* 加载xml
    * 解析xml
        * 封装BeanDefinition
            * 实例化
                * 放到容器中
                    * 从容器获取
@endmindmap
]]>
</code-block>

所以容器其实是一个map，key: String,value: BeanDefinition

![spring概述.png](spring概述.png)

Spring bean是不是单例的

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring bean
    * scope
        * singleton(default)
        * prototype
        * request
        * session
@endmindmap
]]>
</code-block>

为什么用反射不用new创建对象：灵活，可以获取所有的注解，构造器，字段...，而new不可以

在容器创建过程中需要动态改变bean的信息咋么办？

如需要设置以下配置文件的value:
```Text
<pre class="code">
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource"&gt;
    <property name="driverClassName" value="${driver}" />
    <property name="url" value="jdbc:${dbname}" />
    </bean>
</pre>
```
<code-block lang="plantuml">
<![CDATA[
@startmindmap
* PostProcessor(后置处理器增强器)
    * BeanFactoryPostProcessor(增强beanDefinition信息)
    * BeanPostProcessor(增强bean信息)
@endmindmap
]]>
</code-block>

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* 创建对象
    * 实例化
        * 在堆中开辟空间
            * 对象的属性值都是默认值
    * 初始化
        * 给属性设置值
            * 填充属性
            * 执行初始化方法
                * init-method
@endmindmap
]]>
</code-block>

public abstract class PlaceholderConfigurerSupport extends PropertyResourceConfigurer
implements BeanNameAware, BeanFactoryAware {

	/** Default placeholder prefix: {@value}. */
	public static final String DEFAULT_PLACEHOLDER_PREFIX = "${";

	/** Default placeholder suffix: {@value}. */
	public static final String DEFAULT_PLACEHOLDER_SUFFIX = "}";


设置Aware接口属性, Aware的作用？

当Spring容器创建的bean对象在进行具体操作的时候，如果需要容器的其他对象，此时可以将对象实现Aware接口，来满足当前的需要

**这个具体的作用得看后续的代码**

BeanPostProcessor.before和BeanPostProcessor.after的作用  AOP

BeanPostProcessor的实现类AbstractAutoProxyCreator
```Java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
		
	
	@Override
	public @Nullable Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}	
		
			@Override
	public @Nullable Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyBeanReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

After里面实现了创建代理
```Java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
```

在不同的阶段需要处理不同的工作 ，应该咋么办？

观察者模式：监听器，监听事件，多播器

## Spring Boot

### 学习重点清单

&gt; 按“先搭骨架、再填血肉、最后做调优”的思路，把 **最常被问、最容易踩坑、最能体现差距** 的知识点整理成一张速查表。  
&gt; ★ 越多越必背，建议先全扫一遍，再有针对性地深入。

---

**骨架层：启动与配置 ★★★★★**

| 专题        | 关键项                                                                                                                                                                                                           |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **自动配置**  | • `spring.factories` / `@AutoConfiguration` 加载顺序&lt;br&gt;• 条件注解：`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`…&lt;br&gt;• 手写一个 `starter` 的 5 步模板（配置属性→自动配置→条件装配→META-INF 注册→测试） |
| **外部化配置** | • 17 种配置源优先级（官网那张表必须背）&lt;br&gt;• `bootstrap.yml` vs `application.yml`（Spring Cloud 上下文）&lt;br&gt;• 多环境 `spring.profiles.group` 一键切换                                                                          |
| **启动流程**  | • `SpringApplication.run` 10 大步：环境→上下文→刷新→回调&lt;br&gt;• `SpringApplicationRunListener` 扩展点&lt;br&gt;• `ApplicationContextInitializer` 使用场景                                                                    |

---

**血肉层：Web、数据、事务、安全**

| 专题                       | 关键项                                                                                                                                                                                                                                                    |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Spring MVC 零配置** ★★★★★ | • `DispatcherServlet` 自动注册规则&lt;br&gt;• 静态资源映射：`webjars`、`static`、`public`、`resources`、`META-INF/resources`&lt;br&gt;• JSON 消息转换器顺序 & 自定义 `HttpMessageConverter`&lt;br&gt;• 统一异常：`@ControllerAdvice` + `@ExceptionHandler` + `ResponseStatusException` |
| **数据访问** ★★★★            | • 数据源自动配置：HikariCP 为什么是默认？&lt;br&gt;• JpaRepositories 与 MyBatis-Starter 共存时的 Bean 冲突解决&lt;br&gt;• `@Transactional` 传播行为 + 自调用失效（AOP 代理）&lt;br&gt;• `spring-boot-starter-data-jpa` 生产禁慎用 `ddl-auto`                                                     |
| **安全** ★★★               | • Spring Security 过滤器链 11 个过滤器顺序&lt;br&gt;• 自定义 `UserDetailsService` + `PasswordEncoder`&lt;br&gt;• 方法级安全：`@EnableGlobalMethodSecurity(prePostEnabled = true)`                                                                                         |
| **验证** ★★                | • JSR-303 与 Spring Validator 混用&lt;br&gt;• 快速失败模式：`spring.jpa.properties.hibernate.validator.fail_fast`                                                                                                                                                |

---

**高级层：监控、扩展、部署、性能**

| 专题              | 关键项                                                                                                                                                           |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **指标与监控** ★★★★  | • Actuator 端点安全加固（只暴露 `health`、`info`、`metrics`）&lt;br&gt;• 自定义 `HealthIndicator`、`@ReadOperation` 自定义端点&lt;br&gt;• Micrometer 三步埋点：`Counter`、`Timer`、`Gauge` |
| **异步与线程池** ★★★★ | • `@Async` 默认线程池坑：`SimpleAsyncTaskExecutor`&lt;br&gt;• 自定义 `ThreadPoolTaskExecutor` + 线程隔离&lt;br&gt;• 事件驱动：`@EventListener` 线程模型                              |
| **缓存** ★★★      | • 多级缓存抽象：`CacheManager` → `RedisCache` → `Caffeine`&lt;br&gt;• 代理顺序：`@Cacheable` 与 `@Transactional` 同方法时的坑                                                    |
| **部署与调优** ★★★   | • 三种形态：`fat-jar`、`war`、`native-image`（GraalVM）&lt;br&gt;• 启动加速：`spring-context-indexer`、`lazy-initialization`、`AOT`&lt;br&gt;• Docker 内存限制与 `-Xms/-Xmx` 不一致问题 |
| **测试** ★★       | • `@SpringBootTest` 的 `webEnvironment` 枚举&lt;br&gt;• 切片测试：`@DataJpaTest`、`@WebMvcTest`&lt;br&gt;• Testcontainers 一键起真实中间件                                     |

---

**2025 新增/易忽视 ★★**

| 特性                       | 说明                                           |
|--------------------------|----------------------------------------------|
| **Boot 3.x 基线**          | Java 17+、Jakarta EE 9（`javax` → `jakarta`）   |
| **AOT + GraalVM Native** | 启动 &lt;50 ms、内存 &lt;80 MB；需声明反射/动态代理/SpEL 提示 |
| **可观测性**                 | Micrometer Tracing 替代 Sleuth；原生 OTLP 导出      |
| **配置绑定收紧**               | Relaxed Binding 弱化，kebab-case 必须严格匹配         |

---

**学习路径（5 步速成）**

1. 官方文档 **“using-boot”** 小节，只看蓝色高亮框。
2. 打开任意 `spring-boot-starter` 的  
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，对照类图看条件注解。
3. 手写一个 **自定义 starter**（配置属性→自动配置→条件装配→注册→测试），跑通即掌握 80% 自动配置精髓。
4. 用 **Actuator + Micrometer** 暴露接口 QPS & 99 线，搭 Grafana 大盘，面试能讲 10 分钟。
5. 用 **SpringBootTest + Testcontainers + JPA** 写 `@DataJpaTest` 连真实 MySQL，理解切片测试与事务回滚。