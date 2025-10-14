---
date: 2022-09-26T22:51:07
tags:
title: Calcite Parser代码生成详解
---
本文代码均已上传到[gitee](https://gitee.com/hezhangjian/calcite-examples)  
calcite的parser代码生成分为如下两个步骤  
  
![calcite-parser-code-generate-process](Images/calcite-parser-code-generate-process.png)  
  
## 生成Parse.jj

文件目录如下

```bash
├── pom.xml
└── src
    ├── main
    │   ├── codegen
    │   │   ├── config.fmpp
    │   │   ├── includes
    │   │   │   ├── compoundIdentifier.ftl
    │   │   │   └── parserImpls.ftl
    │   │   └── templates
    │   │       └── Parser.jj
```

添加calcite dependency

```xml
        <dependency>
            <groupId>org.apache.calcite</groupId>
            <artifactId>calcite-core</artifactId>
        </dependency>
```



配置`drill-fmpp-maven-plugin`插件如下

```xml
            <plugin>
                <groupId>org.apache.drill.tools</groupId>
                <artifactId>drill-fmpp-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <configuration>
                            <config>src/main/codegen/config.fmpp</config>
                            <output>${project.build.directory}/generated-sources/fmpp</output>
                            <templates>src/main/codegen/templates</templates>
                        </configuration>
                        <id>generate-fmpp-sources</id>
                        <phase>validate</phase>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

codegen 模块的文件都拷贝自对应版本的**calclite** **core/src/main/codegen**路径 https://github.com/apache/calcite/tree/main/core/src/main/codegen

然后把https://github.com/apache/calcite/blob/main/core/src/main/codegen/default_config.fmpp 中的parser属性与config.fmpp中的parser属性合并。就可以通过**mvn package**命令生成Parser.jj了。当然，如果有定制化修改的需求，也可以在这个阶段修改config.fmpp  
  
![calcite-parser-code-generator-fmpp](Images/calcite-parser-code-generator-fmpp.png)  
  
## Parser.jj生成java代码  
  
文件目录如下

```bash
├── pom.xml
├── src
│   ├── main
│   │   ├── codegen
│   │   │   └── Parser.jj
```

Parser.jj就是我们上一步生成的Parser.jj，如果有什么想要的定制化修改，也可以在这个步骤改入到Parser.jj中。

添加calcite dependency

```xml
        <dependency>
            <groupId>org.apache.calcite</groupId>
            <artifactId>calcite-core</artifactId>
        </dependency>
```

配置`javacc-maven-plugin`如下
```xml
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>javacc-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>javacc</id>
                        <goals>
                            <goal>javacc</goal>
                        </goals>
                        <configuration>
                            <sourceDirectory>${project.basedir}/src/main/codegen</sourceDirectory>
                            <includes>
                                <include>**/Parser.jj</include>
                            </includes>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

生成代码  
  
![calcite-parser-code-generator-javacc](calcite-parser-code-generator-javacc.png)  
  
## 无Parser.jj定制化修改，一步生成  
  
## 无Parser.jj定制化修改，一步生成

如果不需要对Parser.jj进行定制化修改，那么可以通过连续运行两个插件来生成代码，这里给出pom文件样例，不再赘述

```xml
            <plugin>
                <groupId>org.apache.drill.tools</groupId>
                <artifactId>drill-fmpp-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <configuration>
                            <config>src/main/codegen/config.fmpp</config>
                            <output>${project.build.directory}/generated-sources/fmpp</output>
                            <templates>src/main/codegen/templates</templates>
                        </configuration>
                        <id>generate-fmpp-sources</id>
                        <phase>validate</phase>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>javacc-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>javacc</id>
                        <goals>
                            <goal>javacc</goal>
                        </goals>
                        <configuration>
                            <sourceDirectory>${project.build.directory}/generated-sources/fmpp</sourceDirectory>
                            <includes>
                                <include>**/Parser.jj</include>
                            </includes>
                            <lookAhead>2</lookAhead>
                            <isStatic>false</isStatic>
                        </configuration>
                    </execution>
                    <execution>
                        <id>javacc-test</id>
                        <phase>generate-test-sources</phase>
                        <goals>
                            <goal>javacc</goal>
                        </goals>
                        <configuration>
                            <sourceDirectory>${project.build.directory}/generated-test-sources/fmpp</sourceDirectory>
                            <outputDirectory>${project.build.directory}/generated-test-sources/javacc</outputDirectory>
                            <includes>
                                <include>**/Parser.jj</include>
                            </includes>
                            <isStatic>false</isStatic>
                            <ignoreCase>true</ignoreCase>
                            <unicodeInput>true</unicodeInput>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```
