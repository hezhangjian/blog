---
title: Go Http SDK设计
link: go-http-sdk-design
date: 2023-10-28 15:59:26
tags:
  - Go
  - SDK
---

根据Go项目的需求和特性，可以为Go的Http SDK项目选择以下命名方式：

- `xxx-client-go`：如果这个项目只有Http SDK，没有其他协议的SDK，推荐使用这个命名方式。
- `xxx-http-client-go`：当存在其他协议的SDK时，可以使用这个命名方式，以区分不同协议的SDK。
- `xxx-admin-go`：当项目使用其他协议作为数据通道，使用HTTP协议作为管理通道时，可以使用这个命名方式。

由于Go语言的调用方式是`包名.结构体名.方法名`，所以在设计SDK时，需要考虑包名、结构体名、方法名的设计。

以xxx业务为例，假设业务名为`xxx`，推荐包名也为`xxx`，结构体名为`Client`。

目录布局可以是这样子的：

```
xxx-client-go/
|-- xxx/
|   |-- client.go
```
