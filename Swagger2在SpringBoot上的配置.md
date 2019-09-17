# Swagger

## 简介

Swagger的目标是为REST APIs定义一个标准的，与语言无关的接口，使人与计算机在看不到源码或者看不到文档或者不能通过网络流量检测的情况下能发现和理解各种服务的功能。当服务Swagger定义，消费者可以通过少量的实现逻辑与远程的服务互动，。类似于低级编程接口，Swagger去掉了很多调用服务时候的猜测。

简而言之：Swagger通过读取注解的方式，将REST API的作用，参数限制等一系列信息通过swagger-ui.html曝露给调用者。

## 原理

反射，扫描所有controller类，以及其中所有标记有mapping的方法（即API的实现方法）。对这些方法，controller的Swagger注解进行扫描，形成API描述信息，呈现在swagger-ui.html，并在界面提供对API的测试(基于curl命令)。

## Swagger注解介绍

### `@Api`

标注在controller上，标记这是一组api

- tags: API分组标签
- value: 如果没有定义tags，则value是tags
- description： deprecated，API的详细描述

### `@ApiOperation`

标注在controller的方法上，标记这是一个api

对一个操作进行描述。

- value： 对该api的简单描述，长度为60个汉字，120个字母
- notes：对该api的详细描述
- httpMethod: 该api请求的动作值，可选”GET”,”POST”,”HEAD”,”PUT”,”DELETE”,”OPTION”,”PATCH”
- code: 返回码，默认为200

### `@ApiImplicitParams和@ApiImplicitParam`

对api的单一参数进行注解，介绍单个参数

- name: 参数名称
- value： 简短介绍
- required： 是否为必选参数
- dataType： 参数类型
- paramType：参数的传入（请求）类型，可选的值有path, query, body, header or form

### `@ApiResponses和@ApiResponse`

描述可能的返回结果。

这个注解用于描述所有可能的成功或错误的返回码。

- code 返回码
- message 返回更容易理解的文本消息
- repsonse返回类型信息，使用全限定类名
- responseContainer 如果返回类型为容器类型，可以设置相应的值，有效值为LIST，SET和MAP

## 配置方式

### 导包

```groovy
    // https://mvnrepository.com/artifact/io.springfox/springfox-swagger2
    implementation group: 'io.springfox', name: 'springfox-swagger2', version: '2.9.2'
    // https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui
    implementation group: 'io.springfox', name: 'springfox-swagger-ui', version: '2.9.2'5
```

会导入以下包:

```groovy
io.springfox:springfox-core
io.springfox:springfox-schema
io.springfox:springfox-spi
io.springfox:springfox-spring-web
io.springfox:springfox-swagger2
io.springfox:springfox-swagger-common
io.springfox:springfox-swagger-ui
```

### 配置SwaggerConfig

```java
@EnableSwagger
//开启swagger22@ConfigurationProperties(prefix = "swagger")//标记当前类中的field会从配置文件中取值@Configuration//标记这个类为配置类（对应于Spring的xml文件）、
@Data //这个注解来自于lombok，标记当前类为数据（自动生成getter和setter）
@EnableWebMvc//这个会覆盖掉WebMvc的默认配置autoconfig，导致静态文件无法访问.
@ComponentScan(basePackages = {"me.yjgbg.demo0.controller"})
public class SwaggerConfig {
    private String name;
    private String email;
    private String url;
    private String title;
    private String description;
    private String version;

    @Bean
    public Docket customDocket(){
        return new Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo());
    }
    @Bean
    public ApiInfo apiInfo(){//api作者信息
        Contact  contact = new Contact(name,url,email);
        return new ApiInfoBuilder()
                .title(title)
                .description(description)
                .contact(contact)
                .version(version)
                .build();
    }
}
```

### 配置WebConfig

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    public void addResourceHandlers(ResourceHandlerRegistry registry){
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");//webmvc的默认配置有这个，但是swaggerConfig中的enableWebMvc替换掉了默认的webmvc设置，因此需要重新配置所有静态资源的地址。但是如果不加入enableWebMvc注解，而是在配置文件中加入swagger-ui.html的资源路径，则不需要配置本类。
        registry.addResourceHandler("index.html")
                .addResourceLocations("classpath:/static/");
        registry.addResourceHandler("/css/**")
                .addResourceLocations("classpath:/static/css/");
        registry.addResourceHandler("/js/**")
                .addResourceLocations("classpath:/static/js/");
        registry.addResourceHandler("/img/**")
                .addResourceLocations("classpath:/static/img/");
    }
}
//配置静态资源映射19//WebMvcConfigurer接口中的所有方法皆有方法体，为配置各种较为常见的handler，上面配置的是静态资源处理器
```

或者关掉SwaggerConfig类中的@EnableWebMvc, 并且在配置文件中加入如下：

```yml
spring:
    resources:
        static-locations:
            - classpath:/META-INF/resources/   #这一行是配置swagger-ui.html的路径
            - classpath:/static/
```

