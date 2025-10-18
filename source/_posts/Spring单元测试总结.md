---
title: Spring单元测试总结
link: spring-unittests
date: 2023-08-12 11:13:01
tags:
  - Java
  - Spring
---

## 目录

- 模块组织
- 测试手段
- 依赖组件

## 典型Spring单元测试模块组织

```
-- xxx-app
-- xxx-util
-- test-common
```

test-common尽量减少依赖，仅依赖必须的非spring组件。也可以统一将需要使用的resources文件放到test-common中。由test-common统一管理，避免每个模块测试都需要拷贝必须的文件。所需的maven配置如下：

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**</include>
                <include>**/**</include>
            </includes>
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>${maven-resources-plugin.version}</version>
            <executions>
                <execution>
                    <id>copy-resources</id>
                    <phase>process-resources</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/resources</outputDirectory>
                        <resources>
                            <resource>
                                <directory>src/main/resources</directory>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

一些典型的配置文件，比如log4j2配置文件，同时，由于test-common不属于测试代码，可能在某些组织下会有更高的要求（如不能存在敏感信息等），如组织有这样的要求，则这类内容不适合放在test-common里统一复用:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="info" monitorInterval="10">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern='%d{yyyy-MM-dd,HH:mm:ss,SSSXXX}(%C:%L):%4p%X[%t#%T]-->%m%n'/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```

## 测试手段

### 利用RestAssured端到端测试http接口

添加依赖

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.3.1</version>
    <scope>test</scope>
</dependency>
```

为了在SpringBoot测试中使用 **`RestAssured`**, 需要配置端口 **webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT**。如：

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class MyRestControllerTest {

    @LocalServerPort
    int port;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;
    }
}
```

随后可以使用RestAssured来请求接口

```java
RestAssured.given().contentType(ContentType.JSON).body("{}").post("url").then().statusCode(200);
```

## 依赖组件

### mariadb

mariadb可以使用mariadb4j

```xml
<dependency>
    <groupId>ch.vorburger.mariaDB4j</groupId>
    <artifactId>mariaDB4j</artifactId>
    <version>3.0.1</version>
    <scope>test</scope>
</dependency>
```

书写Extension并使用

```java
import ch.vorburger.mariadb4j.DB;
import ch.vorburger.mariadb4j.DBConfigurationBuilder;
import org.junit.jupiter.api.extension.AfterAllCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

public class MariaDBExtension implements BeforeAllCallback, AfterAllCallback {

    private DB database;

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        DBConfigurationBuilder configBuilder = DBConfigurationBuilder.newBuilder();
        configBuilder.setPort(3306);
        database = DB.newEmbeddedDB(configBuilder.build());
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        if (database != null) {
            database.stop();
        }
    }
}
```

### ignite

Ignite可以使用现有的junit5集成

```xml
<dependency>
    <groupId>io.github.embedded-middleware</groupId>
    <artifactId>embedded-ignite-junit5</artifactId>
    <version>0.0.3</version>
</dependency>
```

可以直接使用EmbeddedIgniteExtension，还可以使用EmbeddedIgnitePorts自定义Ignite的关键端口号
