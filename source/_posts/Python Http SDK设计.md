---
title: Python Http SDK设计
link: python-http-sdk-design
date: 2023-10-28 16:52:15
tags:
  - Python
  - SDK
---

根据Python项目的需求和特性，可以为Python的Http SDK项目选择以下命名方式：

- `xxx-client-python`：如果这个项目只有Http SDK，没有其他协议的SDK，推荐使用这个命名方式。
- `xxx-http-client-python`：当存在其他协议的SDK时，可以使用这个命名方式，以区分不同协议的SDK。
- `xxx-admin-python`：当项目使用其他协议作为数据通道，使用HTTP协议作为管理通道时，可以使用这个命名方式。

由于Python的调用方式通常是`模块名.类名.方法名`。
