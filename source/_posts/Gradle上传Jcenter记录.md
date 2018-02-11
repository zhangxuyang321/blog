---
title: Gradle上传Jcenter记录
date: 2017-02-13 17:54:24
tags: Android工具使用
categories: Android开发工具
---

本文忽略了申请Jcenter账号等一些问题,只是详细记录了在Androidstudio中的配置.

***
## gradle配置
### 项目的build.gradle中
在dependencies中添加如下代码中的三个classpath

```Groovy
dependencies {
       .......
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.2'
        classpath "org.jfrog.buildinfo:build-info-extractor-gradle:3.1.1"
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.4.1'
    }
```

<!-- more -->

### library的build.gradle中
在需要上传的library的build.gradle中添加如下脚本

```Groovy
android {
.......
}
dependencies {
........
}
ext {
    bintrayRepo = 'maven'  //maven包使用maven管理项目,固定值
    bintrayName = 'step'  //项目名称

    publishedGroupId = 'com.xyz.step' //library包名
    libraryName = 'step'  //library名称
    artifact = 'step'  //类库的名称
    libraryDescription = '流程指示器' //项目描述
    siteUrl = 'https://github.com/zhangxuyang321/StepView'     //项目地址
    gitUrl = 'https://github.com/zhangxuyang321/StepView.git'  //项目的版本控制

    libraryVersion = '1.0.0'  //当前版本

    developerId = 'zhangxy'  //填的是jcenter上的用户名
    developerName = 'xyz'   //开发信息,开发者用户名
    developerEmail = 'zhangxuyang321@gmail.com'  //开发者邮箱

// 开源协议
    licenseName = 'The Apache Software License, Version 2.0'      
    licenseUrl = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
    allLicenses = ["Apache-2.0"]
}

//需要配置的插件
apply from: 'https://raw.githubusercontent.com/nuuneoi/JCenter/master/installv1.gradle'
apply from: 'https://raw.githubusercontent.com/nuuneoi/JCenter/master/bintrayv1.gradle'
```

## local.properties中配置

```Groovy
bintray.user=zhangxy  //jcenter 用户名
bintray.apikey=xxxxxxxxxxxxxxxxxxxxxxxx //jcenter API Key
```


## Gradle命令

```Groovy
./gradlew install
./gradlew bintrayUpload
```
