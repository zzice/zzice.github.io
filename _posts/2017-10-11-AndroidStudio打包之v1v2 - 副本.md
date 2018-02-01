---
title: Android Studio 打包时 Signature Version V1 V2
date: 2017-08-13 23:10:01
categories: 
tags: Android_Studio
---

> Android 7.0中引入了APK Signature Scheme v2，v1是jar Signature来自JDK
> V1：通过ZIP条目进行验证，这样APK签署后可进行许多修改，可以移动甚至重新压缩文件。
> V2：验证压缩文件的所有字节，而不是单个ZIP条目，因此，在签名后无法再更改(包括zipalign)。正因如此，现在在编译过程中，我们将压缩、调整和签署合并成一步完成。好处显而易见，更安全而且新的签名可缩短在设备上进行验证的时间（不需要费时地解压缩然后验证），从而加快应用安装速度。

<!-- more -->

![](http://zzice.github.io/resoures/images/Signature_Version_V1_V2.png)

### 解决方案一 ###
v1和v2的签名使用
只勾选v1签名并不会影响什么，但是在7.0上不会使用更安全的验证方式
只勾选V2签名7.0以下会直接安装完显示未安装，7.0以上则使用了V2的方式验证
同时勾选V1和V2则所有机型都没问题

### 解决方案二 ###
在app的build.gradle的android标签下加入如下

``` gradle 
signingConfigs {  
    debug {  
        v1SigningEnabled true  
        v2SigningEnabled true  
    }  
    release {  
        v1SigningEnabled true  
        v2SigningEnabled true  
    }  
}  
```
