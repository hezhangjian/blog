---
title: Java Maven lint推荐配置
link: java-maven-lint
date: 2021-12-16 08:55:28
tags:
  - Java
---

# maven checkstyle

## 添加maven plugin依赖

```xml
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-checkstyle-plugin</artifactId>
                <version>3.1.2</version>
                <configuration>
                    <configLocation>config/checkstyle.xml</configLocation>
                    <suppressionsLocation>config/suppressions.xml</suppressionsLocation>
                    <includeTestSourceDirectory>true</includeTestSourceDirectory>
                    <encoding>UTF-8</encoding>
                    <excludes>**/proto/*</excludes>
                </configuration>
            </plugin>
```

- configLocation 存放checkstyle的规则配置文件，附录有样例内容
- SuppressionsLocation 存放屏蔽规则配置文件，附录有样例内容
- includeTestSourceDirectory 是否检测测试文件夹，建议配置为true

![maven-lint-checkstyle-config](Images/maven-lint-checkstyle-config.png)

## 结束

最后就可以通过`mvn checkstyle:check`来检查您的工程啦。如果有违反了checkstyle的地方，命令行会提示出错的地方和违反的规则，如下图所示

![maven-lint-checkstyle-fail1](Images/maven-lint-checkstyle-fail1.png)

![maven-lint-checkstyle-fail2](Images/maven-lint-checkstyle-fail2.png)

## 附录

### 规则配置文件举例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE module PUBLIC
        "-//Puppy Crawl//DTD Check Configuration 1.3//EN"
        "https://checkstyle.org/dtds/configuration_1_3.dtd">

<!-- This is a checkstyle configuration file. For descriptions of
what the following rules do, please see the checkstyle configuration
page at http://checkstyle.sourceforge.net/config.html -->
<module name="Checker">
    <module name="FileTabCharacter">
        <!-- Checks that there are no tab characters in the file. -->
    </module>

    <!-- All Java AST specific tests live under TreeWalker module. -->
    <module name="TreeWalker">

        <module name="SuppressionCommentFilter">
            <property name="offCommentFormat" value="CHECKSTYLE.OFF\: ([\w\|]+)"/>
            <property name="onCommentFormat" value="CHECKSTYLE.ON\: ([\w\|]+)"/>
            <property name="checkFormat" value="$1"/>
        </module>

        <module name="SuppressWarningsHolder" />

        <!--

        IMPORT CHECKS

        -->

        <module name="RedundantImport">
            <!-- Checks for redundant import statements. -->
            <property name="severity" value="error"/>
            <message key="import.redundancy"
                     value="Redundant import {0}."/>
        </module>

        <module name="AvoidStarImport">
            <property name="severity" value="error"/>
        </module>

        <module name="RedundantModifier">
            <!-- Checks for redundant modifiers on various symbol definitions.
              See: http://checkstyle.sourceforge.net/config_modifier.html#RedundantModifier
            -->
            <property name="tokens" value="METHOD_DEF, VARIABLE_DEF, ANNOTATION_FIELD_DEF, INTERFACE_DEF, CLASS_DEF, ENUM_DEF"/>
        </module>

        <!--
            IllegalImport cannot blacklist classes, and c.g.api.client.util is used for some shaded
            code and some useful code. So we need to fall back to Regexp.
        -->
        <module name="RegexpSinglelineJava">
            <property name="format" value="com\.google\.api\.client\.util\.(ByteStreams|Charsets|Collections2|Joiner|Lists|Maps|Objects|Preconditions|Sets|Strings|Throwables)"/>
        </module>

        <!--
             Require static importing from Preconditions.
        -->
        <module name="RegexpSinglelineJava">
            <property name="format" value="^import com.google.common.base.Preconditions;$"/>
            <property name="message" value="Static import functions from Guava Preconditions"/>
        </module>

        <module name="UnusedImports">
            <property name="severity" value="error"/>
            <property name="processJavadoc" value="true"/>
            <message key="import.unused"
                     value="Unused import: {0}."/>
        </module>

        <!--

        JAVADOC CHECKS

        -->

        <!-- Checks for Javadoc comments.                     -->
        <!-- See http://checkstyle.sf.net/config_javadoc.html -->
        <module name="JavadocMethod">
            <property name="scope" value="protected"/>
            <property name="severity" value="error"/>
            <property name="allowMissingParamTags" value="true"/>
            <property name="allowMissingReturnTag" value="true"/>
        </module>

        <!-- Check that paragraph tags are used correctly in Javadoc. -->
        <!--        <module name="JavadocParagraph"/>-->

        <module name="JavadocType">
            <property name="scope" value="protected"/>
            <property name="severity" value="error"/>
            <property name="allowMissingParamTags" value="true"/>
        </module>

        <module name="JavadocStyle">
            <property name="severity" value="error"/>
            <property name="checkHtml" value="true"/>
        </module>

        <!--

        NAMING CHECKS

        -->

        <!-- Item 38 - Adhere to generally accepted naming conventions -->

        <module name="PackageName">
            <!-- Validates identifiers for package names against the
              supplied expression. -->
            <!-- Here the default checkstyle rule restricts package name parts to
              seven characters, this is not in line with common practice at Google.
            -->
            <property name="format" value="^[a-z]+(\.[a-z][a-z0-9]{1,})*$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="TypeNameCheck">
            <!-- Validates static, final fields against the
            expression "^[A-Z][a-zA-Z0-9]*$". -->
            <metadata name="altname" value="TypeName"/>
            <property name="severity" value="error"/>
        </module>

        <module name="ConstantNameCheck">
            <!-- Validates non-private, static, final fields against the supplied
            public/package final fields "^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$". -->
            <metadata name="altname" value="ConstantName"/>
            <property name="applyToPublic" value="true"/>
            <property name="applyToProtected" value="true"/>
            <property name="applyToPackage" value="true"/>
            <property name="applyToPrivate" value="false"/>
            <property name="format" value="^([A-Z][A-Za-z0-9_]*|FLAG_.*)$"/>
            <message key="name.invalidPattern"
                     value="Variable ''{0}'' should be in ALL_CAPS (if it is a constant) or be private (otherwise)."/>
            <property name="severity" value="error"/>
        </module>

        <module name="StaticVariableNameCheck">
            <!-- Validates static, non-final fields against the supplied
            expression "^[a-z][a-zA-Z0-9]*_?$". -->
            <metadata name="altname" value="StaticVariableName"/>
            <property name="applyToPublic" value="true"/>
            <property name="applyToProtected" value="true"/>
            <property name="applyToPackage" value="true"/>
            <property name="applyToPrivate" value="true"/>
            <property name="format" value="^[a-z][a-zA-Z0-9]*_?$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="MemberNameCheck">
            <!-- Validates non-static members against the supplied expression. -->
            <metadata name="altname" value="MemberName"/>
            <property name="applyToPublic" value="true"/>
            <property name="applyToProtected" value="true"/>
            <property name="applyToPackage" value="true"/>
            <property name="applyToPrivate" value="true"/>
            <property name="format" value="^[a-z][a-zA-Z0-9]*$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="MethodNameCheck">
            <!-- Validates identifiers for method names. -->
            <metadata name="altname" value="MethodName"/>
            <property name="format" value="(^[a-z][a-zA-Z0-9]*(_[a-zA-Z0-9]+)*$|Void)"/>
            <property name="severity" value="error"/>
        </module>

        <module name="ParameterName">
            <!-- Validates identifiers for method parameters against the
              expression "^[a-z][a-zA-Z0-9]*$". -->
            <property name="severity" value="error"/>
        </module>

        <module name="LocalFinalVariableName">
            <!-- Validates identifiers for local final variables against the
              expression "^[a-z][a-zA-Z0-9]*$". -->
            <property name="severity" value="error"/>
        </module>

        <module name="LocalVariableName">
            <!-- Validates identifiers for local variables against the
              expression "^[a-z][a-zA-Z0-9]*$". -->
            <property name="severity" value="error"/>
        </module>

        <!-- Type parameters must be either one of the four blessed letters
        T, K, V, W, X or else be capital-case terminated with a T,
        such as MyGenericParameterT -->
        <module name="ClassTypeParameterName">
            <property name="format" value="^(((T|K|V|W|X|R)[0-9]*)|([A-Z][a-z][a-zA-Z]*))$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="MethodTypeParameterName">
            <property name="format" value="^(((T|K|V|W|X|R)[0-9]*)|([A-Z][a-z][a-zA-Z]*T))$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="InterfaceTypeParameterName">
            <property name="format" value="^(((T|K|V|W|X|R)[0-9]*)|([A-Z][a-z][a-zA-Z]*T))$"/>
            <property name="severity" value="error"/>
        </module>

        <module name="LeftCurly">
            <!-- Checks for placement of the left curly brace ('{'). -->
            <property name="severity" value="error"/>
        </module>

        <module name="RightCurly">
            <!-- Checks right curlies on CATCH, ELSE, and TRY blocks are on
            the same line. e.g., the following example is fine:
            <pre>
              if {
                ...
              } else
            </pre>
            -->
            <!-- This next example is not fine:
            <pre>
              if {
                ...
              }
              else
            </pre>
            -->
            <property name="option" value="same"/>
            <property name="severity" value="error"/>
        </module>

        <!-- Checks for braces around if and else blocks -->
        <module name="NeedBraces">
            <property name="severity" value="error"/>
            <property name="tokens" value="LITERAL_IF, LITERAL_ELSE, LITERAL_FOR, LITERAL_WHILE, LITERAL_DO"/>
        </module>

        <module name="UpperEll">
            <!-- Checks that long constants are defined with an upper ell.-->
            <property name="severity" value="error"/>
        </module>

        <module name="FallThrough">
            <!-- Warn about falling through to the next case statement.  Similar to
            javac -Xlint:fallthrough, but the check is suppressed if a single-line comment
            on the last non-blank line preceding the fallen-into case contains 'fall through' (or
            some other variants that we don't publicized to promote consistency).
            -->
            <property name="reliefPattern"
                      value="fall through|Fall through|fallthru|Fallthru|falls through|Falls through|fallthrough|Fallthrough|No break|NO break|no break|continue on"/>
            <property name="severity" value="error"/>
        </module>

        <!-- Checks for over-complicated boolean expressions. -->
        <module name="SimplifyBooleanExpression"/>

        <!-- Detects empty statements (standalone ";" semicolon). -->
        <module name="EmptyStatement"/>

        <!--

        WHITESPACE CHECKS

        -->

        <module name="WhitespaceAround">
            <!-- Checks that various tokens are surrounded by whitespace.
                 This includes most binary operators and keywords followed
                 by regular or curly braces.
            -->
            <property name="tokens" value="ASSIGN, BAND, BAND_ASSIGN, BOR,
        BOR_ASSIGN, BSR, BSR_ASSIGN, BXOR, BXOR_ASSIGN, COLON, DIV, DIV_ASSIGN,
        EQUAL, GE, GT, LAND, LE, LITERAL_CATCH, LITERAL_DO, LITERAL_ELSE,
        LITERAL_FINALLY, LITERAL_FOR, LITERAL_IF, LITERAL_RETURN,
        LITERAL_SYNCHRONIZED, LITERAL_TRY, LITERAL_WHILE, LOR, LT, MINUS,
        MINUS_ASSIGN, MOD, MOD_ASSIGN, NOT_EQUAL, PLUS, PLUS_ASSIGN, QUESTION,
        SL, SL_ASSIGN, SR_ASSIGN, STAR, STAR_ASSIGN"/>
            <property name="severity" value="error"/>
        </module>

        <module name="WhitespaceAfter">
            <!-- Checks that commas, semicolons and typecasts are followed by
                 whitespace.
            -->
            <property name="tokens" value="COMMA, SEMI, TYPECAST"/>
        </module>

        <module name="NoWhitespaceAfter">
            <!-- Checks that there is no whitespace after various unary operators.
                 Linebreaks are allowed.
            -->
            <property name="tokens" value="BNOT, DEC, DOT, INC, LNOT, UNARY_MINUS,
        UNARY_PLUS"/>
            <property name="allowLineBreaks" value="true"/>
            <property name="severity" value="error"/>
        </module>

        <module name="NoWhitespaceBefore">
            <!-- Checks that there is no whitespace before various unary operators.
                 Linebreaks are allowed.
            -->
            <property name="tokens" value="SEMI, DOT, POST_DEC, POST_INC"/>
            <property name="allowLineBreaks" value="true"/>
            <property name="severity" value="error"/>
        </module>

        <module name="OperatorWrap">
            <!-- Checks that operators like + and ? appear at newlines rather than
                 at the end of the previous line.
            -->
            <property name="option" value="NL"/>
            <property name="tokens" value="BAND, BOR, BSR, BXOR, DIV, EQUAL,
        GE, GT, LAND, LE, LITERAL_INSTANCEOF, LOR, LT, MINUS, MOD,
        NOT_EQUAL, PLUS, QUESTION, SL, SR, STAR "/>
        </module>

        <module name="OperatorWrap">
            <!-- Checks that assignment operators are at the end of the line. -->
            <property name="option" value="eol"/>
            <property name="tokens" value="ASSIGN"/>
        </module>

        <module name="ParenPad">
            <!-- Checks that there is no whitespace before close parens or after
                 open parens.
            -->
            <property name="severity" value="error"/>
        </module>

        <module name="ModifierOrder"/>

    </module>
</module>
```

## 屏蔽规则配置文件举例

```xml
<?xml version="1.0"?>
<!DOCTYPE suppressions PUBLIC
        "-//Puppy Crawl//DTD Suppressions 1.1//EN"
        "https://checkstyle.org/dtds/configuration_1_3.dtd">

<suppressions>
    <!-- suppress all checks in the generated directories -->
    <suppress checks=".*" files=".+[\\/]generated[\\/].+\.java"/>
    <suppress checks=".*" files=".+[\\/]generated-sources[\\/].+\.java"/>
    <suppress checks=".*" files=".+[\\/]generated-test-sources[\\/].+\.java"/>
</suppressions>
```

# maven dependency-check

## 引入dependnecy-check插件

项目中原有的依赖是这样的

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.41.Final</version>
</dependency>
```

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>${dependency-check-maven.version}</version>
    <configuration>
        <suppressionFiles>
            <suppressionFile>src/owasp-dependency-check-suppressions.xml</suppressionFile>
        </suppressionFiles>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <msbuildAnalyzerEnabled>false</msbuildAnalyzerEnabled>
        <nodeAnalyzerEnabled>false</nodeAnalyzerEnabled>
        <yarnAuditAnalyzerEnabled>false</yarnAuditAnalyzerEnabled>
        <pyDistributionAnalyzerEnabled>false</pyDistributionAnalyzerEnabled>
        <pyPackageAnalyzerEnabled>false</pyPackageAnalyzerEnabled>
        <pipAnalyzerEnabled>false</pipAnalyzerEnabled>
        <pipfileAnalyzerEnabled>false</pipfileAnalyzerEnabled>
        <retireJsAnalyzerEnabled>false</retireJsAnalyzerEnabled>
        <msbuildAnalyzerEnabled>false</msbuildAnalyzerEnabled>
        <mixAuditAnalyzerEnabled>false</mixAuditAnalyzerEnabled>
        <nugetconfAnalyzerEnabled>false</nugetconfAnalyzerEnabled>
        <assemblyAnalyzerEnabled>false</assemblyAnalyzerEnabled>
        <skipSystemScope>true</skipSystemScope>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>aggregate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

然后可以通过`mvn clean install verify -DskipTests`来检测。这个demo下，会输出

```
[ERROR] One or more dependencies were identified with vulnerabilities that have a CVSS score greater than or equal to '7.0': 
[ERROR] 
[ERROR] netty-all-4.1.41.Final.jar: CVE-2019-16869(7.5), CVE-2021-37136(7.5), CVE-2020-11612(7.5), CVE-2021-37137(7.5), CVE-2019-20445(9.1), CVE-2019-20444(9.1), CVE-2020-7238(7.5)
[ERROR] 
[ERROR] See the dependency-check report for more details.
```

实际使用时，由于dependency-check检查相对耗时，一般通过单独的profile来控制开关


## 屏蔽CVE漏洞

如果出现`dependency-check`误报或者是评估该漏洞不涉及，可以通过`supression file`来屏蔽

### 屏蔽单一CVE漏洞

```
  <suppress>
    <notes><![CDATA[
   file name: zookeeper-prometheus-metrics-3.8.0.jar
   ]]></notes>
    <sha1>849e8ece2845cb0185d721233906d487a7f1e4cf</sha1>
    <cve>CVE-2021-29425</cve>
  </suppress>
```

### 通过文件正则来屏蔽CVE漏洞

```
    <suppress>
        <notes>CVE-2011-1797 FP, see https://github.com/jeremylong/DependencyCheck/issues/4154</notes>
        <filePath regex="true">.*netty-tcnative-boringssl-static.*\.jar</filePath>
        <cve>CVE-2011-1797g</cve>
    </suppress>
```
