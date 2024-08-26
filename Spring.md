[toc]

#### 1 spring基础

##### 1.1 什么是spring

springFramework中有很多模块可以协助我们进行开发，

- 比如IOC和AOP的容器
- 很方便的对数据库进行访问（JDBC）
- 声明式的事务
- 支持Servlet的MVC
- 很方便的集成第三方组件、单元测试

##### 1.2 spring springMVC springBoot区别

spring包含很多功能模块，提供IOC依赖注入，springMVC等模块需要依赖该模块
springMVC：提供构建MVC架构的WEB程序功能（model、view、controller，将业务逻辑、数据、显示分离）
springboot可以简化xml文件配置，starter开箱即用（比如使用springboot简化springMVC的配置）

#### 2 IOC

##### 2.1 ioc的理解

IOC控制反转是一种思想，**将手动创建对象的控制权利，交给框架IOC容器来管理** ，原来需要手动new对象，将对象之间的依赖关系交给IOC容器管理（IOC容器可以创建对象、管理依赖、调用对象方法），需要对象时直接将该对象注入，而不需要考虑对象如何被创建出来。IOC容器实际上是一个Map，**在spring中需要通过XML文件来配置Bean，在springBoot中使用注解进行配置bean**

- 获取IOC容器：IOC容器即Context，通过获得SpringContext，再使用getBean方法获得bean[获取Spring的IOC容器的几种方式 - 简书 (jianshu.com)](https://www.jianshu.com/p/61fdaeed784d)（参数为配置文件或者配置类）
- 配置类：推荐使用java注解配置（@Configuration与@Bean）完全代替XML配置(<beans> <bean>)，。@Configuration声明当前类为一个配置类，相当于spring的xml文件，@Bean用在配置类中的方法上将该方法返回的对象将变成IOC容器中的对象。配置类中可以加上@ComponentScan扫描被@Component注解的bean（相当于<beans>中的<context:component-scan base-package="pojo"/>）。
  - spring启动时会自动扫描配置类，进行相关的类加载（扫描启动类所在包及其子包）[SpringBoot启动配置类（二）【@Configuration注解的配置类如何被加载到？】_springboot @configuration根据状态是否加载-CSDN博客](https://blog.csdn.net/fu123123fu/article/details/95762622)

##### 2.2 springbean

bean就是被IOC容器管理的对象，可以通过xml文件，配置类，注解来配置。

- 将类声明为Bean的注解： @Component  @Controller @Service @Repository（Dao层，用于数据库），使用无参构造函数初始化,无论是否使用，都会被通过无参构造函数初始化

- @Component和@Bean的区别？：

  - @bean用于在配置类（被@Configuration注解的类）中的方法上将该方法返回的对象将变成IOC容器中的对象。@Component用于类名
  - @bean 第三方包中引入bean只能用@bean，starter中的都是配置类+@bean的形式
  - @Component需要通过扫包来自动加入IOC容器中

- 类被加上了@Component注解就能被加载到容器中吗？：不能，需要被组件扫描（ComponentScan可以再xml中或@ComponentScan注解中）才行,

  - 组件扫描方式：[Spring的component-scan XML配置和@ComponentScan注解配置_spring中xml配置扫描目录-CSDN博客](https://blog.csdn.net/duke_ding2/article/details/126617024)

    - 注解方式扫描：@ComponentScan被集成在了启动类@SpringBootApplication中，**启动类启动后会去扫包把加了注解的类加到IOC容器中。**

    - xml方式扫描：
      ```
      <beans
      	......
         <context:component-scan base-package="pojo"/>
      </beans>
      ```

      

- DI注入有哪些？：@AutoWired @Resource @Inject，

  - Autowired（spring框架的），按照类型进行匹配（根据接口匹配实现类），当接口有多个实现类时，根据接口类型就无法正确注入对象了，此时根据类名进行匹配（类名首字母小写），在@AutoWire下使用@Qualifier(value="类名")  显示指定实现类名

    ```java
    @Autowired
    @Qualifier(value = "smsServiceImpl1")
    private SmsService smsService;
    ```

  - @Resource
    默认通过name进行匹配，也可以显示的通过类名或接口名@Resource(name = "smsServiceImpl1")匹配。

##### 2.3 bean的作用域有哪些

（在@Bean @Component下使用@Scope(value)进行配置）

- signleton：IOC容器中的实例是唯一的，单例模式
- prototype：每次都会获得信的bean实例，
- request：仅web应用中可用，每次http请求都会产生一个新的bean，仅在当前http request内有效。。。。

##### 2.4 Bean是线程安全的吗？

- 对于prototype来说，每次调用使用的都是新的对象，不存在线程安全的问题。
- 对于singleton来说，是最常用的，是线程不安全的可能存在竞争的问题，但是因为大多数的bean都是无状态的如dao，service（**其没有可变的成员变量，在每一个用户访问的线程栈中相当于单独的实例，没有共享变量**），==如果是有状态的单例Bean，将可变的成员变量保存在ThreadLocal中==

##### 2.5 ==Bean的生命周期==

- 单例对象
  单例对象的生命周期与IOC容器相同
- 多例对象
  - 实例化：IOC容器找到配置文件中的Bean的定义，使用反射调用无参构造函数创建bean的实例
  - 属性赋值：为bean设置相关依赖和属性，如@AutoWire注入对象，@Value注入值，set方法注入，
  - 初始化：
    - **如果实现了相应的Aware接口，调用相应的方法，Aware接口能让Bean拿到Spring容器资源**（让bean向容器表示需要获得某种基础依赖，如bean名称、bean类加载器、bean工厂、配置信息、AppicationContext）
    - **如果实现了BeanPostProcessor接口的Bean，会作用在（被调用）初始化的前后，以实现AOP功能（调用该方法返回值会替换原本的bean作为代理），如果实现InstantiationAwareBeanPostProcessor接口会作用在实例化的前后**
    - 如果实现了init-method方法，执行指定方法
  - 销毁：如果bean不需要了（长期不用且没有对象引用）会被销毁，如果Bean实现了DisposableBean接口或者配置了destory-method（xml中）属性，调用destory方法
    <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240407190417186.png" alt="image-20240407190417186" style="zoom:33%;" />

##### 2.6注入机制（注入对象的属性和依赖）

[Spring中bean的四种注入方式 - 特务依昂 - 博客园 (cnblogs.com)](https://www.cnblogs.com/tuyang1129/p/12873492.html)
[Spring之基于注解的注入 - Mars.wang - 博客园 (cnblogs.com)](https://www.cnblogs.com/wangbin2188/p/9014400.html)
[Spring基础篇：利用注解将外部Properties属性注入到Bean中的方法-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/1202555#:~:text=通过注解的方式 1 1.在Bean上加入@Compoent注解 2 2.定义配置类 3,3.在配置类中使用@PropertySource注解 解析properties 4 4.在Bean上使用@value注入属性 5 4.测试)

- set方法注入，即类一定要为属性提供set方法，IOC容器是通过调用bean的set方法为属性注入值的，在xml文件中，使用property标签进行set注入, name的值为set方法名去掉set后将第一个字符小写的结果
   ```
   <bean id="myCar" class="cn.tewuyiang.pojo.Car">
   	<property name="speed" value="100"/>   
       <property name="price" value="99999.9"/>
   </bean>
   ```

- 构造器注入，通过类的有参构造函数
  ```
  <bean id="myCar" class="cn.tewuyiang.pojo.Car">
      <!-- 通过constructor-arg的name属性，指定构造器参数的名称，为参数赋值 -->
      <constructor-arg name="speed" value="100" />
      <constructor-arg name="price" value="99999.9"/>
  </bean>
  ```

- 静态工厂注入
  ```
  public class SimpleFactory {
      /** 静态工厂，返回一个Car的实例对象*/
      public static Car getCar() {
          return new Car(12345, 5.4321);
      }
  }
  //xml配置
  <bean id="car" class="cn.tewuyiang.factory.SimpleFactory" factory-method="getCar"/>
  ```

- 注解注入
  @AutoWired注入依赖对象，@Value注入基本类型或封装类型（无需使用set）
  <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240407174950078.png" alt="image-20240407174950078" style="zoom:50%;" />

  ```
  @Component
  @PropertySource(value = "classpath:config.properties") //从properties中注入
  public class student {
      @Value("${name}")
      public String name;
      @Value("${age}")
      public int age;
      @Value("#{${map}}")
      public Map<String, Integer> map;
      @Value("#{${list}}")
      public List<String> list;
         
      //省略了get、set和constuct方法，但是这些在程序中是必须有的
  }
  ```

##### 2.7  BeanFactory和Application关系

#### 3 Aop

[Spring AOP - 注解方式使用介绍（长文详解） - 掘金 (juejin.cn)](https://juejin.cn/post/6844903987062243341)

- springAOP依赖于动态代理技术，动态代理在运行时生成代理对象，而不是像静态代理在编译时生成代理对象，允许开发者在运行时指定要代理的接口和行为，在不修改源码的情况下增强方法的功能。当对象有接口时通过JDK的动态代理，当对象没有接口时通过Cglib字节码操作生成子类来代理对象。可以将横切关注点从核心业务的逻辑中分离出来，常用与如日志、缓存、权限检查、性能统计、接口限流、事务管理等。

- 常见的注解：@Aspect用于定义切面，实现具体的代理逻辑。@Pointcut定义切点，用来匹配哪些Joinpoint连接点要被切面增强，切点可以是注解标注的方法（即加注解的地方为切点）也可以为路径表达式，通知@Before、@After、@Around、@AfterReturning、@AfterThrowing通知，即切面在什么时候执行

- 在哪里切？切点（注解、路径来匹配连接点）  什么时候切？通知（before、after）、怎么切？AOP逻辑

  Joinpoint连接点，被拦截到的点，spring中指被动态代理拦截到的目标类的方法

##### 3.1 注解

```java
@Target(ElementType.METHOD) //Target用来控制注解可以用在哪些目标前，ElementType.METHOD表示该注解可以用在方法上 ElementType.PARAMETER表示用在方法的参数前。
@Retention(RetentionPolicy.RUNTIME)//Retention表示该注解生命周期，RUNTIME为jvm加载class文件时之后也存在（可以通过反射获取）。CLASS为被保留在class文件（类加载就没了），JVM加载class后被遗弃（只在源代码中，编译后不可见，无法通过反射获取），为默认。
//自定义注解需要interface关键词，注解可以看作是一种特殊的标记，程序在编译或者运行时可以检测到这些标记而进行一些特殊的处理
@Target({ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented //表明该注解标记的元素可以被Javadoc 或类似的工具文档化
public @interface TestAnnotation{
    //定义final静态属性
    //定义抽象方法
}
//自定义切面类使用@Aspect注解，在注解切点进行AOP。
@Aspect
@Component
public class Log{
    //定义切点 所有在注解@Log修饰的方法都会植入，@Log为自定义注解，也可以为自带的注解
    @Pointcut("@annotation(cn.javaguide.annotation.Log)")
    public void log(){
        
    }
    @Around("log()") //什么时候注入，方法执行前后
    public Object doAround(JoinPonit joinpoint){
        ...
    }
}
```

#### 4 事务

对方法开启事务，则该方法执行的所有sql都会被放到一个事务中，否则该方法执行的每个sql都为一个单独的事务

两种事务管理方式：可以进行编程式事务管理TransactionTemplate、TransactionManager，或者进行声明式事务管理（使用注解@Transactional，通过AOP）。spring提供事务管理器接口PlatformTransactionManager（有获得事务、提交事务、回滚事务三个方法），让各个平台如JDBC自己去实现接口DataSourceTransactionManager。

@Transactional中的参数

- 事务传播行为Propagation：spring在TransactionDefinition接口中规定了7中类型的事务传播行为。解决业务层方法之间相互调用的事务问题。（即被事务修饰的方法被嵌套进另一个方法时事务如何传播）

  - REQUIRED，如果当前没有事务就新建一个事务，如果已经存在事务就加入该事务中。Amethod调用Bmethod，如果A、B都开始事务，A回滚则B也回滚，B回滚则A也回滚，**在同一个事务中try-catch会被感知，不同事务中try-catch不会被感知**

  - REQUIRES_NEW，无论外部是否已经开启事务，都新建一个事务，且事务相互独立，互不干扰（**一个事务try-catch不会被外部事务感知**）。A为REQUIRED，B为REQUIRES_NEW，则A回滚则B不回滚，B对外报错A回滚B回滚，不对外报错B回滚A不回滚

  - NESTED嵌套，如果外部有事务则开启嵌套事务（**作为外部事务的子事务，外部回滚则子事务回滚、子事务回滚，对外异常则外部回滚，内部try-catch异常则外部不回滚**），如果外部没有事务则新建一个单独事务。

    总结：NESTED和REQUIRED一个作为子事务，一个加入外部事务，外部事务回滚则这两个方法的事务都会回滚，而REQUIRED加入外部事务，因此内外有一个出错都会回滚，而NESTED作为外部事务的子事务，自己内部抛出异常回滚外部不会回滚。NESTED和REQUIRE_NEW一个作为子事务一个开启新事务，都可以做到内部回滚而外部不回滚，但是外部回滚NESTED必须回滚，外部回滚REQUIRES_NEW不用回滚。

- 事务隔离级别

  - 默认可重复读。有读未提交、读已提交、可重复读、串行读。

- 事务超时时间

  即一个事务允许执行的最长时间，超过该时间还未执行完则回滚，默认-1，无超时时间。

- 事务回滚规则

  默认情况对于RunTimeException和Error会回滚，遇到Checked检查异常不会回滚，可以通过rollbackfor自定义回滚的异常类型

- 常见配置：propagation、isolation、readonly、timeout、rollbackfor

- 事务注解的原理：@Transactional使用到AOP实现，AOP使用动态代理实现，如果代理的类实现了接口则用JDK的动态代理（通过实现接口生成代理类），否则通过Cglib动态代理（操作字节码生成被代理类的子类），一个类的方法被@Transactional注解的话，spring会在启动时生成一个代理类，调用方法时，实际上调用的是invoke的方法，在目标方法执行前开启事务，执行中有问题则回滚事务，执行结束提交事务。无法在

#### 5 自动装配

[为什么SpringBoot自动装配类要加上@Configuration注解？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/612903567)

#### 6 手撕spring

1. 手写模拟spring
   - 首先spring要有IOC容器，因此生成一个Context类JayContext
   - 模拟通过@Configuration注解的配置类来获取bean，即JayContext的构造函数参数为`配置类.class`，因此还需要写配置类AppConfig
   - 配置类AppConfig需要有组件扫描@ComponentScan扫描被@Component注解的bean
   - 因此需要生成一个@ComponentScan的注解Innotation，该注解的@Target(ElementType.TYPE)只能写在类上面，@Retention(RetentionPolicy.RUNTIME)在jvm加载class文件时之后也存在生效（生命周期），该注解中要有属性（指定扫描路径）
   - 定义bean还需要@Component注解，再生成一个@Component注解，该注解也有属性（指定bean的名字）

2. Spring扫描实现找到扫描路径下写了@Component的类（即找到bean）
   - Spring容器怎么知道自己的扫描路径呢？：context容器的构造函数参数为配置类.class，通过配置类上的注解@ComponentScan的value获得扫描路径。<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240505175424876.png" alt="image-20240505175424876" style="zoom:67%;" />
   - 因此在容器JayContext中的构造函数中要检查配置类是否有@ComponentScan注解类，通过反射获取到该注解类，并得到注解类中的value值（即扫描路径 如com.zhouyu.service），要扫描该包下编译后的.class字节码文件，而不是源代码.java文件

3. 手写模拟beanDefinition的生成

   - beanDefinition定义容器中bean的元数据对象，可以通过getBeanDefinition(beanName)方法来获得beanDefinition，例如bean的类名、作用域、生命周期、依赖关系、配置信息（构造函数、属性）。==**保存了bean的配置和属性，这些信息会告诉spring容器如何创建和配置bean**==。**beanDefinition对象通常由spring容器在启动过程中根据配置信息或注解生成**。根据bean配置的不同来源（如xml、注解、java），beanDefinition也有很多种
     [Spring系列 什么是BeanDefinition（超通俗易懂、超细致）-CSDN博客](https://blog.csdn.net/weixin_45663027/article/details/133842110)
   - 在扫描的过程中，不应该直接将bean创建出来，因为bean分为单例和多例，多例时只有在使用到时才创建bean，且在context中有getBean方法中会根据beanName来返回bean实例，因此需要知道该类bean的配置，而在调用bean时才通过类名找到该类再解析该类是否有scope注解等等配置就太麻烦了，所以在扫描时就需要对每个bean生成beanDefinition保存bean的配置和属性。
   - 因此构造函数扫描的时候先生成beanDefinition对象，再保存到context的beanDefinitionMap中
   - **==构造函数扫描总结==**：扫描生成beanDefinition、实例化单例bean
     - 先获取到配置类中@componentScan对应的class目录，遍历目录中所有的类class对象是否有@Component注解，再解析其中的配置如@Scope并生成对应的beanDefinition对象，保存在容器的beanDefinitionMap中。
     - 遍历beanDefinitionMap的配置，调用getBean方法获取bean实例，将其中singleton单例的bean利用createBean方法生成实例，并生成单例池singletonObjects，将单例实例放入单例池中（key为bean的名字，val为object的bean实例）

4. 手写getBean方法的底层实现

   - 已经扫描过目录生成beanDefinition后，调用getBean方法获取bean实例，首先检查beanDefinitionMap中是否有该beanName的配置，再获取到该bean的配置并根据配置生成bean对象，如单例或者多例（单例就从单例池中获取并返回，如果是单例但是单例池中没有该对象，则创建该对象并放入单例池再返回（即懒加载模式，再生成容器时不创建实例，使用时才创建），多例就每次都创建新的bean实例并返回）

5. 手写bean的创建流程  createBean
   在getBean方法中会调用createBean来创建bean，createBean的参数为beanName和该bean的beanDefinition配置。通过beanDefinition获取到bean的class类，再通过反射 如使用无感构造方法`Obj o=clazz.getConstructor().newInstance()`获取到bean对象

6. 手写模拟依赖注入 

   **获得bean实例的方式除了通过context容器的getBean方法，还可以通过依赖注入，@Autowired注解的方式。**

   - **beanName**：当@Component注解的value为空，即没有手动的为bean命名，则将bean默认命名为类名的首字母小写
   - 写一个@Autowired注解，其@Target为ElementType.FIELD即只能使用在字段上。
   - 自动注入在哪一步做？：==创建一个bean的同时注入依赖==，**先找到加了Autowired注解的属性，再给该属性赋值**，在bean实例创建的时候，将bean其中的依赖注入的对象属性进行赋值。即在createBean方法中（在getBean和生成单例池中会被调用）。在createBean中通过反射获取生成的bean实例的所有字段，遍历字段查看是否被Autowired注解了 `if(f.isAnnotationPresent(Autowired.class))` ，反射f.set(属性所在对象,属性值)，属性值为getBean(f.getName)
     <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240509231603929.png" alt="image-20240509231603929" style="zoom:50%;" />
   - 注入时，先byType后byName，byType找不到对应的bean时再通过beanName找。
   - **==getBean==**：生成bean对象，注入依赖，初始化bean对象（执行Aware回调函数设置bean的属性，再执行init方法与InitializingBean接口的方法）

7. Aware回调机制

   实现了Aware接口的bean具有被spring容器通知的能力（**通过回调的方式调用实现了Aware子接口的set方法将相应的参数给bean的属性**），spring容器再生成了bean对象，注入依赖后，会回调bean中实现的Aware接口的方法。Aware为一个空接口，具体回调什么方法由其子接口来确定，**即bean实现的Aware的子接口的方法会在bean生成并注入依赖后被spring容器回调**
   [【死磕 Spring】----- IOC 之 深入分析 Aware 接口-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1380827)

   - 如BeanNameAware，bean类实现BeanNameAware接口与setBeanName方法，spring容器会在生成bean实例与注入完依赖后调用bean对象实现该接口的setBeanName(String var1)方法，将beanName返回给bean对象。

8. 手写模拟spring初始化机制
   spring容器的getBean方法，除了生成bean实例，注入依赖，回调Aware接口以外，还会完成bean的初始化方法（实现InitializingBean接口或者配置Init方法）。

   - spring提供了InitializingBean接口，为bean提供了属性初始化的处理方法。该接口只有afterPropertiesSet方法，spring会自动为实现该接口的bean在Aware回调后执行该方法

     **执行顺序优先级：构造方法 > postConstruct >afterPropertiesSet > init方法。**
     [Spring中的InitializingBean的使用详解_initializingbean的作用-CSDN博客](https://blog.csdn.net/TreeShu321/article/details/108180366)

9. 手写BeanPostProcessor机制

   **可以在初始化前后更灵活的处理bean对象**
   bena实例初始化后，就要考虑bean的AOP了。首先需要了解BeanPostProcessor接口即bean的后置处理器，利用该机制可以对bean的创建过程进行控制，该接口提供两个方法，postProcessBeforeInitialization和postProcessAfterInitialization。要将BeanPostProcessor写为一个bean（即实现该接口并加上@Component注解）重写两个方法，spring扫描时就能扫描到这个实现了BeanPostProcessor接口的bean，在初始化前后执行这两个方法。**通常用于处理所有bean的通用逻辑，而不是在方法中指定特定的bean处理**

   这两个方法的参数为：beanName，bean对象
   [谈谈Spring中的BeanPostProcessor接口 - 特务依昂 - 博客园 (cnblogs.com)](https://www.cnblogs.com/tuyang1129/p/12866484.html)
   <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240510013136840.png" alt="image-20240510013136840" style="zoom:50%;" />

   - 在扫描目录生成beanDefinition时，将实现了BeanPostProcessor接口的bean，生成实例，并在context容器中创建一个BeanPostProcessor实例的ArrayList并保存到其中，之后在遍历beanDefinitionMap调用getbean时，会在成生成bean实例，依赖注入，Aware回调，之后遍历ArrayList并调用每个BeanPostProcessor的postProcessBeforeInitialization方法，之后进行初始化，再调用postProcessAfterInitialization方法。

10. 手写模拟AOP机制
    若需要对bean进行AOP，即需要context返回的bean就是已经动态代理过的代理对象。可以写在postProcessAfterInitialization中，返回代理对象

    - 实现原理：cglib代理对象会继承代理类，并且在代理类中有个属性为被代理类的对象，执行代理方法时先执行切面类的方法，再执行原被代理类的对象的方法。从而实现动态代理

​		







































