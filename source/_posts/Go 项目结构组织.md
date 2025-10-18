---
title: Go 项目结构组织
link: go-project-structure
date: 2023-10-08 22:30:24
tags:
  - Go
---
## Web后端项目结构组织

要点：
- 使用`model`、`service`，而不是`modles`、`services`。差别不大，节约一个字母，更加简洁。
- 如果是企业内部的微服务，基本不会、极少把部分的功能以library的形式开放出去，internal目录在这个时候就略显鸡肋，可以省略。

备注:
- xxx、yyy代表大块的业务区分：如用户、订单、支付
- aaa、bbb代表小块的业务区分：如(用户的)登录、注册、查询

### 方案一：多业务模块通过文件名区分，不分子包

适用于小型项目

注：handler、model、service要留意方法、结构体、接口的命名，避免冲突

```
example/
|-- cmd/
|   |-- example-server/
|       |-- example-server.go (start gin app, manage handler, middleware)
|-- pkg/
|   |-- handler/
|       |-- aaa_handler.go
|       |-- bbb_handler.go
|   |-- middleware/
|       |-- aaa_middleware.go
|       |-- bbb_middleware.go
|   |-- model/
|       |-- aaa_model.go
|       |-- bbb_model.go
|   |-- service/
|       |-- aaa_service.go
|       |-- bbb_service.go
|   |-- ignite/
|       |-- ignite.go
|       |-- ignite_test.go
|   |-- influx/
|       |-- influx.go
|       |-- influx_test.go
|-- docker-build/
|   |-- scripts/
|       |-- start.sh
|-- Dockerfile
```

### 方案二：多业务模块通过包名区分，但不拆分model和service

方案二更适用于由多个小模块组合而成的项目，每个小模块不会太大，复用度较高。

```
example/
|-- cmd/
|   |-- example-server/
|       |-- example-server.go (start gin app, manage handler, middleware)
|-- pkg/
|   |-- handler/
|       |-- xxx/
|           |-- xxx_aaa_handler.go
|           |-- xxx_bbb_handler.go
|       |-- yyy/
|           |-- yyy_aaa_handler.go
|           |-- yyy_bbb_handler.go
|   |-- middleware/
|       |-- xxx/
|           |-- xxx_aaa_middleware.go
|       |-- yyy/
|           |-- yyy_bbb_middleware.go
|   |-- xxx/
|       |-- xxx_aaa_model.go
|       |-- xxx_aaa_service.go
|   |-- yyy/
|       |-- yyy_bbb_model.go
|       |-- yyy_bbb_service.go
|   |-- ignite/
|       |-- ignite.go
|       |-- ignite_test.go
|   |-- influx/
|       |-- influx.go
|       |-- influx_test.go
|-- docker-build/
|   |-- scripts/
|       |-- start.sh
|-- Dockerfile
```

### 方案三：多业务模块通过包名区分，并在下层拆分model和service

方案三更适用于由多个大模块组合而成的项目，每个大模块都很大，复用度较低，较少的互相调用。

方案三在service依赖多个service的情况下，会发生命名冲突。

```
example/
|-- cmd/
|   |-- example-server/
|       |-- example-server.go (start gin app, manage handler, middleware)
|-- pkg/
|   |-- handler/
|       |-- xxx/
|           |-- xxx_aaa_handler.go
|       |-- yyy/
|           |-- yyy_bbb_handler.go
|   |-- middleware/
|       |-- xxx/
|           |-- xxx_aaa_middleware.go
|       |-- yyy/
|           |-- yyy_bbb_middleware.go
|   |-- xxx/
|       |-- model/
|           |-- xxx_aaa_model.go
|       |-- service/
|           |-- xxx_aaa_service.go
|   |-- yyy/
|       |-- model/
|           |-- yyy_bbb_model.go
|       |-- service/
|           |-- yyy_bbb_service.go
|   |-- ignite/
|       |-- ignite.go
|       |-- ignite_test.go
|   |-- influx/
|       |-- influx.go
|       |-- influx_test.go
|-- docker-build/
|   |-- scripts/
|       |-- start.sh
|-- Dockerfile
```
