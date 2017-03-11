---
title: '[Maven Essentials] 生命周期和插件'
date: 2013-01-22 11:30:00
categories: ['工具, 库与框架', Maven]
tags: [Maven, 依赖管理]
---

## 概述

maven命令行的输入往往就对应了生命周期，如mvn package就表示执行默认生命周期阶段package。Maven的生命周期是抽象的，其实际行为都由插件来完成，如package阶段的任务可能就会由maven-jar-plugin来完成。
 
生命周期和插件两者协同工作，密不可分。

<!-- More -->

## 生命周期

Maven生命周期就是为了对所有的构建过程进行抽象和统一。
 
包括了：

- 清理
- 初始化
- 编译
- 测试
- 打包
- 集成测试
- 验证
- 部署
- 站点生成
 
Maven的生命周期时抽象的，这意味着生命周期本身不做任何实际的工作。在Maven的设计中，实际的任务(如编译源代码)都交由插件来完成。这种思想和设计模式中的模板方法(Template Method)非常相似。

以上的每个构建步骤都可以绑定一个或者多个插件行为，而且Maven为大多数构建步骤编写并绑定了默认插件。比如编译源代码和测试分别由以下两个插件来完成：

- maven-compiler-plugin
- maven-surefire-plugin

比如在一个项目的pom文件中，定义了如下的插件：

```
<build>  
   <plugins>  
      <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
         <artifactId>maven-compiler-plugin</artifactId>  
      </plugin>  
      <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
         <artifactId>maven-javadoc-plugin</artifactId>  
        <version>2.8.1</version>  
      </plugin>  
   </plugins>  
</build>  
```

它们分别用来编译源代码和生成javadoc文档。

### 三套生命周期

Maven不止提供了一套生命周期，实际上，Maven拥有三套相互独立的生命周期，它们分别为clean，default和site。它们的意义分别是：

- **clean**：清理项目
- **default**：构建项目
- **site**：建立项目站点

每个生命周期包含一些阶段(phase)，这些阶段是有顺序的，并且后面的阶段依赖于前面的阶段，用户和Maven最直接的交互方式就是调用这些生命周期阶段。
 
较之于生命周期阶段的前后依赖关系，三套生命周期本身是相互独立的，用户可以仅仅调用clean生命周期的某个阶段，或者仅仅调用default生命周期的某个阶段，而不会对其他生命周期产生任何影响。例如，当用户调用clean生命周期的clean阶段的时候，不会触发default生命周期的任何阶段。

#### clean生命周期

clean生命周期的目的是清理项目，包含以下三个阶段：

1. pre-clean:执行一些清理前需要完成的工作
2. clean:清理上一次构建生成的文件
3. post-clean:执行一些清理后需要完成的工作

#### default生命周期

default生命周期定义了真正构建时需要执行的所有步骤，它是所有生命周期中最核心的部分，由于所含有的阶段很多，下面只列出重要的阶段：

1. process-sources:处理项目主资源文件，一般而言，是对src/main/resources目录的内容进行变量替换等工作后，复制到项目输出的主classpath目录中
2. compile:编译项目的主源码。一般而言，是将src/main/java目录下的Java文件编译并输出到主classpath目录中
3. process-test-sources:处理项目测试资源文件。一般而言，是处理src/test/resources目录下的资源文件然后复制输出到测试classpath中
4. test-compile:编译测试代码。一般而言，是将src/test/java目录下的Java代码编译并输出至测试classpath目录中
5. test:使用单元测试框架运行测试，测试代码不会被打包或者部署
6. package:接受编译好的代码，打包成可发布的格式，如jar，war等
7. install:将包安装到Maven本地仓库，供本地其它Maven项目使用
8. deploy:将最终的包赋值到远程仓库，供其它开发人员和Maven项目使用

更加详细的说明，可以参考[官方文档](http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

#### site生命周期

site生命周期的目的是建立和发布项目站点，Maven能够基于POM所包含的信息，自动生成一个友好的站点，方便团队交流和发布项目信息。包含如下阶段:

1. pre-site:执行一些在生成项目站点之前需要完成的工作
2. site:生成项目站点文档
3. post-site:执行一些在生成项目站点之后需要完成的工作
4. site-deploy:将生成的项目站点发布到服务器上

### 命令行与生命周期

从命令行执行Maven任务的最主要方式就是调用Maven的生命周期阶段，需要注意的是，各个生命周期阶段是相互独立的，而一个生命周期的阶段是有前后依赖关系的，举几个例子：

- mvn clean：该命令调用clean生命周期的clean阶段。实际执行的阶段为clean生命周期的pre-clean和clean阶段
- mvn test：该命令调用default生命周期的test阶段。实际执行的阶段为default生命周期的直到test的所有阶段，这也解释了为什么在执行测试的时候，项目的代码能够自动得以编译
- mvn clean install：该命令调用clean生命周期的clean阶段和default生命周期的install阶段。实际执行的为clean生命周期的pre-clean、clean阶段以及default生命周期的直到install的所有阶段。该命令结合了两个生命周期，在执行真正地项目构建之前清理项目是一个很好的实践
- mvn clean deploy site-deploy：该命令调用clean生命周期的clean阶段，default生命周期的deploy阶段以及site生命周期的site-deploy阶段。实际执行的为clean生命周期的pre-clean,clean阶段以及default和site生命周期的所有阶段

## 插件相关概念

### 插件目标

对于插件本身，为了能够复用代码，它往往能够完成多个任务。例如maven-dependency-plugin插件，它能够基于项目依赖做很多事情，比如：

- 分析项目依赖，找出潜在的无用依赖
- 列出项目的依赖树，帮助分析以来来源
- 列出项目所有已解析的依赖

而为以上的每个功能都编写一个独立的插件显然是不可取的，因为这些任务背后有很多可以复用的代码，因此，这些功能聚集在一个插件里，每个功能就是一个插件目标。
 
maven-dependency-plugin插件有十多个目标，每个目标对应了一个功能，上述提到的几个功能对应的目标是：

- dependency:analyze
- dependency:tree
- dependency:list

### 插件绑定

Maven的生命周期与插件相互绑定，用以完成实际的构建任务。具体而言，是生命周期的阶段和插件的目标相互绑定，以完成某个具体的构建任务。例如源码编译这一任务，是将default生命周期的compile阶段和maven-compiler-plugin这一插件的compile目标进行绑定。
 
#### 内置绑定

为了简化在构建项目时用户的操作，Maven提供了大量的内置绑定。列举如下:

|  生命周期阶段 |  插件目标 | 执行任务  |
|---|---|---|
| process-resources  | maven-resources-plugin:resources  | 复制主资源文件至主输出目录  |
|  compile | maven-compiler-plugin:compile  | 编译主代码至主输出目录  |
|  test-compile | maven-compiler-plugin:testCompile  | 编译测试代码至测试输出目录  |
| test  | maven-surefire-plugin:test  | 执行测试用例  |
| package  |  maven-jar-plugin:jar |  创建项目jar包 |
|  install | maven-install-plugin:install  | 将项目输出构件安装到本地仓库  |
| deploy  | maven-deploy-plugin:deploy  | 将项目输出构件部署到远程仓库  |

除了默认的打包类型jar之外，常见的打包类型还有war，pom，maven-plugin，ear等。

#### 自定义绑定

除了内置绑定以外，用户还能自己选择将某个插件目标绑定到生命周期的某个阶段上，典型的应用如创建项目的源码jar包，内置的插件绑定关系中并没有涉及这一任务，因此需要用户自行配置。
 
可以使用maven-source-plugin来完成这个任务，它的jar-no-fork目标能够将项目的主代码打包成jar文件，可以将其绑定到default生命周期的verify阶段(集成测试之后，install之前的一个阶段)：

```
<build>  
  <plugins>  
    <plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-source-plugin</artifactId>  
      <version>2.1.1</version>  
      <executions>  
        <execution>  
          <id>attach-sources</id>  
          <phase>verify</phase>  
          <goals>  
            <goal>jar-no-fork</goal>  
          </goals>  
        </execution>  
      </executions>  
    </plugin>  
  </plugins>  
</build>  
```

有时候，即使不通过phase元素配置生命周期阶段，插件目标也能够绑定到生命周期中去。例如，可以删除上述配置中的phase一行，再次执行mvn verify，仍然可以看到该目标得以执行。出现这种现象的原因是：有很多插件的目标在编写时已经定义了默认绑定阶段。可以使用maven-help-plugin查看插件的详细信息：

```
mvn help:describe-Dplugin=org.apache.maven.plugins:maven-source-plugin:2.11 -Ddetail
```
 
如果多个目标被绑定到同一个阶段，那么这些插件声明的先后顺序决定了目标的执行顺序。





