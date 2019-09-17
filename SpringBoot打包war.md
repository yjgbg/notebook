## 修改pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> 
    </parent>
    <groupId>com.example</groupId>
    <artifactId>wartest</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>war</packaging><!--这是修改的地方-->
    <name>wartest</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency><!--这是修改的地方-->
            <groupId>org.springframework.boot</groupId><!--这是修改的地方-->
            <artifactId>spring-boot-starter-tomcat</artifactId><!--这是修改的地方-->
            <scope>provided</scope><!--这是修改的地方-->
        </dependency><!--这是修改的地方-->

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
</project>
```

## 修改Spring入口类

```java
package com.example.wartest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

@SpringBootApplication
public class WartestApplication extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(WartestApplication.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(WartestApplication.class);
    }
}

```

然后在maven的控制面板（Ctrl+Tab+M)点击\${包名}->Lifecycle->package，即可生成war包，地址在与src同级的target目录下

war包里面仍然会有内置tomcat存在，但是不影响项目正常启动，也可以手动删除。