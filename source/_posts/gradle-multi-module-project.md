---
title: 使用Gradle构建多项目
date: 2021-06-19 10:19:26
categories:
- Package Manager
---

从创建一个最简单的gradle项目开始, 然后介绍multi module项目

# Creating simplest Gradle project for Java
以创建一个Redis Client为例, 引入一个第三方库Jedis依赖.

1. 首先创建Root Project, 即创建一个文件夹, 添加`settings.gradle`文件并编辑
```gradle
rootProject.name = 'simplest-gradle-project'

```
2. 在root目录下, 创建gradle wrapper的必要文件, `gradlew`, `gradlew.bat`, `gradle`(文件夹), 这些文件可以通过 Spring Initializr 新建一个项目得到.
3. 在每个subproject下创建`build.gradle`文件, 如下内容(示例)
```gradle
plugins {
	id 'java'
}

group = 'org.demo'
version = '0.0.1-SNAPSHOT'

repositories {
    maven {
      url 'https://maven.aliyun.com/repository/public/'
    }
    maven {
      url 'https://maven.aliyun.com/repository/spring/'
    }
    mavenLocal()
    mavenCentral()
}

dependencies {
	// implementation 'redis.clients:jedis:jedis-3.6.2'
}
```
4. 如果通过gradle task创建src文件目录(在build.gradle文件中添加下述代码), 然后运行 `./gradlew :createDirs`, 或者直接通过IDEA的gradle插件, 双击`simplest-gradle-project > Tasks > other > createDirs`
```gradle
// 创建缺失的src目录
task createDirs {
    sourceSets*.java.srcDirs*.each{
        it.mkdirs()
    }
    sourceSets*.resources.srcDirs*.each{
        it.mkdirs()
    }
}
```

参考官方文档整理而来,
https://spring.io/guides/gs/multi-module/

# Creating a Multi Module Project
1. 首先创建Root Project, 即创建一个文件夹, 添加`settings.gradle`文件并编辑(首先确定自己有多少子项目)
```gradle
rootProject.name = 'gradle-multi-module'

include 'subproject1'
include 'subproject2'
```
2. 在root目录下, 创建gradle wrapper的必要文件, `gradlew`, `gradlew.bat`, `gradle`(文件夹), 这些文件可以通过 Spring Initializr 新建一个项目得到.
3. 在root目录下, 创建subprojects文件夹, `mkdir -p src/main/java/com/example/multimodule/service`
4. 在每个subproject下创建`build.gradle`文件, 如下内容(示例):
```gradle
plugins {
	id 'org.springframework.boot' version '2.5.2'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
    maven {
      url 'https://maven.aliyun.com/repository/public/'
    }
    maven {
      url 'https://maven.aliyun.com/repository/spring/'
    }
    mavenLocal()
    mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### Gradle
对Gradle做出一些总结.

gradle是一个扩展性很强的build tool, 比Maven更加灵活. 
1. Gradle makes it easy to build common types of project — say Java libraries — by adding a layer of conventions and prebuilt functionality through plugins.
2. Gradle models its builds as Directed Acyclic Graphs (DAGs) of tasks (units of work). What this means is that a build essentially configures a set of tasks and wires them together — based on their dependencies — to create that DAG. Once the task graph has been created, Gradle determines which tasks need to be run in which order and then proceeds to execute them. Gradle构建项目的过程是执行一系列的task, 这些task构成了一个有向无环图DAG. 对于Java项目这些task就是: check, assemble, jar等等, 这些任务是内置的不需要用户定义.
这里还经常提到一个概念: DSL . DSL 其实是 Domain Specific Language 的缩写，中文翻译为领域特定语言; 而与 DSL 相对的就是 GPL, 这里的 GPL 并不是我们知道的开源许可证, 而是 General Purpose Language 的简称，即通用编程语言，也就是我们非常熟悉的 Objective-C、Java、Python 以及 C 语言等等。比如Regex, HTML等, Gradle中支持Grovvy. 与 GPL 相对，DSL 与传统意义上的通用编程语言 C、Python 以及 Haskell 完全不同。通用的计算机编程语言是可以用来编写任意计算机程序的，并且能表达任何的可被计算的逻辑，同时也是 {% post_link theory-turing-completeness '图灵完备' %} 的。
3. 没有必要全局安装Gradle，在项目中使用Gradle Wrapper即可，同时实际中要配置GRADLE_USER_HOME环境变量，设置Gradle Cache的存放位置，以免在C盘形成很多垃圾文件

