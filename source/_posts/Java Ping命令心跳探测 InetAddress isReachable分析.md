---
title: Java Ping命令心跳探测 InetAddress isReachable分析
link: java-ping
date: 2020-01-12 09:35:09
tags:
  - Java
---

# 业务需求分析与解决方案

在业务场景中，当需要利用ping命令对主机进行心跳探测时，直接在代码中fork进程执行ping命令虽然可行，但这种方法开销较大，并且处理流程易出错，与使用标准库相比缺乏优雅性。因此，本文探讨了使用Java的InetAddress类的isReachable方法作为替代方案。

根据资料指出，Java的InetAddress类在root用户权限下通过执行ping命令进行探测，在非root用户权限下则通过访问TCP端口7进行探测。为验证这一点，本文撰写了相应的demo代码并进行了测试（详见：[GitHub - heart-beat](https://github.com/hezhangjian/heart-beat)）。

```
import lombok.extern.slf4j.Slf4j;

import java.net.InetAddress;

@Slf4j
public class PingTestMain {

    public static void main(String[] args) throws Exception {
        String testIp = System.getProperty("TestIp");
        InetAddress inetAddress = InetAddress.getByName(testIp);
        boolean addressReachable = inetAddress.isReachable(500);
        log.info("address is reachable is {}", addressReachable);
    }

}
```
# 测试实验

## root用户下执行程序

![java-ping-root-success.png](Images/java-ping-root-success.png)

java程序打印结果也是true

## 在普通用户权限下的测试

![java-ping-user-fail.png](Images/java-ping-user-fail.png)

此时可以看到,我们的客户端程序向目标tcp7端口发送了一个报文,虽然java程序打印结果为true,但是因为收到了RST包导致的.在当今的网络安全要求下,7端口往往不会开放

## 在目标网络屏蔽TCP端口7的情况下执行程序

```
iptables -A INPUT -p tcp --dport 7 -j DROP
```

发送的报文没有收到RST包,此时java程序返回false.不是我们预期的结果

## 普通用户权限下携带特权的测试

进一步的研究发现，Java程序发送ping命令需要创建raw socket，这要求程序具有root权限或cap_net_raw权限。赋予Java程序创建raw socket的权限后重新测试，发现程序能够正确发送ping命令，达到预期效果。

```
setcap cap_net_raw+ep /usr/java/jdk-13.0.1/bin/java

```
发现如下报错
```
java: error while loading shared libraries: libjli.so: cannot open shared object file: No such file or directory
```
使用https://askubuntu.com/questions/334365/how-to-add-a-directory-to-linker-command-line-in-linux规避添加so文件权限

随后抓包,发现还是发送了ping命令,达到了我们预期的效果

# 总结

本文通过一系列测试得出结论，root用户权限下的Java程序会使用ping命令进行探测。若普通用户不具备相应权限，则会尝试探测TCP端口7，但在安全组未开启该端口的情况下会导致预期结果不一致。推荐赋予java程序特权,使得InetAddress类能够使用ping命令进行探测
