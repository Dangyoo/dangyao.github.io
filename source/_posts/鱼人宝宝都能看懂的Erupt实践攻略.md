---
title: 鱼人宝宝都能看懂的Erupt实践攻略
date: 2023-03-30 16:17:16
categories: 实践
tags: Erupt
---
## 闲言碎语

最近工作中需要实现一个简单的表单式呈现工具，主要用来展现一些元数据、业务流程数据，然后管理一些表和业务流程的关系等

主要需要的能力是能够做表单展示、能够编辑内容、能够数据互通（非手动）

本来考虑的是用鹅的微搭，结果搞了半天发现这东西主要是用来快速实现小程序的，其他东西就是做了个样子，数据导入导出API？那是什么东西？

所以考虑其他工具，之前用过明道云，算是比较成熟且好用的产品了，可惜要花钱，不是我能决定的

之前搭过Django后台管理，实现过一些需求，不过这次领导推荐了Erupt，说自己玩过几次，那我也来玩玩

<!--more-->

## 本地跑Demo

第一步肯定是先在本地搭一个，然后再去服务器上搭（谁成想领导说你怎么不用Docker，要什么服务器）

[官方文档](https://www.yuque.com/erupts/erupt/foa2bt)

跟着官方的部署步骤进行操作

1. 因为后续正式使用是用的MySQL，所以首先在本地装好MySQL，版本影响不大，但是8.0和常见5.0的语法不太一样[MySQL安装](https://www.runoob.com/mysql/mysql-install.html)，新建一个database以供项目使用，示例中用的是`erupt`
2. 在[Spring Initializr](https://start.spring.io/)创建一个项目。这里左侧Project选择`Maven`，Language选择`Java`，Spring Boot选择`2.7.12`（暂时3.x的支持没有2.x的好），Java版本选`17`，然后下方选择`GENERATE`生成一个demo.jar压缩包并下载
3. 解压缩，在IDEA中打开为Maven工程，这一步如果操作错了导致打开后IDEA右侧没有Maven选项，可以参考[本文](https://cloud.tencent.com/developer/article/1882517)进行补救
4. 有几个关键的文件，一个是`pom.xml`，在里边添加依赖包，例如erupt的UI页面包、用户权限包，或者Java连接MySQL的依赖包等；另一个是`resources`文件夹，里边存的是后面会用到的配置信息、静态文件等等资源
5. 根据官方文档在`pom.xml`添加所需的依赖包，将示例中的`${erupt.version}`改为具体的版本号如`1.11.7`
    ``` xml
    <!--例如一个MySQL5的依赖包-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.49</version>
        <scope>runtime</scope>
    </dependency>
    ```
   修改后会在IDEA右上角出现一个`m`字符的图标，点击`Load Maven Changes`，可以下载对应依赖包，快捷键是`Command + Shift + I`
6. 根据官方文档在`resources`文件夹下新建`public`文件夹，并在其中添加`app.js`，`app.css`，`home.html`文件，按照示例（点击官方文档表格后边的`#`）复制内容进去
7. 根据官方文档在`resources`文件夹下新建`application.yml`文件（properties应该也行但是我没成功），注意数据库链接中，示例的`jdbc:mysql://127.0.0.1:3306/erupt`中的`erupt`是事先建好的database
8. 根据官方文档在Spring Boot入口类，也就是`demo.src.main.java.com.example.demo.DemoApplication`中，添加 @EntityScan、@EruptScan 注解，添加后如果IDEA中显示红色，则需要引入，鼠标放在红色上点击`Import class`即可，或者快捷键`Alt + Shift + Enter`
    ``` java
    import org.springframework.boot.autoconfigure.domain.EntityScan;
    import xyz.erupt.core.annotation.EruptScan;
    ```
9. 运行`DemoApplication`，可以选择在左侧目录树右键选择`Run`，也可以在文件中运行`main`方法
10. 启动后会自动在对应的database中新建一些表，然后在`http://localhost:8080`可以看到登录页，默认用户名密码都是`erupt`

过程中遇到什么问题优先查看[官方Q&A](https://www.yuque.com/erupts/erupt/vr4md2#PQuoK)，搜索引擎搜到对应内容挺不容易，也可以考虑去B站看视频操作

## 数据表的创建

按照官方[入门示例](https://www.yuque.com/erupts/erupt/waztcb)开始通过创建Class来给数据库建表，并通过管理菜单绑定到页面上

1. 在入口类同级别创建model文件夹，也就是`demo.src.main.java.com.example.demo.model`，用来存放所有会用到的Class，一个Class对应一个可以在管理菜单进行绑定的`类型值`
2. 一些可能用到的语法，view和edit的各类配置项参考[官方文档](https://www.yuque.com/erupts/erupt/gec455)以及[示例](https://www.erupt.xyz/#!/contrast)
   ```java
   private Long id;  # 字段id的类型为Long
   
   @Lob
   private String comment;  # 字段comment的类型为text
   
   views = @View(title = "名称", sortable = true)  # 字段可以点击表头进行排序
   edit = @Edit(title = "名称", readonly = @Readonly, search = @Search(vague = true))  # 字段不可编辑，可以进行模糊搜索（页面上方会有虚线搜索框，false为精确搜索，搜索框为实线）
   ```

## 部署

本地把项目创建好并且导入部分数据玩熟了之后，开始考虑部署的问题，本来想着整个内部服务器上去操作以便就行了，但是领导非要让用Docker部署到k8s上，学吧有啥办法

### 啥是Docker

虽然之前听说过这，大概有个印象是能做到环境隔离，然后能快速部署，具体怎么实现，完全不知道，参考[文档](https://yeasy.gitbook.io/docker_practice/)学了半天基本搞明白怎么操作了

按照个人浅薄但够用的理解来说，Docker整个技术就是将某个应用所需的软件依赖编写在Dockerfile中，通过Docker引擎将所需的依赖打包到一个镜像image中，使得这个镜像可以在任何硬件上运行的过程

### 咋打包这个erupt项目

首先这个项目本身需要打包成一个jar包，为了运行这个jar包，需要一个能够运行java的JDK，然后最底层需要一个linux操作系统

那对于Dockerfile来说，最先应该引入一个linux操作系统镜像，然后引入一个对应版本的JDK镜像，然后将这个项目的jar包导入，然后运行就好了

```markdown
FROM xxx/jdk:17  # 这里先把linux+JDK打包成了一个镜像传到了仓库中
MAINTAINER dangyoo
EXPOSE 8080
ADD demo.jar /app.jar  # 把本地的项目jar包导入
ENTRYPOINT ["java","-jar","/app.jar"]  # 运行这个包
```

在把项目打包成demo.jar的时候，最好使用Maven工具进行，IDEA本身的built会有很多依赖冲突的问题，在打包前最好将测试跳过，否则会连接数据库，如果数据库配置的是内网数据库，就会连不上报错

跳过测试设置：`Settings - Build, Execution, Deployment - Build Tools - Maven - Runner - Properties - Skip Tests`打勾

使用Maven打包：`右上角Maven - demo - Lifecycle - package`

打包后默认文件路径是在`demo/target/demo-0.0.1-SNAPSHOT.jar`这里
