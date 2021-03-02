---
title: "Maven使用指南"
date: 2021-02-20T09:40:14+08:00
draft: true
categories:
  - 打包组件
  - Java
  - Maven
tags:
  - 打包组件
  - Java
  - Maven
---

# 介绍

Maven就是是专门为Java项目打造的管理和构建工具，它的主要功能有：

- 提供了一套标准化的项目结构；
- 提供了一套标准化的构建流程（编译，测试，打包，发布……）；
- 提供了一套依赖管理机制。

## Maven项目结构

一个使用Maven管理的普通的Java项目，它的目录结构默认如下

```ascii
a-maven-project
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       ├── java
│       └── resources
└── target
```

项目的根目录`a-maven-project`是项目名，它有一个项目描述文件`pom.xml`，

存放Java源码的目录是`src/main/java`，

存放资源文件的目录是`src/main/resources`，

存放测试源码的目录是`src/test/java`，

存放测试资源的目录是`src/test/resources`，

最后，所有编译、打包生成的文件都放在`target`目录里。这些就是一个Maven项目的标准目录结构。

所有的目录结构都是约定好的标准结构，我们千万不要随意修改目录结构。使用标准结构不需要做任何配置，Maven就可以正常使用。

我们再来看最关键的一个项目描述文件`pom.xml`，它的内容长得像下面：

```xml
<project ...>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.itranswarp.learnjava</groupId>
	<artifactId>hello</artifactId>
	<version>1.0</version>
	<packaging>jar</packaging>
	<properties>
        ...
	</properties>
	<dependencies>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>
	</dependencies>
</project>
```

其中，`groupId`类似于Java的包名，通常是公司或组织名称，`artifactId`类似于Java的类名，通常是项目名称，再加上`version`，一个Maven工程就是由`groupId`，`artifactId`和`version`作为唯一标识。我们在引用其他第三方库的时候，也是通过这3个变量确定。例如，依赖`commons-logging`：

```xml
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.2</version>
</dependency>
```

使用`<dependency>`声明一个依赖后，Maven就会自动下载这个依赖包并把它放到classpath中。

## 安装Maven

要安装Maven，可以从[Maven官网](https://maven.apache.org/)下载最新的Maven 3.6.x，然后在本地解压，设置几个环境变量：

```
M2_HOME=/path/to/maven-3.6.x
PATH=$PATH:$M2_HOME/bin
```

Windows可以把`%M2_HOME%\bin`添加到系统Path变量中。

然后，打开命令行窗口，输入`mvn -version`，应该看到Maven的版本信息：

## 在IDE中使用Maven

略

# 依赖管理

Maven的一个重要作用就是为我们管理依赖。首先我们先展示一下常见的依赖示例，稍后我们会对其进行详细介绍

```xml
<dependency>
    <groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<version>5.1.48</version> 
    <scope>provided</scope>
</dependency>
```

## 依赖的定义

### 依赖正式包

假如我们的项目需要mysql-connector-java这个依赖，在Maven中我们需要通过以下的格式来唯一确定一个依赖

```xml
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>5.1.48</version>    
```

- groupId：属于组织的名称，类似Java的包名；
- artifactId：该jar包自身的名称，类似Java的类名；
- version：该jar包的版本。

### 依赖快照包

特殊的，如果我们依赖的是某个包的快照版本，那么每次打包项目时如果我们想要获取到该快照的最新版本，有多种方式

1）在执行Maven命令时添加-U参数，无视maven配置文件来强制拉取所依赖的快照包的最新版本

2）在maven的配置文件settings.xml中执行快照仓库以及快照的更新周期

```xml
<settings>
  <profiles>
		<profile>
      <id>default</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
      <repositories>
         <repository>
            <id>nexus</id>
            <name>xxx nexus3</name>
					  <url>http://xxxxx/nexus3/repository/public/</url>
          </repository>
       </repositories>
			<pluginRepositories>
                <pluginRepository>
                    <id>xxx</id>
                    <url>http://xxxxx/nexus3/repository/public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    < !-- 在这里配置快照 -->
                    <snapshots>
                        <!-- 启用快照 -->
                        <enabled>true</enabled>
                        <!-- 配置更新周期,always每次都更新,daily每天更新一次,interval:X 每x分钟更新一次,never永不更新 -->  
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>
</settings>
```

## 依赖范围

Maven定义了几种依赖范围，分别是`compile`、`test`、`runtime`和`provided`：

| scope    | 说明                                          | 示例            |
| :------- | :-------------------------------------------- | :-------------- |
| compile  | 编译时需要用到该jar包（默认）                 | commons-logging |
| test     | 编译Test时需要用到该jar包                     | junit           |
| runtime  | 编译时不需要，但运行时需要用到                | mysql           |
| provided | 编译时需要用到，但运行时由JDK或某个服务器提供 | servlet-api     |

## 依赖的声明与依赖的引入

我们在项目中如果要依赖某个包，有两种方式

1）直接引入

```xml
<dependency>
    <groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<version>5.1.48</version> 
</dependency>
```

2) 先声明依赖版本再引入

```xml
<!-- 在这里统一声明各依赖的版本号,只进行声明并不实际引入 -->
<dependencyManagement>
	<dependency>
    <groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<version>5.1.48</version> 
	</dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.9.RELEASE</version>
	</dependency>
</dependencyManagement>

<!-- 在这里进行依赖的实际引入 -->
<dependencies>
  <!-- 如果不指定版本,那么就会从dependencyManagement中去查找这个包并引入,例如这里就会实际引入5.1.48版本的mysql驱动 -->
	<dependency>
    <groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
	</dependency>
  <!-- 如果指定版本,那么就会以这里声明的版本号引入,例如这里会引入2.1.0版本的mybatis-spring-boot-starter -->
  <dependency>
     <groupId>org.mybatis.spring.boot</groupId>
     <artifactId>mybatis-spring-boot-starter</artifactId>
     <version>2.1.0</version>
  </dependency>
</dependencies>
```

**dependencies**

- 引入依赖
- 即使子项目中不写 dependencies ，子项目仍然会从父项目中继承 **dependencies** 中的所有依赖项

**dependencyManagement**

- 声明依赖，并不引入依赖。
- 子项目默认不会继承父项目 **dependencyManagement** 中的依赖
- 只有在子项目中写了该依赖项，并且没有指定具体版本，才会从父项目中继承（version、exclusions、scope等读取自父pom）
- 子项目如果指定了依赖的具体版本号，会优先使用子项目中指定版本，不会继承父pom中申明的依赖

## 依赖机制与调解方案

### 依赖传递

Maven根据POM下载依赖时，会自动解析Maven项目的依赖。假如我们的工程中引入了a，a依赖b，b依赖c和d，那么我们只需要引入了a，maven会自动帮我们引入b、c、d。

依赖传递的发生有两种情况：一种是存在模块之间的**继承**关系，在继承父模块后同时引入了父模块中的依赖，可通过**可选依赖机制**放弃依赖传递到子模块；另一种是引包时附带引入该包所依赖的包，该方式是引起依赖冲突的主因。

### 依赖优化

实际上Maven是比较“智能”的，它能够自动解析直接依赖和传递性依赖，根据预定义规则判断依赖范围的合理性，也可以对部分依赖进行适当调整来保证构件版本唯一。

即使这样，还会有些情况使Maven误判，因此手工进行依赖优化还是相当有必要的。读者可以使用maven-dependency-plugin提供的三个目标来实现依赖分析：

```bash
$ mvn dependency:list
$ mvn dependency:tree
$ mvn dependency:analyze
```

若读者还需更精细的分析结果，可以在命令后使用诸如以下参数：

```text
-Dverbose
-Dincludes=<groupId>:<artifactId>
```

当然了，现在在不同的IDE中也提供了各式各样的可视化插件方便我们排查依赖关系，例如Idea中我们可以使用Maven Helper来快速查看依赖关系。

### 依赖调解

如果我们的项目中发生了依赖冲突,比如a依赖b、c，b也依赖c，那么可能存在c的两个不同版本，这时候就需要对依赖进行调解。依赖调解遵循以下两大原则：路径最短优先、声明顺序优先，但是，若相同类型但版本不同的依赖存在于同一个pom文件，依赖调解两大原则都不起作用，需要采用**覆盖策略**来调解依赖冲突，最终会引入最后一个声明的依赖。

### 剔除依赖

接触依赖冲突时也有一种简单粗暴的方式，就是直接剔除依赖

```xml
<dependency>
  <groupId>org.glassfish.jersey.containers</groupId>
  <artifactId>jersey-container-grizzly2-http</artifactId>
  <!-- 剔除依赖 -->
  <exclusions>
    <exclusion>
      <groupId>org.glassfish.hk2.external</groupId>
      <artifactId>jakarta.inject</artifactId>
    </exclusion>
    ...
  </exclusions>
</dependency>
```

# 构建流程

Maven不但有标准化的项目结构，而且还有一套标准化的构建流程，可以自动化实现编译，打包，发布，等等。

## 概念介绍

为了实现标准化的构建流程，Maven中定义了这么几个重要概念

- lifecycle ： 生命周期，相当于Java的package，它包含一个或多个phase；
- phase ： 阶段，相当于Java的class，它包含一个或多个goal；
- goal ： 相当于class的method，它其实才是真正干活的。

Maven有三套相互独立的生命周期，分别是
1）Clean Lifecycle 在进行真正的构建之前进行一些清理工作
2）Default Lifecycle: 构建的核心部分，包含 编译，测试，打包，安装和部署等。
3）Site Lifecycle： 生成项目报告，站点，发布站点。

### Clean生命周期

Clean生命周期一共包含三个阶段

| 阶段       | 作用                                  |
| ---------- | ------------------------------------- |
| pre-clean  | 执行一些需要在clean之前完成的工作     |
| clean      | 移除所有上一次构建生成的文件          |
| post-clean | 执行一些需要在clean之后立刻完成的工作 |

###  Default生命周期

Default生命周期中包含了日常工作的绝大部分阶段

| phase                          | function                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| validate                       | validate the project is correct and all necessary information is available. |
| initialize                     | initialize build state, e.g. set properties or create directories. |
| generate-sources               | generate any source code for inclusion in compilation.       |
| process-sources                | process the source code, for example to filter any values.   |
| generate-resources             | generate resources for inclusion in the package.             |
| process-resources              | copy and process the resources into the destination directory, ready for packaging. |
| compile                        | compile the source code of the project.                      |
| process-classes                | post-process the generated files from compilation, for example to do bytecode enhancement on Java classes. |
| generate-test-sources          | generate any test source code for inclusion in compilation.  |
| process-test-sources           | process the test source code, for example to filter any values. |
| generate-test-resources        | create resources for testing.                                |
| process-test-resources         | copy and process the resources into the test destination directory. |
| test-compile                   | compile the test source code into the test destination directory |
| process-test-classes           | post-process the generated files from test compilation, for example to do bytecode enhancement on Java classes. For Maven 2.0.5 and above. |
| test                           | run tests using a suitable unit testing framework. These tests should not require the code be packaged or deployed. |
| prepare-package                | perform any operations necessary to prepare a package before the actual packaging. This often results in an unpacked, processed version of the package. (Maven 2.1 and above) |
| package                        | take the compiled code and package it in its distributable format, such as a JAR. |
| pre-integration-test   perform | actions required before integration tests are executed. This may involve things such as setting up the required environment. |
| integration-test               | process and deploy the package if necessary into an environment where integration tests can be run. |
| post-integration-test          | perform actions required after integration tests have been executed. This may including cleaning up the environment. |
| verify                         | run any checks to verify the package is valid and meets quality criteria. |
| install                        | install the package into the local repository, for use as a dependency in other projects locally. |
| deploy                         | done in an integration or release environment, copies the final package to the remote repository for sharing with other developers and projects. |



### Site生命周期

Site生命周期一共包含四个阶段

| 阶段        | 作用                                                   |
| ----------- | ------------------------------------------------------ |
| pre-site    | 执行一些需要在生成站点文档之前完成的工作               |
| site        | 生成项目的站点文档                                     |
| post-site   | 执行一些需要在生成站点文档之后完成的工作，为部署做准备 |
| site-deploy | 将生成的站点文档部署到特定的服务器上                   |



## 构建命令

我们使用`mvn`这个命令时，后面的参数是phase，Maven自动根据生命周期运行到指定的phase。

更复杂的例子是指定多个phase，例如，运行`mvn clean package`，Maven先执行`clean`生命周期并运行到`clean`这个phase，然后执行`default`生命周期并运行到`package`这个phase，实际执行的phase如下：

- pre-clean
- clean （注意这个clean是phase）
- validate
- ...
- package

在实际开发过程中，经常使用的命令有：

`mvn clean`：清理所有生成的class和jar；

`mvn clean package`：先清理，再执行到package；

`mvn clean install`：先清理，再执行到install；

`mvn clean deploy`：先清理，再执行到deploy；

这里额外解释一下 package、intall、deploy命令的区别

| 命令\包含阶段 | 编译、单元测试、打包 | 将包部署到本地Maven | 将包部署到远程Maven |
| ------------- | -------------------- | ------------------- | ------------------- |
| package       | YES                  | NO                  | NO                  |
| install       | YES                  | YES                 | NO                  |
| deploy        | YES                  | YES                 | YES                 |

# 插件

## 插件的使用

我们以`compile`这个phase为例，如果执行：

```
mvn compile
```

Maven将执行`compile`这个phase，这个phase会调用`compiler`插件执行关联的`compiler:compile`这个goal。

实际上，执行每个phase，都是通过某个插件（plugin）来执行的，Maven本身其实并不知道如何执行`compile`，它只是负责找到对应的`compiler`插件，然后执行默认的`compiler:compile`这个goal来完成编译。

所以，使用Maven，实际上就是配置好需要使用的插件，然后通过phase调用它们。

Maven已经内置了一些常用的标准插件：

| 插件名称 | 对应执行的phase |
| :------- | :-------------- |
| clean    | clean           |
| compiler | compile         |
| surefire | test            |
| jar      | package         |

如果这些标准插件不能满足，那么可以自定义插件。例如在日常的工作中，我们可能会需要将项目目录中的配置文件目录、SQL目录、项目WAR/JAR包组合打成一个TAR包，这时候我们就可以使用maven-assemply-plugin来完成。

```xml
<plugins>
                <plugin>
                    <artifactId>maven-assembly-plugin</artifactId>
                    <configuration>
                        <finalName>${project.build.finalName}-server</finalName>
                        <descriptors>
                            <!-- 在src/main/assembly/assembly.xm中配置需要打包的目录 -->
                            <descriptor>src/main/assembly/assembly.xml</descriptor>
                        </descriptors>
                    </configuration>
                    <executions>
                        <!-- 执行package阶段的single方法 -->
                        <execution>
                            <id>make-assembly</id>
                            <phase>package</phase>
                            <goals>
                                <goal>single</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
</plugins>
```

然后是assembly.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>

<assembly>
    <!-- 可自定义，这里指定的是项目环境 -->
    <id>assembly</id>

    <!-- 打包的类型，如果有N个，将会打N个类型的包 -->
    <formats>
        <format>tar.gz</format>
    </formats>

    <includeBaseDirectory>true</includeBaseDirectory>

    <fileSets>
        <!-- 将src/bin目录下的所有脚本输出到打包后的bin目录中,并赋予0755读写权限 -->
        <fileSet>
            <directory>${basedir}/src/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <fileMode>0755</fileMode>
            <lineEnding>unix</lineEnding>
            <includes>
                <include>**.sh</include>
                <include>**.bat</include>
            </includes>
        </fileSet>

        <!-- 指定输出target/classes中的配置文件到config目录中 -->
        <fileSet>
            <directory>${basedir}/target/classes/config</directory>
            <outputDirectory>conf/config</outputDirectory>
            <fileMode>0644</fileMode>
            <includes>
                <include>*</include>
            </includes>
        </fileSet>

        <!-- 将项目启动jar打包到lib目录中 -->
        <fileSet>
            <directory>${basedir}/target</directory>
            <outputDirectory>lib</outputDirectory>
            <fileMode>0755</fileMode>
            <includes>
                <include>${project.build.finalName}.jar</include>
            </includes>
        </fileSet>
...
    </fileSets>
</assembly>
```

## 插件介绍

| 名称      | 类型 | 描述                                                         |
| --------- | ---- | ------------------------------------------------------------ |
| clean     | 核心 | 清理上一次执行创建的目标文件                                 |
| complier  | 核心 | 编译java代码                                                 |
| resources | 核心 | 编译resources资源文件                                        |
| install   | 核心 | 安装 jar/war等可执行包，将创建生成的 jar 拷贝到 .m2/repository 下面 |
| deploy    | 核心 | 将jar/war等可执行包发布到远程仓库                            |
| jar       | 打包 | 将java代码打成jar包                                          |
| war       | 打包 | 将java代码打成war包                                          |
| shade     | 打包 | 在打包过程中，包含依赖库/重命名依赖库/选择性的打包，可以对依赖有所取舍 |
| assembly  | 打包 | 将指定文件或目录打成压缩包，例如将doc/、conf/、jar一起打成tar包 |
| archetype | 工具 | 生成 Maven 项目骨架                                          |
| versions  | 工具 | 一键更改多个模块中的版本号                                   |

# 模块管理

为了降低项目管理的复杂度，一般将项目拆分为多个模块，将多个模块共有的依赖都在父模块中声明和引入，这样子模块就可以直接使用。例如如下的工程结构

```ascii
multiple-project
├── pom.xml
├── parent
│   └── pom.xml
├── module-base
│   ├── pom.xml
│   └── src
├── module-dao
│   ├── pom.xml
│   └── src
├── module-service
│   ├── pom.xml
│   └── src
├── module-management
│   ├── pom.xml
│   └── src
├── module-api
│   ├── pom.xml
│   └── src
└── module-war
    ├── pom.xml
    └── src
```

parent pom

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.itranswarp.learnjava</groupId>
    <artifactId>parent</artifactId>
    <version>1.0</version>
    <packaging>pom</packaging>

    <name>parent</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.28</version>
        </dependency>
...
        </dependency>
    </dependencies>
</project>
```

Module-service pom

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.itranswarp.learnjava</groupId>
        <artifactId>parent</artifactId>
        <version>1.0</version>
        <relativePath>../parent/pom.xml</relativePath>
    </parent>

    <artifactId>module-a</artifactId>
    <packaging>jar</packaging>
    <name>module-service</name>
    
    <!-- 如果service模块需要依赖dao模块那就直接把dao模块引入 -->
    <dependencies>
        <dependency>
            <groupId>com.itranswarp.learnjava</groupId>
            <artifactId>module-dao</artifactId>
            <version>1.0</version>
        </dependency>
    </dependencies>
    
</project>
```

根目录 pom

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.itranswarp.learnjava</groupId>
    <artifactId>build</artifactId>
    <version>1.0</version>
    <packaging>pom</packaging>
    <name>build</name>

    <modules>
        <module>parent</module>
        <module>module-base</module>
        <module>module-dao</module>
        <module>module-service</module>
        ...
    </modules>
</project>
```

这样，在根目录执行`mvn clean package`时，Maven根据根目录的`pom.xml`找到包括`parent`在内的共4个`<module>`，一次性全部编译。

# Profile

在实际的项目开发中，通常是多套环境相互隔离的，一般划分为开发环境/测试环境/生产环境。不同环境中的配置一般并不相同，如数据库地址等。在Maven可以使用Profile来完成多套环境的配置共存。在Java开发中，一般采用Spring+Maven相融合的方式使用Profile。

首先需要我们在POM中添加多个Profile

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <profile.active>dev</profile.active>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <profile.active>prod</profile.active>
        </properties>
    </profile>
</profiles>
```

其次为了让 maven 在构建时用 profile 中指定的属性来替换 `application.properties` 中的内容，需要我们在Maven的pom中对properties类文件单独进行处理。

```xml
<resources>
    <resource>
        <directory>src/main/resources</directory>
        <!-- 这里的目的是先不要将application*.properties打包到项目中 -->
        <excludes>
            <exclude>application*.properties</exclude>
        </excludes>
    </resource>
    <resource>
        <directory>src/main/resources</directory>
        <!-- 这里开启 filtering，maven 会将文件中的 %XX%或者@XX@ 替换 profile 中定义的 XX 变量/属性(如果不想使用%XX%或者@XX@，可以通过配置delimiters来改变)。另外，我们还通过 includes 来告诉 maven 根据profile 来复制对应的 properties 文件 -->
        <filtering>true</filtering>
        <includes>
            <include>application.properties</include>
            <include>application-${profile.active}.properties</include>
        </includes>
    </resource>
</resources>
```

接下来我们需要将maven指定的profile的值传递给spring。在 `application.properties` 文件中加入下面这行：

```
spring.profiles.active = @profile.active@
```

这里 `profile.active` 是在 maven profile 中的 `properties` 定义的，而 `@XX@` 的语法则是上节提到的 maven filtering 替换变量的语法。

最后就是构建步骤了，在构建时我们通过为maven命令指定参数-P来实现profile的使用。例如mvn clean package -P<profile_name>。例如，mvn参数中的profile为dev，那么它会被传递并替换到`application.properties`中，然后spring将spring.profiles设置为dev，这样就完成了Maven和Spring的profile联动。

# 发布Artifact

略



# 参考链接

[Maven基础 - 廖雪峰](https://www.liaoxuefeng.com/wiki/1252599548343744/1255945359327200)

[Maven生命周期 - data4](https://www.jianshu.com/p/fd43b3d0fdb0)

[深入理解maven构建生命周期和各种plugin插件 - 阿童木](https://blog.csdn.net/zhaojianting/article/details/80321488)

