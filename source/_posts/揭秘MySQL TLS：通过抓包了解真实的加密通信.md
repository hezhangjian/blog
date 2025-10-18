---
title: 揭秘MySQL TLS：通过抓包了解真实的加密通信
link: mysql-default-tls
date: 2023-02-09 08:21:43
tags:
  - MySQL
---

你的mysql客户端和服务端之间开启tls了吗？你的回答可能是No，我没有申请证书，也没有开启mysql客户端，服务端的tls配置。

可是当你抓取了3306 mysql的端口之后，你会发现，抓出来的包里居然有Client Hello、Server Hello这样的典型TLS报文。

![mysql-tls-pcap](Images/mysql-tls-pcap.png)

其实，Mysql的通信是否加密，是由客户端和服务端共同协商是否开启的，客户端与服务端都处于默认配置下的话，有些类似于StartTls。

## 服务端侧

在连接建立时，Mysql服务端会返回一个**Server Greeting**，其中包含了一些关于服务端的信息，比如协议版本、Mysql版本等等。在其中有一个flag的集合字段，名为
**Capabilities Flag**，顾名思义，这就是用来做兼容性，或者说特性开关的flag，大小为2个字节，其中的第12位，代表着**CLIENT_SSL**
，如果设置为1，那代表着如果客户端具备能力，服务端可以在后面的会话中切换到TLS。可以看到里面还有一些其他的flag，事务、长密码等等相关的兼容性开关。

![mysql-pcap-capabilities-flag](Images/mysql-pcap-capabilities-flag.png)

我们可以测试一下设置为0的行为，只需要在my.cnf中添加

```bash
echo "ssl=0" >> /etc/my.cnf
```

重启mysql。再度进行抓包，就发现没有tls的报文了，都是在使用明文进行通信了。

![mysql-plain-pcap](Images/mysql-plain-pcap.png)

## 客户端侧

这个协商过程也可以在客户端进行控制，客户端对应的参数是sslMode，可以设置为DISABLED、PREFERRED、REQUIRED、VERIFY_CA、VERIFY_IDENTITY，分别代表不使用ssl、优先使用ssl、必须使用ssl、验证CA、验证身份。默认的行为是PREFERRED，example:

比如配置sslMode为DISABLED，那么客户端就不会使用ssl进行通信，而是使用明文。

```text
r2dbc:mysql://localhost:3306/test?sslMode=DISABLED
```

![mysql-client-disable-tls](Images/mysql-client-disable-tls.png)

## 总结

| 客户端             | 服务端          | 结果             |
|-----------------|--------------|----------------|
| DISABLED        | ssl=0        | PLAIN          |
| DISABLED        | ssl=1        | PLAIN          |
| PREFERRED       | ssl=0        | PLAIN          |
| PREFERRED       | ssl=1        | TLS            |
| REQUIRED        | ssl=0        | Fail           |
| REQUIRED        | ssl=1        | TLS            |
| VERIFY_CA       | ssl=0        | Fail           |
| VERIFY_CA       | ssl=1 + CA配置 | TLS，客户端验证证书    |
| VERIFY_IDENTITY | ssl=0        | Fail           |
| VERIFY_IDENTITY | ssl=1 + CA配置 | TLS，客户端验证证书和域名 |

注：

- VERIFY_CA：确保服务器证书由受信任的CA签发，但不验证证书的主机名或IP地址。
- VERIFY_IDENTITY：不仅验证证书的CA签发，还额外验证证书的主机名或IP地址与服务器的实际地址是否一致。
