---
title: maven插件学习
date: 2017-04-05 13:32:36
tags: maven, 插件, 打包
toc: true
categories: 工具学习
---
maven有三个生命周期:clean, default, site。 clean主要的目的是为了清理上次构建产生的文件, default主要是为了构建项目, site主要了为了搭建和发布项目web站点, 比如项目信心, 以便团队之间更好地交流。我们一般主要关注default声明周期, 这里会大致介绍下常用的phase:

|属性|介绍|默认绑定插件|
|:-|:-|:-|
|validate|验证项目是正确的||
|process-sources|将resources从主函数代码里面拷贝到输出文件夹中, 一般是target里面.|maven-resources-plugin|
|compile|对java进行编译产生class文件|maven-compiler-plugin|
|process-test-sources|将测试resources从测试函数代码里面拷贝到输出文件夹中.|maven-resources-plugin|
|test-compile|对测试环境java进行编译产生class文件|maven-compiler-plugin|
|test|对项目里面的单元测试进行测试, 测试代码不会被打包部署|maven-surefire-plugin|
|package|将编译的class打包成jar包|maven-jar-plugin|
|install|打包后将jar放入本地仓库.m2中|maven-install-plugin|
|deploy|打包后将jar放入远程仓库中|maven-install-plugin|
本文将主要主要对打包方式进行讲解。
## 打包方式
而打包的方式主要分为四种:默认jar打包、shade打包、assembly打包、source打包

|属性|介绍|默认绑定插件|查看使用参数|
|:-|:-|:-|:-|
|jar|仅仅将class文件打进包中, 不包含依赖的jar包, 不是可执行jar包, 缺少依赖的jar class|maven-jar-plugin|mvn jar:help|
|shade|在jar的基础上, 将class的依赖也打包进来|maven-shade-plugin|mvn shade:help|
|assembly|当前节点所拥有的线程|maven-assembly-plugin|mvn assembly:help|
|source|把当前源码打进包|maven-source-plugin|mvn source:help|
针对每种goal具体使用, 还可以使用类似命令查看: `mvn shade:help -Ddetail=true -Dgoal=<goal-name>`
## shade打包
shade打包主要是将依赖全部放入jar文件中, 以便可以直接成为可执行的jar包, 这里将以一个示例进行讲解:
```
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <configuration>
                    <!-- put your configurations here -->
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <finalName>${project.artifactId}-${project.version}-with-dependencies</finalName>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>MyQueryMojo</mainClass>
                                    <manifestEntries>
                                        <ha>s</ha>
                                    </manifestEntries>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```
可以看到, 我们定义了引入的plugin shade的版本号。接下来我们将对每个标签进行详细说明:
1 configuration:配置信息, 我们可以对所有executions定义一个默认的配置信息, 当然在具体的execution定义执行计划, 是可以覆盖这里的默认配置的。
2 execution: 相当于一个执行计划, 都放在executions里面。

|配置|介绍|
|:-|:-|
|phase|我们需要定义这个执行计划在生命阶段执行, 默认是在package阶段执行。|
|goals|执行目标, 我们可以通过mvn shade:help可以该阶段的所有目标, 默认为shade。|
|configuration|对打包可以配置一些参数|
在configuration中, 我们可以通过finalName定义产生的jar名称, 通过transformer可以将某些artifactId按照某种方式进行整合到jar中, 我们以常用的示例进行介绍ManifestResourceTransformer, ManifestResourceTransformer允许我们向打包文件里面的MANIFEST.MF添加配置属性, 属性元素都要放在manifestEntries里面, mainClass除外, 可以放在外层。mvn还提供许多针对资源进行整合的Transformer, 详情可参考<a href="http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#">官方文档</a>。
同时在configuration中还可以配置别的参数, 比如outputDirectory定义了jar存放位置。 include/exclude: 可以从某个jar包中排除某些类给排除掉。 所有参数可以<a href="http://maven.apache.org/plugins/maven-shade-plugin/shade-mojo.html">参考</a>;.

## assembly打包
Maven Assembly Plugin 和 Shade Plugin 都可以用来在构建单一 Jar 包时，将所有 Dependency 打入这个最终生成的 Jar 中去。但是两者在具体的行为上有所不同：Assembly 插件不仅会将 Dependency 中的 Class 文件打入最终的 Jar 包，还会将 Dependency 中的资源文件，诸如 properties 文件打入最终的 Jar 包。当项目和其 Dependency 中有同名的资源文件是，就会发生冲突，项目中的同名文件便不会加入到最终的 Jar 包中。如果这个文件是一个关键的配置文件，便会导致问题。而 Shade Plugin 不存在这样的问题。(<a href="https://ssailyang.iteye.com/blog/1745589">参考</a>)

## 自定义插件
我们还可以自定义插件, 具体可参考文档:
https://www.cnblogs.com/oscar1987121/p/10959083.html
http://maven.apache.org/plugins/maven-shade-plugin/examples/use-shader-other-impl.html

## 使用经验
1.若在项目app0使用shade/assembly打包后产生jar1, 则第三方依赖将会全部转成class文件, 比如fastJson, fastJson的版本号将在这里全部被抹掉。若别的项目app1引用app0 jar1包。 那么在app1中, 通过maven dependency将看不到fastJason版本号,也只能看到fastJson的class文件。
比如我们在app0配置的pom.xml如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>dep1</groupId>
    <artifactId>dep1</artifactId>
    <version>1.0.3-SNAPSHOT</version>
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.1.43</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.1</version>
                <executions>
                    <execution>
                        <id>ds</id>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
但是我们在app1中看到的app0的配置文件却是这样的
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>dep1</groupId>
  <artifactId>dep1</artifactId>
  <version>1.0.3-SNAPSHOT</version>
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-shade-plugin</artifactId>
        <version>2.4.1</version>
        <executions>
          <execution>
            <id>ds</id>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```
我们通过mvn dependency:tree也完全看不到fastJson的版本号, 可以看到, app1里面看到app0的配置文件里面全部没有了fastJson的信息了(但是实际引入的fastJson版本, 其实在jar包内META-INF/maven/com.alibaba/fastjson路径里面的pom.xml可以看到)。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/mvn_shade1.png" height="200" width="400"/>
这样的话, 尽管jar包有冲突, 比如下图的那个冲突, java选择的那个版本是dep1后面的那个fastJson, 尽管下面那个fastJson路径更短(shade打包方式, 使fastJson相关class全部打到dep1里面了, 索引路径更短, 而且dep1先声明的)。
2.通过默认jar方式, 打包内容将不会被包含第三方依赖。那么还是可以看到app0里面引入的fastJons版本信息。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/mvn_shade2.png" height="200" width="550"/>
3.若出现jar包冲突的话, 那么选取最短路径上的jar, 同时会覆盖被冲突的包。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/mvn_shade3.png" height="200" width="400"/>
比如这里将会选择下面那个fastJson版本。

## 在mvn中查看第三方依赖
1. 通过mvn dependency:tree/list查看
2. 在Intellij IDea的右边-> Maven Projects右键 -> show Dependencies可查看。