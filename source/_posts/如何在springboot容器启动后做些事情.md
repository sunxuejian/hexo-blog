title: 如何在springboot容器启动后做些事情
date: 2018-10-21 10:57:31
tags:
  - SpringBoot
  - Bean初始化顺序
---

通常有些事情是需要在整个spring容器完全构建之后让程序自动调用处理，有些人喜欢用 <font color=red> `@PostConstruct` </font> 来作为程序启动时加载我觉得是不合理的，在spring容器的装载过程如果出现了问题，我们需要适当的让他自己停止整个程序，而使用  <font color=red> `@PostConstruct` </font> 有时候并不能带来好的效果，接下来我们举出几种方法，可以我们在整个spring容器正常构建完之后，做出一些我们自己的处理

<!-- more -->

# `@PostConstruct` (类构造方法执行后调用)

---

> 从Java EE5规范开始，Servlet增加了两个影响Servlet生命周期的注解（Annotation）:这里我们只介绍<font color=red> `@PostConstruct` </font> ，这个注解被用来修饰一个非静态的void()方法.而且这个方法不能有抛出异常声明，被 <font color=red> `@PostConstruct` </font> 修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Serclet的inti()方法。被<font color=red> `@PostConstruct` </font> 修饰的方法会在构造函数之后，init()方法之前运行。

@PostConstruct 确实能做到在依赖注入完毕并且构造方法完毕后能自动被执行，但是如果你的程序中有很多除需要使用的 `@PostConstruct` ,并且要保证他们的执行顺序这就比较麻烦了,

``` java
@Slf4j
@Component
public class MqMock {

    @PostConstruct
    public void init() {
        log.info(" MqMock initialized ....");
    }
}
```

```java
@Slf4j
@Component
public class Config {

    @PostConstruct
    public void init(){
        log.info("Config initialized ...");
    }
}
```
<font color=red>执行结果:</font>

![日志](/image/springboot/postlog.jpg)

从日志看来，这两个方法被 <font color=red> `@PostConstruct` </font> 修饰过后都被自动执行了，但是他们只是在当前bean被构建完之后调用的，不是容器彻底构建完，而且顺序也无法保证，当然你可以通过 <font color=red> `@Order` </font> 来控制类的构造顺序，这里我详细提了，**是否是需要使用<font color=red> `@PostConstruct` </font> 这个就看你们的需求了**.

 ---

# `CommandLineRunner`

---

[官方文档介绍](https://docs.spring.io/spring-boot/docs/2.0.6.RELEASE/reference/htmlsingle/#boot-features-command-line-runner)
引用一段介绍：
> If you need to run some specific code once the SpringApplication has started, you can implement the ApplicationRunner or CommandLineRunner interfaces. Both interfaces work in the same way and offer a single run method, which is called just before SpringApplication.run(…​) completes.

大概意思就是如果你想要在spring应用启动完成后执行一些代码，你可以实现 <font color=red>ApplicationRunner</font>接口，或者<font color=red>`CommandLineRunner`</font>接口，这两个接口的作用是一致的。

还是刚刚那个例子这回我们在<font color=red>CommandLineRunner</font>接口方法中调用，这回可以指定顺序了，你可以随心所欲把他放在哪一行执行
```java
@Slf4j
@SpringBootApplication
public class BootDemoApplication implements CommandLineRunner {
    @Autowired
    private MqMock mqMock;

    @Autowired
    private Config config;

    public static void main(String[] args) {
        SpringApplication.run(BootDemoApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        log.info("before ...");
        mqMock.init();
        log.info("after ...");
        config.init();
    }
}

```

<font color=red>执行结果</font>：

![日志](/image/springboot/started.png)

从日志中分析，很明显能看到springboot已经执行完毕，这是后我们只需要把我们的类注入过来，进行方法调用，就完事了，是不是很简单，而且这个过程是可控制的。

---

#  `ApplicationStartedEvent`

---

在springboot构建过程中它会调用一组监听器，来传递给不同的处理器来进行各自的处理，关于springboot的监听器机制，我会在另一篇文档详细说明，这里我们就还是以我们当前文章的主题进行。我们这里使用的是`ApplicationStartedEvent`来做到和 <font color=red> `CommandLineRunner` </font> 等同的效果

>  `ApplicationStartedEvent`是springboot整个容器构建过程最后一个事件，所以我们只需要监听这个事件，就能达到和CommandLineRunner一样的效果，


```java
@Slf4j
public class MethodControl implements ApplicationListener<ApplicationStartedEvent> {

    @Override
    public void onApplicationEvent(ApplicationStartedEvent applicationStartedEvent) {
        ConfigurableApplicationContext context = applicationStartedEvent.getApplicationContext();
        MqMock mqMock = context.getBean(MqMock.class);
        Config config = context.getBean(Config.class);
        log.info("测试springboot started event之后启动");
        log.info("before ...");
        mqMock.init();
        log.info("after ...");
        config.init();
    }
}
```

<font color=red>特别注意：我们在实现监听器后必须要给他注册到springboot启动程序中，这个动作是靠注解做不到的，下面我们举例两种方式
1. 在resource目下创建 <font color=red> `META-INF` </font> 目录，并在该目录下创建 <font color=red> `spring.factories` </font>文件

![创建目录](/image/springboot/metainf.png)

<font color=red> spring.factories内容</font>：

![配置文件](/image/springboot/ies.png)

2. 在启动方法中添加监听器

```java
@Slf4j
@SpringBootApplication
public class BootDemoApplication {

    public static void main(String[] args) {
        SpringApplication
                .run(BootDemoApplication.class, args)
                .addApplicationListener(new MethodControl());
    }
}
```

> 以上方式 任选其一到即可

<font color=red>好了，我们配置完监听器后来看一下效果</font>：

![日志输出](/image/springboot/log.png)

好了，到这我们三种方式都已经结束了，更推荐<font color=red> `CommandLineRunner` </font>，如果你需要获取到整个上下文的信息，可能监听器更适合你。

---

>  [数据库版本控制工具(Flyway)，支持程序升级(spring-boot),maven命令操作](https://github.com/sunxuejian/Springboot-plugins/tree/master/db-manager)

> 有什么疑问联系我：

<center>![Image 微信](/image/wechar.jpg)</center>
