---
title: Maven Wrapper最佳实践
date: 2021-11-07 16:22:04
tags:
- maven
---

将一个maven项目所有信息都保留在项目根目录下, 包括maven本身


## Maven Wrapper安装
在项目根目录下
```
mvn -N io.takari:maven:0.7.7:wrapper
mvn -N io.takari:maven:wrapper -Dmaven=3.6.3
```
This creates two files (mvnw, mvnw.cmd) and a hidden directory (.mvn). mvnw can be used in Unix-like environments and mvnw.cmd can be used in Windows.
这将会创建两个文件和一个目录. `mvnw`用来Unix-like环境, `mvnw.cmd`用在Windows环境.

Instead of the usual mvn command, they would use mvnw. for example:
```shell
./mvnw clean install
```

## Maven Wrapper原理
The `.mvn/wrapper` directory has a jar file `maven-wrapper.jar` that downloads the required version of Maven if it’s not already present. It installs it in the `./m2/wrapper/dists` directory under the user’s home directory.

Where does it download Maven from? This information is present in the mvn/wrapper/maven-wrapper.properties file:
```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.5.2/apache-maven-3.5.2-bin.zip
wrapperUrl=https://repo.maven.apache.org/maven2/io/takari/maven-wrapper/0.5.6/maven-wrapper-0.5.6.jar
```

## Maven Wrapper贯彻的思想
Years ago, I was on a team developing a desktop-based Java application. We wanted to share our artifact with a couple of business users in the field to get some feedback. It was unlikely they had Java installed. Asking them to download, install, and configure version 1.2 of Java (yes, this was that long ago!) to run our application would have been a hassle for them.

Looking around trying to find how others had solved this problem, I came across this idea of **“bundling the JRE”**. The idea was to include within the artifact itself the Java Runtime Environment that our application depended on. Then users don’t need to have a particular version or even any version of Java pre-installed - a neat solution to a specific problem.

Over the years I came across this idea in many places. Today when we containerize our application for cloud deployment, it’s the same general idea: encapsulate the dependent and its dependency into a single unit to hide some complexity.
The Maven Wrapper makes it easy to build our code on any machine, including CI/CD servers. We don’t have to worry about installing the right version of Maven on the CI servers anymore!


## 结合IDEA的最佳实践
收到这个"捆绑思想"的影响. 我希望我可以在开发时, 将项目中的依赖从本地仓库中分离. 一个项目一个依赖库, 就比如node_modules.
1. 安装好maven wrapper
2. 复制一个`settings.xml`到`$PROJECT_DIR$/.mvn/`下
3. `$PROJECT_DIR$/.mvn/`新建文件夹`repository`
4. IDEA中配置`Setting > Build, Execution, Deployment > Build Tools > Maven`
 - 修改`Maven Home Path`为`Use Maven wrapper`, 如果发现没有这个选项说明IDEA没有更新(至少要2021.2.3+的版本)
 - 修改`User settings file`为`$PROJECT_DIR$/.mvn/settings.xml`
 - 修改`Local repository`为`$PROJECT_DIR$/.mvn/repository`
