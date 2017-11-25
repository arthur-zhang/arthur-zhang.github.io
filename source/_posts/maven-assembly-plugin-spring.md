---
layout: post
title:  "一个maven-assembly-plugin spring打包的问题"
date:   2017-10-23 21:17:43
categories: spring maven
---

今天用maven打了一个jar包，然后用`java -jar`来执行，一直出错

```
Exception in thread "main" org.springframework.beans.factory.parsing.BeanDefinitionParsingException: Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace [http://www.springframework.org/schema/context]
```

maven相关配置如下

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <appendAssemblyId>false</appendAssemblyId>
        <archive>
            <manifest>
                <mainClass>com.xxx.AppMainKt</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

检查了各种情况，一番google，发现了一个很坑的地方

这是 assembly 插件的一个 bug：http://jira.codehaus.org/browse/MASSEMBLY-360，它在对第三方打包时，对于 META-INF 下的 spring.handlers，spring.schemas 等多个同名文件进行了覆盖，而不是追加

我们这里的情况是因为，我们用了elastic-job，它引用了spring模块，assembly插件进行打包时，这些配置文件会相互覆盖掉。经验证，最新版maven-assembly-plugin依然有这个问题

解决的办法是用maven-shade-plugin，这个之前做APM的时候有大量用到

```xml
 <plugin>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.xxx.AppMainKt</mainClass>
                    </transformer>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.handlers</resource>
                    </transformer>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.schemas</resource>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```
实际把前后对比的jar中的spring.schemas spring.handlers,确实有不同，其中一个图如下

![](/images/mvn-spirng-1.png)

