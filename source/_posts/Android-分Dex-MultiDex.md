---
title: Android 分Dex (MultiDex)
date: 2016-12-29 16:37:33
tags:
---
## 前言
 辛苦了一阵，博客也终于搭好了，现在我也要学着写写博客，一来给自己以后查找资料方便；二来也练一练文笔，现在一打字、一拿笔，都不知道说些什么了。
## 正文
 昨天把一个功能导进一个老项目中，发现两个项目很多东西都不一样，比如网络库、消息总线、图片加载库等等，很多东西能重用的我就重用了，但还是有很多是新加的，瞎忙了两个小时，把报错的都修改了。但是当我满怀希望的时候，发现gradle build报错：
<!--more-->
```java
Unable to execute dex: method ID not in [0, 0xffff]: 65536
Conversion to Dalvik format failed: Unable to execute dex: method ID not in [0, 0xffff]: 65536
```
 方法数超标了，在ART以前的Android系统中，Dex文件对于方法索引是用一个short类型的数据来存放的。而short的最大值是65535，因此当项目足够大包含方法数目足够多超过了65535(包括引用的外部Lib里面的所有方法)，当运行App就会得到这个错误提示。
## 解决方案
 1. 修改app/build.gradle
		
  在defaultConfig里添加
```
multiDexEnabled true
```
  在defaultConfig同级添加
```
dexOptions {
		preDexLibraries = false
		javaMaxHeapSize "2g"
	}
```
  在dependencies里添加
```
compile 'com.android.support:multidex:1.0.1'
```
  ### 注意
   这一步可能会遇到两个问题
    * gradle版本可能低了，在项目的build.gradle里面把版本提高，比如我这样：
```
classpath 'com.android.tools.build:gradle:2.1.0'
```
    * 在获取multidex的时候可能一直卡在这个地方，原因是被墙了，请自备梯子。

 2. 修改Application

  如果没有自定义Application就直接在AndroidManifest.xml中修改：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	package="com.example.android.multidex.myapplication">
	<application
		...
		android:name="android.support.multidex.MultiDexApplication">
		...
	</application>
</manifest>
```
  如果有自己自定义Application，就去修改自己的Application
   * 要么重写attachBaseContext方法：
```java
 public class HelloMultiDexApplication extends Application {
	@Override
	public void onCreate() {
		super.onCreate();
	}

	@Override
	protected void attachBaseContext(Context base) {
		super.attachBaseContext(base);
		MultiDex.install(this);
	}
}
```
   * 要么继承MultiDexApplication
```java
public class HelloMultiDexApplication extends MultiDexApplication {
	@Override
	public void onCreate() {
		super.onCreate();
	}
}
```
 


## 有问题反馈
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* Email: 2563892038@qq.com
* Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)