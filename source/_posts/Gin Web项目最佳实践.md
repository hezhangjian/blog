---
title: Gin Web项目最佳实践
link: gin-project-best-practice
date: 2024-10-30 05:58:14
tags:
  - Gin
  - Go
---

本文包含，Gin项目推荐布局，一些最佳实践等等。

## Gin项目推荐布局

假设项目名称叫Hearth

- xxx、yyy代表大块的业务区分：如用户、订单、支付
- aaa、bbb代表小块的业务区分：如(用户的)登录、注册、查询

```
example/
|-- cmd/
|   |-- production/
|       |-- hearth.go
|   |-- local/
|       |-- hearth_local.go
|-- pkg/
|   |-- apimodel/ 存放所有的ApiModel，用oapi-codegen解析uadp yaml来生成
|   |-- boot/
|       |-- boot.go //装备Struct，用于Lauch整个项目
|   |-- handler/
|       |-- xxx/
|           |-- xxx_aaa_handler.go
|           |-- xxx_bbb_handler.go
|       |-- yyy/
|           |-- yyy_model.go
|           |-- yyy_aaa_handler.go
|           |-- yyy_bbb_handler.go
|   |-- xxx/
|       |-- xxx_aaa_model.go // 存放持久化model，如数据库表，消息中间件结构，redis结构等
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

## 放弃的布局方式

### 此种布局比较适合独立的包，对api结构体的操作复用较差

```
example/
|-- cmd/
|   |-- production/
|       |-- hearth.go
|   |-- local/
|       |-- hearth_local.go
|-- pkg/
|   |-- boot/
|       |-- boot.go //装备Struct，用于Lauch整个项目
|   |-- handler/
|       |-- xxx/
|           |-- xxx_model.go // 将大块业务的model也放在这里，可以使用oapi-codegen来生成结构体
|           |-- xxx_aaa_handler.go
|           |-- xxx_bbb_handler.go
|       |-- yyy/
|           |-- yyy_model.go
|           |-- yyy_aaa_handler.go
|           |-- yyy_bbb_handler.go
|   |-- xxx/
|       |-- xxx_aaa_model.go // 存放持久化model，如数据库表，消息中间件结构，redis结构等
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
