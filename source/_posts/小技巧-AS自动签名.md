---
title: 小技巧--AS自动签名
date: 2016-12-30 10:23:30
tags:
---
以前eclipse上签名打包是一件很麻烦的事，普通测试也就算了，但是一些第三方必须要你签了名才能使用他们的服务，这样一来就很麻烦，每次签名都要浪费很久的时间。我不知道eclipse支不支持快速签名，但是现在用AS确实方便了很多，配置也很简单。
<!--more-->
找到app/build.gradle文件，在android节点下面添加signingConfigs字段，配置签名信息，然后在buildTypes节点里使用配置的签名信息即可。代码如下：
```gradle
android {
    signingConfigs {
        config {
            keyAlias '别名'
            keyPassword '密码'
            storeFile file('证书存放地址，如：C:/helloWorld.jks')
            storePassword '证书密码'
        }
    }
    compileSdkVersion 23
    buildToolsVersion "23.0.0"

    ······
    
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.config
        }
        debug {
            signingConfig signingConfigs.config
        }
    }
}
```

## 有问题反馈
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* Email: 2563892038@qq.com
* Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)