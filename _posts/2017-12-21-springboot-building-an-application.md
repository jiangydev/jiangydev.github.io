---
layout:     post
title:      "构建一个 Spring Boot 应用"
subtitle:   ""
date:       2017-12-17
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
tags:
    - Spring Boot
---

## Building an Application with Spring Boot

[教程原地址](http://spring.io/guides/gs/spring-boot/)

这个指南提供一个例子 - Spring Boot 怎样帮助你加速和促进应用程序开发。随着你阅读更多 Spring 起步的指南，你将会看到更多 Spring Boot 的使用案例。它的意思是让你快速品味 Spring Boot。如果你想创建你自己的基于 Spring Boot 的项目，访问 [Spring Initializr](https://start.spring.io/)，填充你的项目细节，选择你的选项，然后你可以下载一个 Maven 构建的文件或一个作为 zip 文件的捆绑项目。

#### 你将构建出什么

一个使用 Spring Boot 的简单应用程序并增加一些有用的服务。

#### 你需要什么

* A favorite text editor or IDE
* JDK 1.8 or later
* Gradle 2.3+ or Maven 3.0+
* You can also import the code straight into your IDE:
  * Spring Tool Suite (STS)
  * IntelliJ IDEA

#### 怎样完成这个指南

像大多数 Spring 入门指南一样，你可以从头开始并完成每个步骤，也可以绕过基本安装步骤。无论哪种方式，最终会有工作的代码。

1. 从头开始，通过 Gradle 构建或通过 Maven、IDE 构建。

2. 跳过基础安装，步骤如下：

  * 下载和解压源仓库或使用 Git clone: `git clone https://github.com/spring-guides/gs-spring-boot.git`

  * 切换 (cd) 到`gs-spring-boot/initial`

完成后，可以和 `gs-spring-boot/complete` 中的代码检查一下。

#### 学习用 Spring Boot 能做什么

Spring Boot 提供一种快速的方式构建应用程序。它观察你的 classpath 和已经配置的 beans，做出你丢失东西的理由充分的认定并增加它。使用 Spring Boot，你能关注更多的项目功能，更少的精力在基础架构上。

例如：

  1. 使用 Spring MVC?

    有几个你几乎总是需要的特定 beans，Spring Boot 自动增加它们。一个 Spring MVC app 也需要一个 servlet 容器，所以 Spring Boot 自动配置嵌入的 Tomcat.

  2. 使用 Jetty?

    如果这样，你可能不想要 Tomcat，替换成 Jetty 就好了，Spring Boot 为你处理。

  3. 使用 Thymeleaf?

    有少数的 beans 总是必须加到你的应用程序上下文，Sring Boot 为你增加它们。

这只是几个 Spring Boot 提供自动化配置的例子。同时，Spring Boot 不会阻挡你的路。例如，如果你使用 Thymeleaf，Spring Boot 自动添加 `SringTemplateEngine` 到你的应用程序上下文。但是，如果你用自己的设置定义了 `SringTemplateEngine` Spring Boot 则不会添加。这使你在控制你的部分少一些精力。

> Spring Boot 不生成代码或编辑你的文件。相反，当你启动你的应用程序，Spring Boot 动态地将 beans 和设置连线，并将它们应用到应用程序的上下文。

#### 创建一个简单的 web 应用程序

现在你可以给一个简单的应用程序创建一个 web controller。

`src/main/java/hello/HelloController.java`

```java
package hello;

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;

// 使用 @RestController 注解，意味着准备好使用 Spring MVC 处理请求
// @RestController 组合了 @Controller 和 @ResponseBoby，这两个注解的结果是返回数据，而不是视图。
@RestController
public class HelloController {

    // 将 '/' 映射到 index() 方法；
    @RequestMapping("/")
    public String index() {
        return "Greetings from Spring Boot!";
    }

}
```

#### 创建一个应用程序类

`src/main/java/hello/Application.java`

```java
package hello;

import java.util.Arrays;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
// @SpringBootAppication 是一个方便的注解，增加了以下内容：
// 1. @Configuration：标志这个类作为一个为应用程序上下文的 bean 定义的源。
// 2. @EnableAutoConfiguration：告诉 Spring Boot 开始增加基于 Classpath 设置、其他 beans 和不同属性设置的 beans。
// 3. 你通常想给一个 Spring MVC app 增加 @EnableWebMvc，但当 Spring Boot 看见 spring=webmvc 在 classpath 时，就自动添加了。这个标志应用程序作为一个 web 应用程序并激活键行为例如设置一个 DispatcherServlet.
// 4. @ComponentScan 告诉 Spring 在 hello 包中寻找其他的组件、配置和服务，允许它找 controllers.
public class Application {

  // main() 使用 Spring Boot 的 SpringApplication.run() 方法来启动一个应用程序。
  public static void main(String[] args) {
      SpringApplication.run(Application.class, args);
  }

  // 被标记 @Bean 并且在启动时运行。它取所有创建的（无论是你的 app 还是 Spring Boot 自动添加的）beans，排序并打印出它们。
  @Bean
  public CommandLineRunner commandLineRunner(ApplicationContext ctx) {
      return args -> {
         System.out.println("Let's inspect the beans provided by Spring Boot:");

          String[] beanNames = ctx.getBeanDefinitionNames();
          Arrays.sort(beanNames);
          for (String beanName : beanNames) {
              System.out.println(beanName);
          }
     };
  }
}
```

你会注意到没有一行 XML，也没有 web.xml，这个应用程序纯 Java 并且你没必要配置任何管道工程(plumbing)或基础设置。

#### 运行应用程序

运行程序，执行：`./gradlew build && java -jar build/libs/gs-spring-boot-0.1.0.jar`

如果使用 Maven，执行：`mvn package && java -jar target/gs-spring-boot-0.1.0.jar`

执行后，你会清晰地看到 `org.springframework.boot.autoconfigure` beans 还有 `tomcatEmbeddedServletContainerFactory`.

检查 service:

  ```
  $ curl localhost:8080
  Greetings from Spring Boot!
  ```

补充：

  使用 `mvn dependency:tree` 可以在 cmd 下以树状结构显示当前项目依赖的 jar 包。

  从依赖的包可以看出，运行项目时，默认加载嵌入在 Spring Boot 中的 Tomcat 容器，如果要使用 Jetty，需要在 pom.xml 去除 Tomcat，然后再依赖 Jetty。

    ```XML
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-web</artifactId>  
        <exclusions>  
            <exclusion>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-starter-tomcat</artifactId>  
            </exclusion>  
        </exclusions>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-jetty</artifactId>  
    </dependency>
    ```

    ```java
    /**
     *jetty: Unable to start EmbeddedWebApplicationContext due to missing EmbeddedServletContainerFactory bean.
     * @return
     */  
    @Bean  
    public EmbeddedServletContainerFactory servletContainer() {  
        JettyEmbeddedServletContainerFactory factory =  
                new JettyEmbeddedServletContainerFactory();  
        return factory;  
    }
    ```

#### 添加单元测试

> mock测试: 在测试过程中，对于某些不容易构造或者不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法。

你想给你添加的端点（endpoint）添加一个测试，Spring Test 已经提供一些机制，很容易包含在项目中。

Gradle: `testCompile("org.springframework.boot:spring-boot-starter-test")`

Maven:

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
  </dependency>
  ```

现在，写一个简单的单元测试来通过你的端点 mock request 和 response。

  `src/test/java/hello/HelloControllerTest.java`

  ```java
  package hello;

  import static org.hamcrest.Matchers.equalTo;
  import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
  import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.http.MediaType;
  import org.springframework.test.context.junit4.SpringRunner;
  import org.springframework.test.web.servlet.MockMvc;
  import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

  @RunWith(SpringRunner.class)
  // 下面两个注解一起使用，注入 MockMvc 实例。
  @SpringBootTest
  @AutoConfigureMockMvc
  public class HelloControllerTest {

      // 这个 MockMvc 来自 Spring Test, 允许你通过一个方便的构建类集合发送 HTTP 请求到 DispatcherServlet 并断言结果；
      @Autowired
      private MockMvc mvc;

      @Test
      public void getHello() throws Exception {
          mvc.perform(MockMvcRequestBuilders.get("/").accept(MediaType.APPLICATION_JSON))
  		.andExpect(status().isOk())
  		.andExpect(content().string(equalTo("Greetings from Spring Boot!")));
      }
  }
  ```

  使用 `@SpringBootTest`，会创建整个应用程序上下；一个可替换的选择是使用 `@WebMvcTest` 创建仅仅 web 层上下文。Spring Boot 自动尝试在两种情况下查找应用程序的主程序类，但如果要生成不同的东西，你可以将其重写或缩小（narrow it down）。

正如模拟 HTTP 请求周期，我们也能使用 Spring Boot 编写一个非常简单的全栈集成测试（full-stack integration test）。

  `src/test/java/hello/HelloControllerIT.java`

  ```java
  package hello;

  import static org.hamcrest.Matchers.equalTo;
  import static org.junit.Assert.assertThat;

  import java.net.URL;

  import org.junit.Before;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.context.embedded.LocalServerPort;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.boot.test.web.client.TestRestTemplate;
  import org.springframework.http.ResponseEntity;
  import org.springframework.test.context.junit4.SpringRunner;

  @RunWith(SpringRunner.class)
  // 在随机端口启动
  @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
  public class HelloControllerIT {

      // 运行时发现实际端口
      @LocalServerPort
      private int port;

      private URL base;

      @Autowired
      private TestRestTemplate template;

      @Before
      public void setUp() throws Exception {
        this.base = new URL("http://localhost:" + port + "/");
      }

      @Test
      public void getHello() throws Exception {
        ResponseEntity<String> response = template.getForEntity(base.toString(), String.class);
  	    assertThat(response.getBody(), equalTo("Greetings from Spring Boot!"));
      }
  }
  ```

#### 添加生产级服务(production-grade services)

如果你正在为你的公司构建一个网站，你可能需要增加一些管理服务。Spring Boot 提供几个现成的执行器模块（actuator module），如 health, audits, beans 等。

添加依赖到配置文件：

  Gradle: `compile("org.springframework.boot:spring-boot-starter-actuator")`

  Maven:

    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    ```

然后重启 app.

你会看到一个新的 RESTful 端点的集合被添加到应用程序，有许多管理服务是由 Spring Boot 提供的。

  ```
  : Mapped "{[/error],methods=[],params=[],headers=[]...
  : Mapped "{[/error],methods=[],params=[],headers=[]...
  : Mapped URL path [/**] onto handler of type [class...
  : Mapped URL path [/webjars/**] onto handler of typ...
  : Mapped "{[/info],methods=[GET],params=[],headers=...
  : Mapped "{[/autoconfig],methods=[GET],params=[],he...
  : Mapped "{[/mappings],methods=[GET],params=[],head...
  : Mapped "{[/trace],methods=[GET],params=[],headers...
  : Mapped "{[/env/{name:.*}],methods=[GET],params=[]...
  : Mapped "{[/env],methods=[GET],params=[],headers=[...
  : Mapped "{[/configprops],methods=[GET],params=[],h...
  : Mapped "{[/metrics/{name:.*}],methods=[GET],param...
  : Mapped "{[/metrics],methods=[GET],params=[],heade...
  : Mapped "{[/health],methods=[GET],params=[],header...
  : Mapped "{[/dump],methods=[GET],params=[],headers=...
  : Mapped "{[/beans],methods=[GET],params=[],headers...
  ```

  他们包含：errors, [environment](http://localhost:8080/env), [health](http://localhost:8080/health), [beans](http://localhost:8080/beans), [info](http://localhost:8080/info), [metrics](http://localhost:8080/metrics), [trace](http://localhost:8080/trace), [configprops](http://localhost:8080/configprops), and [dump](http://localhost:8080/dump).

  > 也有一个 /shutdown 的端点, 但它仅仅通过默认 JMX 方式才可见。为了使它生效, 添加 endpoints.shutdown.enabled=true 到你的 application.properties 文件.

  可以很容易地检查 app 的 health:

    ```
    $ curl localhost:8080/health
    {"status":"UP"}
    ```

  你可以通过 curl 尝试调用 shutdown:

    ```
    $ curl -X POST localhost:8080/shutdown
    {"timestamp":1513937561289,"status":404,"error":"Not Found","message":"No message available","path":"/shutdown"}
    ```

    因为没有启用 shutdown, 请求被不存在的所阻止。

  关于这些 RESTful 点的详细信息，以及如何使用 application.properties 文件调整它们的设置，可以阅读 [docs about the endpoints](https://docs.spring.io/spring-boot/docs/1.5.9.RELEASE/reference/htmlsingle/#production-ready-endpoints).

#### Spring Boot 的启动器

[Spring Boot’s "starters"](https://docs.spring.io/spring-boot/docs/1.5.9.RELEASE/reference/htmlsingle/#using-boot-starter)

#### JAR 支持和 Groovy 支持
