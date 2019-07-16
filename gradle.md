# 1. 前言
Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化建构工具。它使用一种基于Groovy的特定领域语言(DSL Domain-Specific-Language)来声明项目设置，而不是传统的XML。当前其支持的语言限于Java、Groovy和Scala，计划未来将支持更多的语言。Gradle可以帮你管理项目中的差异,依赖,编译,打包,部署...,你可以定义满足自己需要的构建逻辑,写入到build.gradle中供日后复用，通俗的说：gradle是打包用的。

## 1.1 配置Gradle环境
配置Gradle环境前，确保已经安装配置好Java环境，下载Gradle后解压并配置环境变量，具体可以参考-[配置Gradle](https://blog.csdn.net/u010316188/article/details/84257383)
```java
// Gradle版Hello word
// 新建build.gradle文件：
task hello{	// 定义一个任务Task名为hello
    doLast{ // 添加一个动作Action，表示在Task执行完毕后回调doLast闭包中的代码
        println'Hello World'//输出字符串，单双号均可
    }
}
// 命令行：
gradle hello // 执行build.gradle中名为Hello的任务
// 输出：
Hello World
```
## 1.2 Groovy基础
Groovy是个灵活的动态脚本语言，基于JVM虚拟机的一种动态语言，语法和Java很相似，又兼容Java，且在此基础上增加了很多动态类型和灵活的特性，如支持闭包和DSL。
具体语言特性教程可以参考-[Groovy教程](https://www.w3cschool.cn/groovy/)

# 2. Android Gradle脚本
一个Android Studio和Gradle的项目目录如下：
![Android Studio项目Gradle](http://image.huawei.com/tiny-lts/v1/images/096a5262a60ff67d226a_399x508.png@900-0-90-f.png)
我们先来看看Android Gradle项目中那些涉及到gradle的文件分别是什么意思。
## 2.1 Gradle wrapper
```xml
|--gradle
|   |--wrapper
|        |--gradle-wrapper.jar
|        |--gradle-wrapper.properties
|--gradlew
|--gradlew.bat
```
wrapper顾名思义是对Gradle的一层包装，上面目录中`gradlew`和`gradlew.bat`分别是Linux和Windows下的可执行脚本，`gradle-wrapper.jar`是具体业务逻辑实现的jar包，`gradlew`最终还是使用Java执行的这个jar包来执行相关`Gradle`操作，`gradle-wrapper.properties`是配置文件，用于配置使用哪个版本的Gradle，其包含的配置文件如下图所示：
![gradle-wrapper.properties](http://image.huawei.com/tiny-lts/v1/images/abab7262a61127ea6239_993x271.jpg@900-0-90-f.jpg)
配置举例：
```java
#Wed Jun 19 10:09:08 GMT+08:00 2019
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-5.1.1-all.zip
```
## 2.2 Settings.gradle 多项目构建
此文件用于初始化以及工程树的配置，大多数用于配置子工程，在Gradle中多个工程是通过工程树来表示的，相当于我们在Android Studio看到的Project和Module概念一样，根工程相当于Project，子工程相当于Module，一个Project可以有很多Module，一个子工程只有在`Setting.gradle`中配置了才会生效。
配置举例：
```java
// 添加:app和:common这两个module参与构建
include ':app' 
project(':app').projectDir = new File('存放目录')
include':common'
project(':common').projectDir = new File('存放目录')
```
如果不指定上述存放目录，则默认为是`Settings.gradle`其同级目录。

## 2.3 Build文件
每个Project都会有一个Build文件，该文件是该Project的构建入口，在此文件中可以对该Project进行配置，如配置版本，插件，依赖库等。
既然每个Project都有一个Build文件，那么Root Project也不例外，在Root Project中可以对子Module进行统一配置。
> Project的build.gradle：整个Project的共有属性，包括配置版本、插件、依赖库等信息
> Module的build.gradle：各个module私有的配置文件
### 2.3.1 Project_build.gradle
```java
buildscript {
    // gradle脚本执行所需依赖仓库
    repositories {
        google()
        jcenter()
        
    }
    // gradle脚本执行所需依赖
    dependencies {
        classpath 'com.android.tools.build:gradle:3.4.1'
    }
}

allprojects {
    // 项目本身需要的依赖仓库
    repositories {
        google()
        jcenter()
        
    }
}
```
那么buildscript中的repositories和allprojects的repositories的作用和区别是什么呢？
1. buildscript里是gradle脚本执行所需依赖，分别是对应的maven库和插件
2. allprojects里是项目本身需要的依赖，比如我现在要依赖maven库的xx库，那么我应该将maven {url '库链接'}写在这里，
而不是buildscript中，不然找不到。

> 参考 作者：CalvinNing -[原文链接](https://www.jianshu.com/p/ee57e4de78a3)
# 3. Gradle插件
## 3.1 插件介绍
Gradle的设计非常好，本身提供一些基本的概念和整体核心的框架，其他用于描述真实使用场景逻辑的都以插件扩展的方式来实现，比如构建Java应用，就通过Java插件来实现，那么自然构建Android应用，就通过Android Gradle插件来实现。Android Gradle插件是基于内置的Java插件实现的。Gradle插件的作用如下：
1. 可以添加任务到项目，比如测试、编译、打包等
2. 可以添加依赖配置到项目，帮助配置项目构建过程中需要的依赖，比如第三方库等
3. 可以向项目中现有的对象类型添加新的扩展属性和方法等，帮助配置和优化构建
4. 可以对项目进行一些约定，比如约定源代码存放位置等
## 3.2 插件种类及用法
### 3.2.1 插件种类
- 二进制插件：实现了org.gradle.api.Plugin接口的插件，可以有plugin id
- 脚本插件：严格上只是一个脚本，可以来自本地或网路
### 3.2.2 插件用法
通过`Project#apply()`方法，有三种用法:Map参数：void apply (Map<String, ?> options)
- 二进制插件：
> id：apply plugin:'java'
> 类型：apply plugin:org.gradle.api.plugins.JavaPlugin
> 简写：apply plugin:JavaPlugin
- 脚本插件：apply from:'version.gradle'
- 第三方发布插件：apply plugin:'com.android.application'

## 3.3 Android Gradle插件
从Gradle的角度看，Android其实就是Gradle的一个第三方插件，它是由Google的Android团队开发的。
### 3.3.1 插件分类
1. App应用工程：生成可运行apk应用；id: com.android.application
2. Library库工程：生成AAR包给其他的App工程公用；id: com.android.library
3. Test测试工程：对App应用工程或Library库工程进行单元测试；id: com.android.test
### 3.3.2 应用Android Gradle插件
主要来看下Android Gradle的build.gradle配置文件：
```java
// 插件id
apply plugin:'com.android.application'
// 自定义配置入口，后续详解
android{
    compileSdkVersion 23 // 编译Android工程的SDK版本
    buildToolsVersion "23.0.1" // 构建Android工程所用的构建工具版本
	
    defaultConfig{
        applicationId "com.example.myapplication" // 配置包名
        minSdkVersion 14 // 最低支持的Android系统的Level
        targetSdkVersion 23 // 表示基本哪个Android版本开发
        versionCode 1 // APP应用内部版本名称
        versionName "1.0" // APP应用的版本名称
    }
    buildTypes{
        release{ // 构建类型
            minifyEnabled false // 是否启用混淆
            proguardFiles getDefaultPraguardFile('proguard-andrcid.txt'), 'proguard-rules.pro' // 配置混淆文件
        }
    }
}
// 配置第三方依赖
dependencies{
   implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.0.2'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test:runner:1.2.0'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
}
```
android{}是Android Gradle插件提供的一个扩展类型，可以让我们自定义Android Gradle工程。defaultConfig是默认的配置，是一个ProductFlavor(构建渠道)，ProductFlavor允许我们根据不同的情况同时生成不同的APK包。buildTypes是一个NamedDomainObjectContainer类型，是一个域对象，可以在buildTypes{}里新增任意多个我们
需要构建的类型，比如debug。
> NamedDomainObjectContainer具体可以参考-[NamedDomainObjectContainer详解](https://www.jianshu.com/p/167cd4b82653)
### 3.3.3 多渠道构建
由于发布或者推广APP的渠道不同，就造成了Android APP可能会有很多个，所以需要针对不同的渠道做不同的处理。
在Android Gradle中，定义了一个叫Build Variant(构建变体/构建产物)的概念，一个构建变体（Build Variant）=构建类型（Build Type）+构建渠道（Product Flavor）。
> - Build Type有release、debug两种构建类型
> - Product Flavor有baidu、google两种构建渠道
> - Build Variant有baiduRelease、baiduDebug、googleRelease、googleDebug四种构件产出
配置好发布渠道后，Android1 Gradle就会生成很多Task，比如assembleBaidu，assembleRelease，assembleBaiduRelease等
- assemble开头的负责生成构件产物(Apk)
> assembleBaidu：运行后会生成baidu渠道的release和debug包
> assembleRelease：运行后会生成所有渠道的release包
> assembleBaiduRelease：运行后只会生成baidu的release包
