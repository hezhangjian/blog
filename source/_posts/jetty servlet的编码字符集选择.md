---
title: jetty servlet的代码字符集选择
date: 2023-06-03 01:05:07
tags:
---

记一次中文指标乱码问题，问题也很简单，如下图所示：

![Untitled](Images/20230603p1.png)

从metricbeat开始找原因，发现其实只要是UTF-8的编码格式就都可以解析，最终发现是webServer返回的数据非UTF-8格式，修改方案也很简单。将servlet中的content-type里面的**text/plain**修改成**text/plain; charset=utf-8**就可以了，如下面代码所示:

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response)
        throws IOException {
    response.setContentType("text/plain");
    response.setStatus(HttpServletResponse.SC_OK);
    response.getWriter().write("<h1>哈哈</h1>");
}
```

我们可以轻易使用一个demo来复现这个问题，在maven中添加如下依赖

```xml
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
    <version>9.4.35.v20201120</version>
</dependency>
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-servlet</artifactId>
    <version>9.4.35.v20201120</version>
</dependency>
```

```java
package com.shoothzj.jetty;

import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;

import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class SimpleJettyServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server(8080);

        ServletContextHandler context = new ServletContextHandler(ServletContextHandler.SESSIONS);
        context.setContextPath("/");
        server.setHandler(context);

        context.addServlet(new ServletHolder(new HelloDefaultServlet()), "/hello-default");
        context.addServlet(new ServletHolder(new HelloUTF8Servlet()), "/hello-utf8");

        server.start();
        server.join();
    }

    public static class HelloDefaultServlet extends HttpServlet {
        @Override
        protected void doGet(HttpServletRequest request, HttpServletResponse response)
                throws IOException {
            response.setContentType("text/plain");
            response.setStatus(HttpServletResponse.SC_OK);
            response.getWriter().write("<h1>哈哈</h1>");
        }
    }

    public static class HelloUTF8Servlet extends HttpServlet {
        @Override
        protected void doGet(HttpServletRequest request, HttpServletResponse response)
                throws IOException {
            response.setContentType("text/plain; charset=UTF-8");
            response.setStatus(HttpServletResponse.SC_OK);
            response.getWriter().write("<h1>哈哈</h1>");
        }
    }
}
```

通过curl命令来复现这个问题

```bash
curl localhost:8080/hello-default
<h1>??</h1>%
curl localhost:8080/hello-utf8
<h1>哈哈</h1>%
```

那么servlet里面的数据如何编码，我们可以dive一下，首先servlet里面有一个函数叫**`response.setCharacterEncoding();`**这个函数可以指定编码格式。其次，servlet还会通过上面的setContentType函数来做一定的推断，比如content-type中携带了charset，就使用content-type中的charset。还有些特定的content-type，比如text/json，在没有设置的情况下，servlet容器会假设它使用utf-8编码。在推断不出来，也没有手动设置的情况下，jetty默认的编码是iso-8859-1，这就解释了乱码的问题。
