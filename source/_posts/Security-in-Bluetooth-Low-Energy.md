---
title: Security in Bluetooth Low Energy
date: 2023-04-26 21:55:20
categories: Bluetooth
tags: 
- BLE
- Security
---

> 本文介绍BLE中安全相关的内容。

<!--more-->

### 1 Overview

#### 1.1 Referring to devices

蓝牙核心规范使用不同的属于描述通信中的两个设备，使用什么属于需要根据语境来确定。概括如下表：

| 术语                    | 语境                                            |
| ----------------------- | ----------------------------------------------- |
| initiator and responder | 通常用在安全相关交互的语境中，如pairing         |
| master and slave        | BLE链路层相关问题中通常将它们成为主设备和从设备 |
| peripheral and central  | GAP中通常称之为外围设备和中心设备               |
| client and server       | GATT和ATT中通常成为客户端和服务器               |

#### 1.2 Security Levels and Models

安全等级和模式定义了”设备，服务和服务请求“的安全要求。

- model 1
  - L1 没有安全要求
  - L2 Unauthenticated pairing with encryption 未经认证的加密配对
  - L3 Authenticated pairing with encryption 经认证的加密配对
  - L4 经认证的LE 安全链接与使用128为强度加密密钥的加密配对

- model 2
  - L1 Unauthenticated pairing with data signing 未经认证的数据签名配对
  - L2 Authenticated pairing with data signing 未经认证的数据签名配对

- model 3
  - L1 没有安全要求
  - L2 使用未经身份认证的 Broadcast_Code
  - L3 使用经过认证的 Broadcast_Code