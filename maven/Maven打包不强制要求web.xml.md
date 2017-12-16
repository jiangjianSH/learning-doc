# maven 打包war文件，去掉web.xml文件存在的强校验

##问题

Servlet 3.0之后对`web.xml`文件是否定义没有强制要求，如果我们使用maven package 我们的项目的时候，会出现如下的错误信息:

```terminal
Error assembling WAR: webxml attribute is required (or pre-existing WEB-INF/web.xml if executing in update mode)
```

## 解决

默认的maven-war-plugin会对`web.xml`是否定义进行强制的校验，这时我们需要更新我们的插件版本，使其能够忽略对`web.xml`强校验，修正方式在项目pom.xml的`<plugins>`标签内增加如下的配置片段:

```xml
<plugin>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <failOnMissingWebXml>false</failOnMissingWebXml>
    </configuration>
</plugin>
```