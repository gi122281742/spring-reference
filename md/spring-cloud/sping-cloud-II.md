# Spring Cloud Config

sc Config 提供了服务边和客户边在分布式系统的外部化配置支持。在config servver中，你有一个中央地来管理所有环境中app额外的属性。这个client和server的映射概念与spring 的Environment 和PropertySource  抽象相同，因此他们非常适合spring app，但是也可以与其他任何app使用任何语言运行。就像一个app部署管道一样，从dev到test，再到production。你可以管理这个配置在这些环境之间，并确保app具有它需要在迁移时运行的所有内容。默认server storage 后端的实现使用git，因此它容易支持配置环境的版本标记以及可以无障碍的多种工具用来管理内容。使用spring配置添加替代实现以及插入他们十分容易。

## 4. Quick Start

使用sc config server的server和client。
首先启动服务,eg:
```
$ cd spring-cloud-config-server
$ ../mvnw spring-boot:run
```
这个服务时sb app，因此你可以在IDE上运行它（main class是ConfigServerApplication）。
接下来尝试client，eg：
```
$ curl localhost:8888/foo/development
{"name":"foo","label":"master","propertySources":[
  {"name":"https://github.com/scratches/config-repo/foo-development.properties","source":{"bar":"spam"}},
  {"name":"https://github.com/scratches/config-repo/foo.properties","source":{"foo":"bar"}}
]}
```
默认定位属性源策略是clone一个git仓库(at spring.cloud.config.server.git.uri)以及使用它来初始化mini SpringApplication。mini-app的环境用来枚举属性源和发布一个json端点。
当application 被注入为SpringApplication 的spring.config.name（什么是常规sb app的一般application），profile是激活的profile（逗号分割的属性），以及lable是客园的git lable（默认master）
sc Config Server 从git仓库pull远程客户端的配置，eg:
```
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
```

## 4.1 Client Side Usage 

在app使用这个特点，你可以创建它作为一个一代sc-config-client 的sb app（比如，查看config-client测试用例或者sample app）。最方便的添加依赖是使用一个 Spring Boot starter（org.springframework.cloud:spring-cloud-starter-config）。有一个meven 用户的的父级pom和bom(spring-cloud-starter-parent）以及Gradle，spring cli用户的spring IO版本管理属性文件。eg：

**pom.xml**
```
   <parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>{spring-boot-docs-version}</version>
       <relativePath /> <!-- lookup parent from repository -->
   </parent>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>{spring-cloud-version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-config</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>

<build>
	<plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
           </plugin>
	</plugins>
</build>

   <!-- repositories also needed for snapshots and milestones -->
```
现在你可以创建一个sb标准app，就像下面的HTTP server：
```java
@SpringBootApplication
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello World!";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```
当一个HTTP server运行，会从默认本地配置服务（如果正在运行）端口8888上获取额外的配置。为了修改这个启动行为，你可以通过bootstrap.properties改变config server 的location（与application。properties相似,但是是关于app context的bootstap期），eg：
```
spring.cloud.config.uri: http://myconfigserver.com
```
默认，如果app没有设置名字，application会被使用。为了修改名字,下面的属性可以被添加到bootstrap.properties文件:
```
spring.application.name: myapp
```
当设置属性${spring.application.name}时，不要在名字前添加保留字application-，以防止问题解析正确的属性源。
bootstrap属性显示在/env端点，并作为高优先级属性源，eg：
```
$ curl localhost:8080/env
{
  "profiles":[],
  "configService:https://github.com/spring-cloud-samples/config-repo/bar.properties":{"foo":"bar"},
  "servletContextInitParams":{},
  "systemProperties":{...},
  ...
}
```
一个属性源叫configService:\<URL of remote repository>/\<file name>包含了值为bar的foo属性，并且高优先级。

> url 在属性元名字是git仓库，而不是config server url。
