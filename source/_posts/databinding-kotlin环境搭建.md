---
title: databinding+kotlin环境搭建
date: 2017-03-20 18:23:31
tags: [kotlin,databinding]
categories: Android
---

## 前言

kotlin是一个基于 JVM 的新的编程语言。它有很多的特性可以使你的code写的很爽，它可以兼容java和java混合开发。

Data Binding是一个实现数据和UI绑定的框架，是一个实现 MVVM 模式的工具，有了 Data Binding，在Android中也可以很方便的实现MVVM开发模式。

<!--more-->

## 配置

### 项目级的build.gradle

```groovy
buildscript {
    ext.kotlin_version = '1.1.0'
    ext.gradle_version = '2.2.3'
    repositories {
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:$gradle_version"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath "org.jetbrains.kotlin:kotlin-android-extensions:$kotlin_version"
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

### modle级的build.gradle

```groovy
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-kapt'
apply plugin: 'kotlin-android-extensions'

android {
  	
  	...
    
    dataBinding{
        enabled true
    }
    sourceSets {
        main.java.srcDirs += 'src/main/kotlin'
    }
}

kapt {
    generateStubs = true
}

dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    kapt "com.android.databinding:compiler:$gradle_version"
}
```



## 有问题反馈

在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

- Email: 2563892038@qq.com
- Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)