---
title: adb 调试安卓
tags:
  - Android
categories: Native
keywords: 'blog,native,android'
description: adb 调试安卓
cover: >-
  https://graph.linganmin.cn/210228/3cc3f6318b2448b18a80ad415f58c9df?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210228/3cc3f6318b2448b18a80ad415f58c9df?x-oss-process=image/format,webp/quality,q_40
abbrlink: adb
date: 2021-02-20 04:30:19
---


```bash

pip3 install --pre --upgrade weditor

python3 -m weditor

adb tcpip 5555

adb connect $deviceIP

adb devices

adb kill-server

```
