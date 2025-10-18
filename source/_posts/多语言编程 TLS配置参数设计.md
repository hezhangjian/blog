---
title: 多语言编程 TLS配置参数设计
link: multi-lang-paradigm-tls-config-design
date: 2023-11-06 10:11:35
tags:
  - Code
  - 多语言编程
---

# 背景

TLS(Transport Layer Security)是一种安全协议，用于在两个通信应用程序之间提供保密性和数据完整性。TLS是SSL(Secure Sockets Layer)的继任者。

不同的编程语言处理TLS配置的方式各有千秋, 本文针对TLS配置参数的设计进行探讨。

代码配置中，建议使用反映状态的参数名。

# 通用参数

- tlsEnable: 是否启用TLS

# Go

推荐使用方式一

方式一：

- tlsConfig *tls.Config: Go标准库的内置TLS结构体

方式二：

由于Go不支持加密的私钥文件，推荐使用文件内容，而不是文件路径，避免敏感信息泄露。

- tlsCertContent []byte: 证书文件内容
- tlsPrivateKeyContent []byte: 私钥文件内容
- tlsMinVersion uint16: TLS最低版本
- tlsMaxVersion uint16: TLS最高版本
- tlsCipherSuites []uint16: TLS加密套件列表

# Java

Java的TLS参数基本上都是基于keystore和truststore来配置的。一般常见设计如下参数：

- keyStorePath: keystore文件路径
- keyStorePassword: keystore密码
- trustStorePath: truststore文件路径
- trustStorePassword: truststore密码
- tlsVerificationDisabled: 是否禁用TLS校验
- tlsHostnameVerificationDisabled: 是否禁用TLS主机名校验，仅部分框架支持。
- tlsVersions: TLS版本列表
- tlsCipherSuites: TLS加密套件列表

# JavaScript

JavaScript可以使用标准库里的tls.SecureContextOptions


# Kotlin

kotlin的Tls与Java相同：

- keyStorePath: keystore文件路径
- keyStorePassword: keystore密码
- trustStorePath: truststore文件路径
- trustStorePassword: truststore密码
- tlsVerificationDisabled: 是否禁用TLS校验
- tlsHostnameVerificationDisabled: 是否禁用TLS主机名校验，仅部分框架支持。
- tlsVersions: TLS版本列表
- tlsCipherSuites: TLS加密套件列表

# Python

推荐使用方式一

方式一

- ssl.SSLContext: Python标准库的内置TLS结构体

方式二

Python可以使用文件路径以及加密的私钥文件。

- tlsCertPath: 证书文件路径
- tlsPrivateKeyPath: 私钥文件路径
- tlsPrivateKeyPassword: 私钥密码
- tlsMinVersion: TLS最低版本
- tlsMaxVersion: TLS最高版本
- tlsCipherSuites: TLS加密套件列表

# Rust

由于常见的Rust TLS实现不支持加密的私钥文件，推荐使用文件内容，而不是文件路径，避免敏感信息泄露。 一般常见如下设计参数:

- tls_cert_content Vec<u8>: 证书内容
- tsl_private_key_content Vec<u8>: 私钥内容
- tls_versions: TLS版本列表
- tls_cipher_suites: TLS加密套件列表
- tls_verification_disabled: 是否禁用TLS校验
