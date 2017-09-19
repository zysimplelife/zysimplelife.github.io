---
layout: post
title:  "Gradle vs Maven in my opinion"
date: 2017-09-17 15:10:00 +0000
categories: Build, Maven, Gradle
---

### 依赖管理更简洁

这个应该是最容易体会到的优势， 举个例子对比一下 
maven 是这样的

```xml
<dependency>
 <groupId>junit</groupId>
 <artifactId>junit</artifactId>
 <version>4.12</version>
 <scope>test</scope>
</dependency>
<dependency>
 <groupId>org.springframework</groupId>
 <artifactId>spring-test</artifactId>
</dependency>
```

而gradle 是这样的. 可以看出来 gradle 简洁了很多， SBT 的依赖定义也和这个类似

```groove
dependencies {
 compile 'org.hibernate:hibernate-core:3.6.7.Final'
 testCompile ‘junit:junit:4.+'
}
```

### 不用预先安装

如果想用编译maven 工程，必须要先安装maven，并且因为不同版本的原因，可能会出现不同的编译问题。 Gradle 以及SBT 都采用了wrapper 方式发布源代码，这样可以在配置文件中指定对应的版本 Gradle.  比如

```groove
ext {
  gradleVersion = "3.5"
  buildVersionFileName = "kafka-version.properties"

  maxPermSizeArgs = []
  if (!JavaVersion.current().isJava8Compatible())
    maxPermSizeArgs = ['-XX:MaxPermSize=512m']

  userMaxForks = project.hasProperty('maxParallelForks') ? maxParallelForks.toInteger() : null

  skipSigning = project.hasProperty('skipSigning') && skipSigning.toBoolean()
  shouldSign = !skipSigning && !version.endsWith("SNAPSHOT") && project.gradle.startParameter.taskNames.any { it.contains("upload") }

  mavenUrl = project.hasProperty('mavenUrl') ? project.mavenUrl : ''
  mavenUsername = project.hasProperty('mavenUsername') ? project.mavenUsername : ''
  mavenPassword = project.hasProperty('mavenPassword') ? project.mavenPassword : ''

  userShowStandardStreams = project.hasProperty("showStandardStreams") ? showStandardStreams : null

  userTestLoggingEvents = project.hasProperty("testLoggingEvents") ? Arrays.asList(testLoggingEvents.split(",")) : null

  generatedDocsDir = new File("${project.rootDir}/docs/generated")

  commitId = project.hasProperty('commitId') ? commitId : null
}


```


### Scope 不同 

Gradle 的scope 比maven少。在Maven世界中，一个依赖项有6种scope，分别是complie(默认)、provided、runtime、test、system、import。而grade将其简化为了4种，compile、runtime、testCompile、testRuntime。 这个很难说是好还是不好， 不过我们大部分的时候4个scope也够用了


### 动态的版本依赖

在版本号后面使用+号的方式可以实现动态的版本管理。这一点在maven 是用 [3.4) 这种方法实现的，但是mvn的问题是会大大增加编译时间，因为maven会吧所有的版本定义都先下载下来，然后进行选择。 gradle没有试过，不知道是不是会有所改善

### 依赖冲突
使用Maven和Gradle进行依赖管理时都采用的是传递性依赖；而如果多个依赖项指向同一个依赖项的不同版本时就会引起依赖冲突。而Maven处理这种依赖关系往往是噩梦一般的存在。而Gradle在解决依赖冲突方面相对来说比较明确。在maven中我们是通过exclusion来解决的。但是问题是在编译阶段我们经常无法发现是否有依赖冲突，这样到实际测试时往往要花很多的时间去发现问题。

```xml
<dependency>
	<groupId>org.mockito</groupId>
	<artifactId>mockito-core</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.hamcrest</groupId>
			<artifactId>hamcrest-core</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```

而gradle是这样解决的

Gradle offers the following conflict resolution strategies:

* Newest: The newest version of the dependency is used. This is Gradle’s default strategy, and is often an appropriate choice as long as versions are backwards-compatible.

* Fail: A version conflict results in a build failure. This strategy requires all version conflicts to be resolved explicitly in the build script. See ResolutionStrategy for details on how to explicitly choose a particular version.


简单的说就是要不然选择最新的，要不然就在编译阶段报错，由开发人员自己解决。 这样就把依赖冲突提前到编译阶段，大大降低trouble shooting 的cost。 例如

```groove
configurations.all {
  resolutionStrategy {
    // fail eagerly on version conflict (includes transitive dependencies)
    // e.g. multiple different versions of the same dependency (group and name are equal)
    failOnVersionConflict()

    // prefer modules that are part of this build (multi-project or composite build) over external modules
    preferProjectModules()
...
  } 
}

```
### 多模块构建

在SOA和微服务的浪潮下，将一个项目分解为多个模块已经是很通用的一种方式。在 maven 中都需要定义一个 parent 的配置来定义common的部分,而这个parent是没有什么具体作用的。而Gradle的推荐做法是在一个build.gradle中定义所有的project，这样就不会有分散在不同地方的配置。同事通过allprojects和subprojects代码块来分别定义里面的配置是应用于所有项目还是子项目。这无疑比Maven要灵活的多，但是不好的地方就是会出现一个非常巨大的build.gradle。 具体那种方式更好则仁者见仁的问题了。

例如目录结构

```
core/src/main/java目录包含core模块的源代码。
core/src/test/java目录包含core模块的单元测试。
app/src/main/java目录包含app模块的源代码。
app/src/main/resources目录包含app模块的资源文件。
```
对应的我们定义project如下
```groove
subprojects {
    apply plugin: 'java'
 
    repositories {
        mavenCentral()
    }
}

project(':app') {
    
}
 
project(':core') {
   
}
```
这样就可以看到project的具体情况如下
```
> gradle projects
:projects
 
------------------------------------------------------------
Root project
------------------------------------------------------------
 
Root project 'multi-project-build'
+--- Project ':app'
--- Project ':core'
 
To see a list of the tasks of a project, run gradle :tasks
For example, try running gradle :app:tasks
 
BUILD SUCCESSFUL
```

### 构建模型

这一个非常重要，或者也是我们用gradle和maven最大的不一样的地方。一开始让我也非常困惑。Maven是用过phase的方式定义构建流程的，而我们的不同操作比如jaxb binding，plugin的执行都是绑定到 不同的phase中。 默认情况下 maven 有一下 phases .

```xml
<phases>
 <phase>validate</phase>
 <phase>initialize</phase>
 <phase>generate-sources</phase>
 <phase>process-sources</phase>
 <phase>generate-resources</phase>
 <phase>process-resources</phase>
 <phase>compile</phase>
 <phase>process-classes</phase>
 <phase>generate-test-sources</phase>
 <phase>process-test-sources</phase>
 <phase>generate-test-resources</phase>
 <phase>process-test-resources</phase>
 <phase>test-compile</phase>
 <phase>process-test-classes</phase>
 <phase>test</phase>
 <phase>prepare-package</phase>
 <phase>package</phase>
 <phase>pre-integration-test</phase>
 <phase>integration-test</phase>
 <phase>post-integration-test</phase>
 <phase>verify</phase>
 <phase>install</phase>
 <phase>deploy</phase>
</phases>
```

这种定义的好处是非常的清晰，类似于设计模式中的模板类，在使用的时候只要在不同的pahse中加上我们要做的操作就好了。 不好的地方不太能并行操作， 比如generate-source和generate-resouce很多的时候是可以并行做的。 于是gradle 在设计的时候又回到了 ANT 的那种方式， 由用户自己定义task，然后在project中定义执行的方式。 这样会比maven灵活很多，但是复杂度和易读性要相对差很多。 所以在这一点上个人更加喜欢maven的方式。

在一点上为了更加容易理解，我们来看一下文档的说明

> 每一个构建都是由一个或多个 projects 构成的. 一个 project 到底代表什么取决于你想用 Gradle 做什么. 举个例子, 一个 project 可以代表一个 JAR 或者一个网页应用. 它也可能代表一个发布的 ZIP 压缩包, 这个 ZIP 可能是由许多其他项目的 JARs 构成的. 但是一个 project 不一定非要代表被构建的某个东西. 它可以代表一件**要做的事, 比如部署你的应用.
> 
> 不要担心现在看不懂这些说明. Gradle 的合约构建可以让你来具体定义一个 project 到底该做什么.
> 
> 每一个 project 是由一个或多个 tasks 构成的. 一个 task 代表一些更加细化的构建. 可能是编译一些 classes, 创建一个 JAR, 生成 javadoc, 或者生成某个目录的压缩文件.

```groove 

#定义默认的任务
defaultTasks 'hello', 'intro'

task hello << {
    println 'Hello world!'
}

##任务依赖
task intro(dependsOn: hello) << {
    println "I'm Gradle"
}
```


此外每一个插件，例如java会定义自己的一系列任务。 最常用的任务就是 build. 这里很类似于maven的phases

```
> gradle build
:compileJava
:processResources
:classes
:jar
:assemble
:compileTestJava
:processTestResources
:testClasses
:test
:check
:build

BUILD SUCCESSFUL

Total time: 1 secs
```

### 总结 

总的来说个人觉得gradle的学习难度要比maven高，但是灵活度会高很多，现在的事实情况是maven基本可以解决当前编译环境下的所有问题，所以如果不是特别需求，应该不会强制转到gradle上面去




