---
layout: post
title: Spark学习(2)之Intellij Idea和Maven部署spark应用
date: 2015-11-11 12:29:30
categories: 大数据
tags: Spark
---
# Spark学习(2)之Intellij Idea和Maven部署spark应用

## 简介

本章主要介绍如何在Intellij Idea里面使用Maven部署spark应用。一开始我尝试使用sbt来进行部署，结果sbt速度奇慢无比，实在无法忍受, 所以最后还是打算使用maven。

首先介绍下环境:

* Intellij Idea 14.1.5
* JDK 1.8.0_25
* Scala 2.10.4 (前文提到spark-1.5.0 只支持scala 2.10 还不能支持2.11)
* Spark 1.5.0
* Hadoop 2.6.0
* Maven 3.3.3
* OS Mac 10.10.4 x86_64

## 部署

### 使用Maven直接构建Scala工程

使用maven构建scala工程，我们需要在创建的时候使用到scala-archetype-simple模版，如果不指定该模版的版本，默认是使用最新版的。目前最新版的scala-archetype-simple为1.5，其中的Scala版本是2.10.0，已经不是最新版的Scala了. 如果要修改Scala的版本，就需要根据自己的情况去修改pom.xml文件里面的Scala、ScalaTest、Surefire以及scala-maven-plugin的版本。
{% highlight bash linenos %}
mvn archetype:generate -DarchetypeGroupId=net.alchim31.maven \
	-DarchetypeArtifactId=scala-archetype-simple \
	-DremoteRepositories=http://scala-tools.org/repo-releases \
	-DarchetypeVersion=1.5 \
	-DgroupId=com.lamborryan \
	-DartifactId=spark \
	-Dversion=1.0-SNAPSHOT \
	-Dpackage=com.lamborryan
{% endhighlight bash %}
### 使用Intellij构建Scala工程

使用Intellij构建Scala工程时候，跟建一般的Maven工程一样，但是需要注意在一开始勾选［"Create from archetype", 由于14.1.5版本的Intelij自带的scala-archetype-simple是1.2, 所以我们点击["Add Archetype"], 然后输入
{% highlight bash linenos %}
GroupId: net.alchim31.maven
ArtifactId: scala-archetype-simple
Version: 1.5
{% endhighlight bash %}
随后选中该archetype, 就可以正常的构建Scala工程了, pom文件如下。
{% highlight xml linenos %}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>lamborryan</groupId>
  <artifactId>sparkTest</artifactId>
  <version>1.0-SNAPSHOT</version>
  <name>${project.artifactId}</name>
  <description>My wonderfull scala app</description>
  <inceptionYear>2010</inceptionYear>
  <licenses>
    <license>
      <name>My License</name>
      <url>http://....</url>
      <distribution>repo</distribution>
    </license>
  </licenses>

  <properties>
    <maven.compiler.source>1.6</maven.compiler.source>
    <maven.compiler.target>1.6</maven.compiler.target>
    <encoding>UTF-8</encoding>
    <scala.tools.version>2.10</scala.tools.version>
    <scala.version>2.10.0</scala.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.scala-lang</groupId>
      <artifactId>scala-library</artifactId>
      <version>${scala.version}</version>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.specs2</groupId>
      <artifactId>specs2_${scala.tools.version}</artifactId>
      <version>1.13</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.scalatest</groupId>
      <artifactId>scalatest_${scala.tools.version}</artifactId>
      <version>2.0.M6-SNAP8</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <sourceDirectory>src/main/scala</sourceDirectory>
    <testSourceDirectory>src/test/scala</testSourceDirectory>
    <plugins>
      <plugin>
        <!-- see http://davidb.github.com/scala-maven-plugin -->
        <groupId>net.alchim31.maven</groupId>
        <artifactId>scala-maven-plugin</artifactId>
        <version>3.1.3</version>
        <executions>
          <execution>
            <goals>
              <goal>compile</goal>
              <goal>testCompile</goal>
            </goals>
            <configuration>
              <args>
                <arg>-make:transitive</arg>
                <arg>-dependencyfile</arg>
                <arg>${project.build.directory}/.scala_dependencies</arg>
              </args>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>2.13</version>
        <configuration>
          <useFile>false</useFile>
          <disableXmlReport>true</disableXmlReport>
          <!-- If you have classpath issue like NoDefClassError,... -->
          <!-- useManifestOnlyJar>false</useManifestOnlyJar -->
          <includes>
            <include>**/*Test.*</include>
            <include>**/*Suite.*</include>
          </includes>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
{% endhighlight xml %}
由于我使用的Scala是2.10.4, 所以需要修改scala.version为2.10.4.
{% highlight xml linenos %}
<scala.version>2.10.4</scala.version>
{% endhighlight xml %}
如果不修改运行该程序时候，会出现以下错误
{% highlight java linenos %}
Error:scalac: Error: org.jetbrains.jps.incremental.scala.remote.ServerException
Error compiling sbt component 'compiler-interface-2.10.0-52.0'
	at sbt.compiler.AnalyzingCompiler$$anonfun$compileSources$1$$anonfun$apply$2.apply(AnalyzingCompiler.scala:145)
	at sbt.compiler.AnalyzingCompiler$$anonfun$compileSources$1$$anonfun$apply$2.apply(AnalyzingCompiler.scala:142)
	at sbt.IO$.withTemporaryDirectory(IO.scala:285)
	at sbt.compiler.AnalyzingCompiler$$anonfun$compileSources$1.apply(AnalyzingCompiler.scala:142)
	at sbt.compiler.AnalyzingCompiler$$anonfun$compileSources$1.apply(AnalyzingCompiler.scala:139)
	at sbt.IO$.withTemporaryDirectory(IO.scala:285)
	at sbt.compiler.AnalyzingCompiler$.compileSources(AnalyzingCompiler.scala:139)
	at sbt.compiler.IC$.compileInterfaceJar(IncrementalCompiler.scala:33)
	at org.jetbrains.jps.incremental.scala.local.CompilerFactoryImpl$.org$jetbrains$jps$incremental$scala$local$CompilerFactoryImpl$$getOrCompileInterfaceJar(CompilerFactoryImpl.scala:87)
	at org.jetbrains.jps.incremental.scala.local.CompilerFactoryImpl$$anonfun$getScalac$1.apply(CompilerFactoryImpl.scala:44)
	at org.jetbrains.jps.incremental.scala.local.CompilerFactoryImpl$$anonfun$getScalac$1.apply(CompilerFactoryImpl.scala:43)
	at scala.Option.map(Option.scala:145)
	at org.jetbrains.jps.incremental.scala.local.CompilerFactoryImpl.getScalac(CompilerFactoryImpl.scala:43)
	at org.jetbrains.jps.incremental.scala.local.CompilerFactoryImpl.createCompiler(CompilerFactoryImpl.scala:22)
	at org.jetbrains.jps.incremental.scala.local.CachingFactory$$anonfun$createCompiler$1.apply(CachingFactory.scala:24)
	at org.jetbrains.jps.incremental.scala.local.CachingFactory$$anonfun$createCompiler$1.apply(CachingFactory.scala:24)
	at org.jetbrains.jps.incremental.scala.local.Cache$$anonfun$getOrUpdate$2.apply(Cache.scala:20)
	at scala.Option.getOrElse(Option.scala:120)
	at org.jetbrains.jps.incremental.scala.local.Cache.getOrUpdate(Cache.scala:19)
	at org.jetbrains.jps.incremental.scala.local.CachingFactory.createCompiler(CachingFactory.scala:23)
	at org.jetbrains.jps.incremental.scala.local.LocalServer.compile(LocalServer.scala:22)
	at org.jetbrains.jps.incremental.scala.remote.Main$.make(Main.scala:62)
	at org.jetbrains.jps.incremental.scala.remote.Main$.nailMain(Main.scala:20)
	at org.jetbrains.jps.incremental.scala.remote.Main.nailMain(Main.scala)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:483)
	at com.martiansoftware.nailgun.NGSession.run(NGSession.java:319)
{% endhighlight java %}
### 编写park应用
进行spark应用编写, 只需加入依赖即可
{% highlight xml linenos %}
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-core_2.10</artifactId>
  <version>1.5.0</version>
</dependency>
{% endhighlight xml %}
spark demo:
{% highlight java linenos %}
package com.lamborryan.spark

import org.apache.spark.SparkContext

object BasicMap{
    def main(args: Array[String]): Unit ={
        val sc = new SparkContext("local", "BasicMap")
        val input = sc.parallelize(List(1,2,3,4))
        val result = input.map(x => x*x)
        println(result.collect().mkString(","))
    }
}
{% endhighlight java %}
至此，spark的开发环境算是部署好了，虽然sbt会更适合scala，奈何实在太慢。

本文完
