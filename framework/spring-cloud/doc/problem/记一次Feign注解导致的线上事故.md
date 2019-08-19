### 问题描述
线上的一个服务在一次版本更新中引入了Spring Cloud Feign的注解  `@EnableFeignClients(basePackages = "xx.xx")`, 同时这个启动类监听了上下文刷新事件`ContextRefreshedEvent`对一个延迟加载`@Lazy`的类进行初始化操作，这个被初始化的类的初始化方法又调用了一个持有`ApplicationContext`静态变量的受spring管理的类的静态方法，大概如下面的样子：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "xx.xx.feign")
public class Application implements ApplicationListener<ContextRefreshedEvent> {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        event.getApplicationContext().getBean(LazyService.class);
    }
}
```

```java
@Service
@Lazy
public class LazyServiceImpl implements LazyService {

    @PostConstruct
    public void init() {
        ....
        SpringContext.getBean(xx.class);
        ...
    }
}
```

```java
@Service
public class SpringContext {
    private static ApplicationContext context;
    @Autowired
    public void setApplicationContext(ApplicationContext applicationContext) {
        SpringContext.applicationContext = applicationContext;
    }

    public static Object getBean(String name) {
        return applicationContext.getBean(name);
    }

    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }
}
```
导致启动的时候报NPE，从错误堆栈上来看，就是applicationContext为null引起的。

### 问题分析
* 首先通过错误堆栈可以推断出在接收到上下文刷新事件时`SpringContext`并没有被初始化，此时其持有的context为空，导致调用报NPE。
* 通过代码比对发现跟上次正常的版本的区别只是新加入了`@EnableFeignClients(basePackages = "xx.xx")`，所以很大原因是这个引起的。于是对两个版本的代码debug发现接收到的刷新事件的源是不一样的。没加Feign注解之前的源是`AnnotationConfigEmbeddedWebApplicationContext`，而加了注解之后变成了`AnnotationConfigApplicationContext`，同时发现这两个context初始化的单例bean是不一样的，后面的只是初始化了Feign相关的bean。至此我们应该是找到确切的原因了。
* 那为什么会出现这种问题呢？
原来是，当加入Feign client到上下文之后，spring cloud会创建一个子的上下文来初始化Feign相关的实例，初始化完成之后会发送上下文刷新事件，而此时父上下文并没有初始化完成。 （参考[https://github.com/springfox/springfox/issues/1207](https://github.com/springfox/springfox/issues/1207)）


### 问题解决
* 方案一：修改监听的事件类型为`ApplicationReadyEvent`，确保应用上下文全部加载完毕。
* 方案二：在`LazyServiceImpl`上添加`@DependsOn('springContext')`，让其依赖于`SpringContext`实例（其通过静态方法调用`SpringContext`并不会触发它的加载动作）的初始化。

