---
title: create-own-spring-boot-starter
date: 2021-07-21 23:47:12
tags:
---

Spring Boot 

<!-- more -->

当 Spring Boot 启动时，它会在类路径中查找名为 spring.factories 的文件。此文件位于 META-INF 目录中。
以 mybatis-spring-boot-starter 为例，其`META-INF/spring.factories`中的内容如下

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

Spring Boot 2.7 引入了一个用于注册自动配置的新文件`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，同时保留了原有的`spring.factories`注册方式。

Spring Boot 3.0 移除了对在`spring.factories`中使用键`org.springframework.boot.autoconfigure.EnableAutoConfiguration`注册自动配置的支持，仅支持`imports`文件注册方式。`spring.factories`中的其他键不受影响。

所以为确保 starter 的兼容性，`spring.factories`和`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`均需要进行配置。

其`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`中的内容如下

```properties
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

- [Spring Boot 3.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#auto-configuration-files)
- [Remove spring.factories auto-configuration support · Issue #29699 · spring-projects/spring-boot (github.com)](https://github.com/spring-projects/spring-boot/issues/29699)
