---
title: Spring记录数据库操作时间的几种方式
link: spring-database-audit-time
date: 2023-12-15 23:22:17
tags:
  - Java
  - Spring
---

Spring记录数据库操作时间的几种方式

# Spring Jpa

@EnableJpaAuditing注解开启Jpa的审计功能，然后在实体类上使用@CreatedDate和@LastModifiedDate注解即可

```java
    @Column(name = "create_time")
    @CreatedDate
    private LocalDateTime createTime;

    @Column(name = "update_time")
    @LastModifiedDate
    private LocalDateTime updateTime;
```

# Spring R2dbc

Spring R2dbc可以使用@CreatedDate和@LastModifiedDate注解来实现。但是需要在Application上开启`@EnableR2dbcAuditing`

```java
    @Column("created_time")
    @CreatedDate
    private LocalDateTime createdTime;

    @Column("updated_time")
    @LastModifiedDate
    private LocalDateTime updatedTime;
```

# 应用程序修改

应用程序修改就比较简单，简单设置一下即可,以PersonPo类为例

```java
PersonPo personPo = new PersonPo();
personPo.setCreateTime(LocalDateTime.now());
personPo.setUpdateTime(LocalDateTime.now());
```

# Mysql场景下利用TIMESTAMP能力

```sql
CREATE TABLE person (
    id INT PRIMARY KEY,
    // ... 其他字段 ...
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
