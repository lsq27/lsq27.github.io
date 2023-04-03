# Spring Boot SPI 机制探究

## 前言
Spring Boot 带给我们的一大便利是当需要引入一个第三方依赖时，如果其有 Starter，可以加入 Starter 依赖，就可以实现自动装配，这中便利就来自 Spring Boot 独特的 SPI 机制。

比如项目中希望引入 Mybatis，只需在 POM 中加入以下代码：
```xml
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>2.3.0</version>
</dependency>
```
剩下的工作就是定义 Mapper 并使用，中间所有繁杂的配置 Starter 都帮忙做了。如此神奇的功能其实实现原理十分简单，下文进行分析。
## 获取配置类的全限定名
从 `@SpringBootApplication` 注解一路向下寻找，依次找`@EnableAutoConfiguration`、`@Import(AutoConfigurationImportSelector.class)`、`getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes)`，可以看到熟悉的 `spring.factories`，此处调用的两个函数即为 Spring SPI 的扫描逻辑，代码如下：
```java
List<String> configurations = new ArrayList<>(
		SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);
Assert.notEmpty(configurations,
		"No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
				+ "are using a custom packaging, make sure that file is correct.");
```
### `META-INF/spring.factories`
第一种方式是读取 classpath 中的所有 `META-INF/spring.factories` 文件，此处有三个循环，第一层遍历获取到的文件位置，第二层遍历文件中的属性，第三层遍历每个属性用 `,` 分割后的字符串（类路径），这样就获取到了所有需要进行装配的 Configuration 类。
```java
Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
while (urls.hasMoreElements()) {
	URL url = urls.nextElement();
	UrlResource resource = new UrlResource(url);
	Properties properties = PropertiesLoaderUtils.loadProperties(resource);
	for (Map.Entry<?, ?> entry : properties.entrySet()) {
		String factoryTypeName = ((String) entry.getKey()).trim();
		String[] factoryImplementationNames =
				StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
		for (String factoryImplementationName : factoryImplementationNames) {
			result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
					.add(factoryImplementationName.trim());
		}
	}
}
```
### `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
在 Spring Boot 2.7.0 中，增加了对另一种自动配置方式的支持，即读取资源文件 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，第一层遍历获取到的文件位置，第二层遍历文件中的行。
```java
String location = String.format(LOCATION, annotation.getName());
Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location);
List<String> importCandidates = new ArrayList<>();
while (urls.hasMoreElements()) {
	URL url = urls.nextElement();
	importCandidates.addAll(readCandidateConfigurations(url));
}
```
## 实例化
得到类名后，Spring 利用 `Class.forName` 将所有需要进行装配的配置类进行加载实例化，不再赘述。
## 总结
学习编码的过程中不能被像 SPI 看起来高深的概念所迷惑，看起来很神奇的自动装配功能其实归根到底就是读取特定名称的配置文件，然后反射获取配置类。
