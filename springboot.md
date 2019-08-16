### 什么是springboot

对于 spring 框架，我们接触得比较多的应该是 spring mvc、 和 spring。而 spring 的核心在于 IOC(控制反转)和 DI (依赖注入)。而这些框架在使用的过程中会需要配置大量 的 xml，或者需要做很多繁琐的配置。
springboot 框架是为了能够帮助使用 spring 框架的开发 者快速高效的构建一个基于 spirng 框架以及 spring 生态 体系的应用解决方案。它是对“约定优于配置”这个理念下 的一个最佳实践。因此它是一个服务于框架的框架，服务 的范围是简化配置文件。

### 约定优于配置的体现

> 1、maven的目录结构：默认有resource文件夹存放配置文件、默认打包方式为jar
>
> 2、spring-boot-starter-web中默认包含spring MVC 相关的依赖以及内置的Tomcat容器，使得构建一个web应用更加简单
>
> 3、默认提供application.properties/yml文件
>
> 4、默认通过spring.profiles.active 属性来决定运行环境时读取的配置文件
>
> 5、@EnableAutoConfiguration默认对于依赖的start进行自动装载

### @SpringBootApplication

```java
@SpringBootApplication
public class SpringbootFirstApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringbootFirstApplication.class, args);
    }
}
```

从启动类的注解@SpringBootApplication开始讲解，揭开springboot的奥秘。

```java
// 复合注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration // 点进去，实际上是@Configuration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

本质上是三个注解组成：@Configuration、@EnableAutoConfiguration、@ComponentScan

#### @Configuration

它是javaconfig的形式基于springIOC容器的配置类使用的一种注解。因 为 SpringBoot 本质上就是一个 spring 应用，所以通过这 个注解来加载 IOC 容器的配置是很正常的。所以在启动类 里面标注了@Configuration，意味着它其实也是一个 IoC 容器的配置类。

传统意义上的 spring 应用都是基于 xml 形式来配置 bean 的依赖关系。然后通过 spring 容器在启动的时候，把 bean 进行初始化并且，如果 bean 之间存在依赖关系，则分析这 些已经在 IoC 容器中的 bean 根据依赖关系进行组装。 直到 Java5 中，引入了 Annotations 这个特性，Spring 框 架也紧随大流并且推出了基于 Java 代码和 Annotation 元 信息的依赖关系绑定描述的方式。也就是 JavaConfig。从 spring3 开始，spring 就支持了两种 bean 的配置方式， 一种是基于 xml 文件方式、另一种就是 JavaConfig 任何一个标注了@Configuration 的 Java 类定义都是一个 JavaConfig 配置类。而在这个配置类中，任何标注了 @Bean 的方法，它的返回值都会作为 Bean 定义注册到 Spring 的 IOC 容器，方法名默认成为这个 bean 的 id。

```java
@Configuration
public class ConfigurationDemo {
    @Bean
    public FirstDemo firstDemo() {
        return new FirstDemo();
    }
}
```

#### @ComponentScan

ComponentScan 这个注解是大家接触得最多的了，相当于 xml 配置文件中的<context:component-scan>。 它的主要作用就是扫描指定路径下的标识了需要装配的类，自 动装配到 spring 的 Ioc 容器中。 标识需要装配的类的形式主要是:@Component、 @Repository、@Service、@Controller 这类的注解标识的 类。ComponentScan 默认会扫描当前 package 下的的所有加 了相关注解标识的类到 IoC 容器中;

#### @EnableAutoConfiguration

对于springboot来说意义重大。EnableAutoConfiguration 的 主 要 作 用 其 实 就 是 帮 助 springboot 应用把所有**符合条件**的**@Configuration 配置** 都**加载到当前 SpringBoot 创建并使用的 IoC 容器中。**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
// 下面会提到import注解
// 下面会提到AutoConfigurationImportSelector
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
   String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
   Class<?>[] exclude() default {};
   String[] excludeName() default {};

}
```

##### @Import

import 注解是什么意思呢? 联想到 xml 形式下有一个 <import resource/> 形式的注解，就明白它的作用了。 import 就是把多个分来的容器配置合并在一个配置中。在 JavaConfig 中所表达的意义是一样的。

```java
@Configuration
@Import(ConfigurationSecondDemo.class)
public class ConfigurationDemo {
    @Bean
    public FirstDemo firstDemo() {
        return new FirstDemo();
    }
}
```

```java
@Configuration
public class ConfigurationSecondDemo {

    @Bean
    public SecondDemo secondDemo() {
        return new SecondDemo();
    }

}
```

```java
public static void main(String[] args) {

    AnnotationConfigApplicationContext applicationContext =
            new AnnotationConfigApplicationContext(ConfigurationDemo.class);
    String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
    for (int i = 0; i < beanDefinitionNames.length; i++) {
        System.out.println(beanDefinitionNames[i]); //输出：secondDemo firstDemo
    }

}
```

##### AutoConfigurationImportSelector

从名字来看，可以猜到它是基于ImportSelector来实现**基于动态bean的加载功能，**之前我们讲过 Springboot @Enable*注解的工作原理 ImportSelector 接口 selectImports 返回的数组(类的全类名)都会被纳入到 spring 容器中。

###### 简单的一个小demo

```java
// 随便定义一个
public class CacheService {
}
```

```java
public class CacheImportSelector implements ImportSelector {
    
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
      // 在这边可以添加任何逻辑，实现动态加载bean的功能。  
      return new String[]{CacheService.class.getName()};// 返回的数组(类的全类名)都会被纳入到 spring 容器中。
    }
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(CacheImportSelector.class) // 定一个注解，仿造@EnableAutoConfiguration
public @interface EnableDefineService {
    Class<?>[] exclude() default {};
}
```

```java
@SpringBootApplication
@EnableDefineService // 加上这个注解
public class EnableMain {
    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(EnableMain.class, args);
        System.out.println(run.getBean(CacheService.class));// com.xavier.springboot.first.springbootfirst.third.CacheService@11963225
    }
}
```

###### 源码分析

定位到AutoConfigurationImportSelector中的selectImports方法，本质上来说，其实 EnableAutoConfiguration 会帮助 springboot 应用把所有符合@Configuration 配置都加载 到当前 SpringBoot 创建的 IoC 容器，而这里面借助了 Spring 框架提供的一个工具类 SpringFactoriesLoader 的 支持。以及用到了 Spring 提供的条件注解 @Conditional，选择性的针对需要加载的 bean 进行条件 过滤。

**SpringFactoriesLoader**

其实和javaSPI机制的原理是一样的，不过它比SPI更好的点在于不会一次性加载所有的类，而是根据key进行加载，从classpath/META-INF/spring.factories文件中，根据key来加载对应的类到spring IOC容器中。

**深入理解条件过滤**

在分析AutoConfigurationImportSelector源码的时候，会先扫描spring-autoconfigure-metadata.properties

![image-20190815144353685](http://ww4.sinaimg.cn/large/006tNc79ly1g61pm07n63j30vi09cagf.jpg)

```java
#Tue Aug 06 08:05:49 GMT 2019
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.data.neo4j.Neo4jBookmarkManagementConfiguration.Configuration=
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration=
org.springframework.boot.autoconfigure.kafka.KafkaAnnotationDrivenConfiguration.Configuration=
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration.ConditionalOnClass=org.influxdb.InfluxDB
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration=
org.springframework.boot.autoconfigure.session.SessionRepositoryFilterConfiguration.Configuration=
org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration=
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration.AutoConfigureBefore=org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.freemarker.FreeMarkerReactiveWebConfiguration.Configuration=
org.springframework.boot.autoconfigure.reactor.core.ReactorCoreAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration.ConditionalOnClass=javax.sql.DataSource,org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration=
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration=
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration.ConditionalOnBean=com.mongodb.reactivestreams.client.MongoClient
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration.ConditionalOnClass=com.hazelcast.core.HazelcastInstance,org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean
org.springframework.boot.autoconfigure.session.MongoReactiveSessionConfiguration.Configuration=
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration.ConditionalOnSingleCandidate=org.springframework.mail.javamail.JavaMailSenderImpl
org.springframework.boot.autoconfigure.session.RedisSessionConfiguration.ConditionalOnClass=org.springframework.data.redis.core.RedisTemplate,org.springframework.session.data.redis.RedisOperationsSessionRepository
............
```

最后在扫描spring.factories对应的类时，会结合前面的元数据进行过滤，为什么要过滤呢？原因是很多的@Configuration其实是依托于其他的框架来加载的，如果当前的classpath环境下没有相关的依赖，则意味着这些类没必要进行加载，所以，通过这种条件过滤可以有效的减少@Configuration类的数量从而降低springboot的启动时间。比如：

```java
// spring.factories文件
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
.....
```

```java
// spring-autoconfigure-metadata.properties文件
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration.ConditionalOnClass=
  com.mongodb.MongoClient
```

MongoDataAutoConfiguration这个@Configuration如果当前classpath下没有MongoClient这个依赖，MongoDataAutoConfiguration不会被加载到IOC容器中。

###### 简单SPI自己实现的demo

项目Xavier-core，代码如下：对此项目进行install打包

```java
public class XavierCore {
    public void say(){
        System.out.println("say hello");
    }
}
```

```java
@Configuration
public class XavierCoreConfig {
    @Bean
    public XavierCore xavierCore(){
        return new XavierCore();
    }
}
```

项目springboot-first依赖于Xavier-core

```java
<dependency>
    <groupId>com.xavier.springboot.core</groupId>
    <artifactId>xavier-core</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

```java
// 启动项目
@SpringBootApplication
public class SpringbootFourApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(SpringbootFourApplication.class, args);
        run.getBean(XavierCore.class).say();
    }
}
```

报错如下：

Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type '**com.xavier.springboot.core.xaviercore.XavierCore**' available

在项目Xavier-core的resources目录加上META-INF/spring.factories文件

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.xavier.springboot.core.xaviercore.XavierCoreConfig
```

再启动SpringbootFourApplication就成功了！

spi实现了，继续测试spring-autoconfigure-metadata.properties配置文件的作用

在项目Xavier-core的resources目录加上META-INF/spring-autoconfigure-metadata.properties

```java
// 条件化bean：当前classpath下必须存在TestClass，才会加载XavierCoreConfig
com.xavier.springboot.core.xaviercore.XavierCoreConfig.ConditionalOnClass=
  com.xavier.springboot.first.springbootfirst.TestClass
```

报了同样的错误如下：

Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type '**com.xavier.springboot.core.xaviercore.XavierCore**' available

在项目springboot-first上加一个类：TestClass。就成功了！。。。。。