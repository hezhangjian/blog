---
title: TypeScript Http SDK设计
link: typescript-http-sdk-design
date: 2023-10-28 16:52:15
tags:
  - SDK
  - TypeScript
---

TypeScript的调用方式通常是

```javascript
import { ClassName } from 'moduleName';
const object = new ClassName();
```

根据TypeScript项目的需求和特性，可以为TypeScript的Http SDK项目选择以下命名方式：

- `xxx-client-ts`：如果这个项目只有Http SDK，没有其他协议的SDK，推荐使用这个命名方式。在npm可以注册为"xxx"。
- `xxx-http-client-ts`：当存在其他协议的SDK时，可以使用这个命名方式，以区分不同协议的SDK。
- `xxx-admin-ts`：当项目使用其他协议作为数据通道，使用HTTP协议作为管理通道时，可以使用这个命名方式。
