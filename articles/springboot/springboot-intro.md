# Spring Boot 实战入门

## 简介

Spring Boot 是由 Pivotal 团队提供的基于 Spring 的框架,旨在简化 Spring 应用的初始搭建和开发过程。

## 主要特性

- ✅ 自动配置
- ✅ 内嵌服务器(Tomcat、Jetty)
- ✅ 简化的依赖管理
- ✅ 生产就绪功能

## 快速开始

### 1. 创建项目

使用 Spring Initializr 或 Maven/Gradle 创建项目:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 2. 编写控制器

```java
@RestController
public class HelloController {
    
    @GetMapping("/hello")
    public String hello() {
        return "Hello, Spring Boot!";
    }
}
```

### 3. 启动应用

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 常用注解

- `@RestController`: 标记为 RESTful 控制器
- `@RequestMapping`: 映射请求路径
- `@Autowired`: 自动注入依赖
- `@Service`: 服务层组件

## 总结

Spring Boot 让 Java Web 开发变得更加简单高效,是现代 Java 开发的首选框架。

---

*最后更新: 2026-05-14*
