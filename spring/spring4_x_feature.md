#spring4 新特性介绍
##spring4.0.x新特性介绍
####spring4--Improved Getting Started Experience
 改进spring.io 网站中spring Getting Started的一些使用方式，方便大家学习和使用spring。
####spring4--Removed Deprecated Packages and Methods
  remove一些废弃的接口和方法 具体废弃的接口和方法请[点击该链接](http://docs.spring.io/spring-framework/docs/3.2.4.RELEASE_to_4.0.0.RELEASE/)
####spring4--Java 8 (as well as 6 and 7)
1.spring 提供对java 8的支持，可以使用lambda expressions and method references ,说到lambda expressions 大家可能都知道，而method references大致是这样的
```java
import java.util.List;
import java.util.ArrayList;

class Zoo {
    private List animalList;
    public Zoo(List animalList) {
        this.animalList = animalList;
		System.out.println("Zoo created.");
    }
}

interface ZooFactory {
    Zoo getZoo(List animals);
}

public class ConstructorReferenceExample {

    public static void main(String[] args) {
		//following commented line is lambda expression equivalent
		//ZooFactory zooFactory = (List animalList)-> {return new Zoo(animalList);};
		ZooFactory zooFactory = Zoo::new;
		System.out.println("Ok");		
		Zoo zoo = zooFactory.getZoo(new ArrayList());
    }
}
```
如果还没有看明白的话，可以百度下method references大致看下具体的讲解和实现过程

2.其次增加了对@Repeatable注解的支持，这个注解主要是用来让注解可以重复使用，如下两个列子
```java
//没有使用@Repeatable注解，采用两个注解来实现能够使用多个注解的方式
public @interface Authority {
     String role();
}

public @interface Authorities {
    Authority[] value();
}

public class RepeatAnnotationUseOldVersion {

    @Authorities({@Authority(role="Admin"),@Authority(role="Manager")})
    public void doSomeThing(){
    }
}
```


```java
//采用之后可以省掉一个注解的使用
@Repeatable(Authorities.class)
public @interface Authority {
     String role();
}
public class RepeatAnnotationUseNewVersion {
    @Authority(role="Admin")
    @Authority(role="Manager")
    public void doSomeThing(){ }
}
```

####Groovy Bean Definition DSL
采用groovy 来实现bean的定义,groovy定义感觉要比注解和xml好看一些，比如定义个dataSource的配置，逻辑很清晰，没有多余的标签
```groovy
import org.hibernate.SessionFactory
import org.apache.commons.dbcp.BasicDataSource

beans {
    xmlns context: "http://www.springframework.org/schema/context"  
    xmlns mvc: "http://www.springframework.org/schema/mvc"  
    context.'component-scan'('base-package': "com.test")  
    mvc.'annotation-driven'('validator': "validator")
    dataSource(BasicDataSource) {
        driverClassName = "org.hsqldb.jdbcDriver"
        url = "jdbc:hsqldb:mem:grailsDB"
        username = "sa"
        password = ""
        settings = [mynew:"setting"]
    }
    sessionFactory(SessionFactory) {
        dataSource = dataSource
    }
    myService(MyService) {
        nestedBean = { AnotherBean bean ->
            dataSource = dataSource
        }
    }
}
```
但是目前使用来说，有一点不好，自动提示功能不完善，作为一个java开发者来说，没有提示功能简直要命了，具体完善的配置可以看看开涛大神的github [https://github.com/zhangkaitao/spring4-showcase](https://github.com/zhangkaitao/spring4-showcase)

####Core Container Improvements
核心内容的增强：

1.增强泛型service的注入，可以把通用的增删改查功能写到BaseService里面

2.增加@Description注解，通过jmx监控的时候，能够很好的看到具体类描述信息，配置如下.感觉没有什么太大用处
```java
@Configuration
public class AppConfig {

    @Bean
    @Description("Provides a basic example of a bean")
    public Foo foo() {
        return new Foo();
    }

}
```

3.增加@Conditional() 注解，这个注解主要用来根据不同条件配置生成bean，比如说有一套代码需要兼容linux和windows，有平台化实现的差异。按照以前来实现可以通过if else判断平台new不同的对象去实现。4.0以后可以通过增加这个注解去实现，只需要自己重写判断条件即可
```java
linux判断条件
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class LinuxCondition implements Condition {

    public boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().getProperty("os.name").contains("Linux");
    }

}

//windows判断条件
public class WindowsCondition implements Condition {

    public boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().getProperty("os.name").contains("Windows");
    }

}

//可以通过如下的方式，根据condition返回的结果，去判断实现类的方式
@Configuration
public class DemoConfig {
    @Bean
    @Conditional(WindowsCondition.class)
    public CommandService commandService() {
        return new WindowsCommnadService();
    }

    @Bean
    @Conditional(LinuxCondition.class)
    public CommandService linuxEmailerService() {
        return new LinuxCommandService();
    }

}
```
4.官方描述：If you use Spring’s meta-annotation support, you can now develop custom annotations that expose specific attributes from the source annotation. 这句话大概意思就是我们现在可以使用元注解，来实现自己的自定义注解，让自己的注解拥有元注解功能，例如：
```java
@Repository  
@Scope("prototype")  
@Transactional
public @interface CustomDao {  
}  
```

上述定义了一个自己CustomDao注解 让其拥有`@Repository` `@Scope("prototype") ` `@Transactional`等注解的功能。


####General Web Improvements

1.增加 `@RestController` 注解，用来代替在`@RequestMapping` 上面添加`@ResponseBody`这种方式，当然如果你的一个类里面又有string返回，又有页面返回，这个注解就没法用，最好是把api接口和页面跳转分开

2.增加`AsyncRestTemplate`，在3.0之前是采用同步的`RestTemplate`，来实现rest服务的client端，4.0以后采用`AsyncRestTemplate`实现异步rest client，方便快捷，比使用直接`httpclient`或者`okhttp`方便一些，具体方式如下
```java
//第一种使用方式
// async call
Future<ResponseEntity<String>> futureEntity = template.getForEntity(
    "http://example.com/hotels/{hotel}/bookings/{booking}", String.class, "42", "21");

// get the concrete result - synchronous call
ResponseEntity<String> entity = futureEntity.get();

//第二种使用方式
ListenableFuture<ResponseEntity<String>> futureEntity = template.getForEntity(
    "http://example.com/hotels/{hotel}/bookings/{booking}", String.class, "42", "21");

// register a callback
futureEntity.addCallback(new ListenableFutureCallback<ResponseEntity<String>>() {
    @Override
    public void onSuccess(ResponseEntity<String> entity) {
        //...
    }

    @Override
    public void onFailure(Throwable t) {
        //...
    }
});
```
####WebSocket, SockJS, and STOMP Messaging
增加websocket，sockjs和stomp协议的支持,相应的demo官方已经有[http://docs.spring.io/spring/docs/4.3.0.SPR-13777-SNAPSHOT/spring-framework-reference/htmlsingle/#websocket-intro](http://docs.spring.io/spring/docs/4.3.0.SPR-13777-SNAPSHOT/spring-framework-reference/htmlsingle/#websocket-intro)
####Testing Improvements
增强了测试注解`@ContextConfiguration, @WebAppConfiguration, @ContextHierarchy, @ActiveProfiles` 这些测试注解都可以通过元注解的方式进行自定义自己的注解

spring-core增加SocketUtils,通过这个类来扫描可用的tcp和udp端口，感觉用处不大，大致官方讲解是这样的（This functionality is not specific to testing but can prove very useful when writing integration tests that require the use of sockets, for example tests that start an in-memory SMTP server, FTP server, Servlet container），大致上来说主要用来测试一些个服务是否开启了比如smtp 或者ftp这些开启的状况。


##spring4.1.x新特性介绍
####JMS Improvements
####Caching Improvements
####Web Improvements
####WebSocket Messaging Improvements
####Testing Improvements


##spring4.1.x新特性介绍
####Core Container Improvements
####Data Access Improvements
####JMS Improvements
####Web Improvements
####Testing Improvements
