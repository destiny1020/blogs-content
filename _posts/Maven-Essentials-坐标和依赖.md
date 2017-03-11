---
title: '[Maven Essentials] 坐标和依赖'
date: 2013-01-22 11:22:00
categories: ['工具, 库与框架', Maven]
tags: [Maven, 依赖管理]
---

## 坐标要素

- groupId

	当前maven项目隶属的实际项目。Maven项目和实际项目不一定是一对一的关系。
 
- artifactId
	
	定义实际项目中的一个maven项目(模块)，推荐的做法是使用实际项目名称作为artifactId的前缀。
 
- version
	
	定义maven项目当前所处的版本。
 
- packaging
	
	定义maven项目的打包方式。默认为jar。
 
- classifier
	
	用来帮助定义构建输出的一些附属构建。比如源代码jar包和文档jar包。该要素不能直接定义，需要通过附加的插件帮助生成。
	
<!-- More -->
 
---

构件的文件名与坐标对应：

一般的规则为artifactId - version [-classifier].packaging其中classifier是可选的。

一个例子：

```
<groupId>com.example.project</groupId>  
<artifactId>backend</artifactId>  
<packaging>jar</packaging>  
```

更推荐的做法应该是将artifactId由backend改成project-backend，这是因为如果还存在一个名为xxx的项目，其中如果也存在一个backend的模块(子项目)，那么最后这两个模块都会生成叫做backend-version.jar的输出，这样不利于定位和分辨。

## 依赖的配置

比如在项目的pom.xml文件中：

```
<dependencies>  
  <dependency>  
    <groupId>com.example.project</groupId>  
    <artifactId>project-cms</artifactId>  
    <version>${project.version}</version>  
    <exclusions>  
      <exclusion>  
        <groupId>cglib</groupId>  
        <artifactId>cglib</artifactId>  
      </exclusion>  
      <exclusion>  
        <groupId>jboss</groupId>  
        <artifactId>javassist</artifactId>  
      </exclusion>  
    </exclusions>  
  </dependency>  
  ……  
</dependencies>
```

dependencies标签下定义了若干个dependency元素，每个dependency元素中包含的要素如下：

- groupId，artifactId，version

	它们是依赖的基本坐标，这是最重要的三个要素，因为只有通过这三个坐标才能找到所需要的依赖。
	
- type

	依赖的类型，对应项目坐标中定义的packaging，大部分情况下，不需要声明它，其默认值为jar。
	
- scope

	依赖的范围。默认的是compile，另外的还有test，provided，runtime以及system。
	
- optional
	
	标记依赖是否可选。
	
- exclusions

	用来排除传递性依赖。
	
### 依赖范围

- compile
	
	默认的依赖范围、使用compile作为依赖范围，那么此依赖对于编译、测试、运行三种classpath都有效。
 
- test
	
	测试依赖范围。此种依赖只对于测试classpath有效，在编译主代码或者运行项目时将无法使用此依赖。典型的例子如JUnit等。
 
- provided

	以提供依赖范围。对于编译和测试classpath有效，但是在运行时无效。典型的例子如servlet-api，编译和测试项目的时候需要，但是在运行项目的时候，由于容器已经提供，就不需要重复引入了。
 
- runtime
	
	运行时依赖范围。对于测试和运行classpath有效，但是在编译主代码时无效。典型的例子如JDBC驱动。
 
- system
	
	系统依赖范围。classpaths相关的依赖关系和provided一致。但是使用system范围的依赖必须通过systemPath元素显式地指定依赖文件的路径。此类依赖不是通过maven仓库解析的，而且往往与本机系统绑定，可能造成构建的不可移植，应该谨慎使用。
 
- import
	
	导入依赖范围。参见后序文章。
	
| 依赖范围  |  编译classpath是否有效 | 测试classpath是否有效  | 运行时classpath是否有效 |  例子 |
|---|---|---|---|---|
| compile  | YES  | YES  | YES  |  默认依赖范围 |
|  test |  NO |  YES |  NO |  JUnit |
| provided  |  YES | YES  | NO  | servlet-api  |
|  runtime | NO  |  YES | YES  | JDBC驱动实现  |
|  system | YES  | YES  | NO  |  本地的，maven仓库之外的类库 |

### 依赖性传递

比如在project-backend项目中为了使用反射的一些功能，引入了如下依赖：

```
<dependency>  
  <groupId>org.reflections</groupId>  
  <artifactId>reflections</artifactId>  
  <version>0.9.8</version>  
</dependency> 
```

而这个构件本身也存在其它的一些依赖，可以通过访问该构件的pom文件来查看：

```
<dependency>  
  <groupId>com.google.guava</groupId>  
  <artifactId>guava</artifactId>  
  <!--<version>12.0</version>--> <!-- jdk1.6+ -->  
  <version>11.0.2</version> <!-- case: when only jdk1.5 isacceptable -->  
</dependency>
```

可以看到这个构件本身又依赖于Google的guava这一项目。
 
因此在最后的实际依赖构件中，guava这一构件会以传递依赖的形式被添加到maven dependency中。
