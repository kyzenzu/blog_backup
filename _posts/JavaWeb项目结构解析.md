---
title: JavaWeb项目结构解析
date: 2024-10-28
tags: 
  - Java
---

环境：

* `Eclipse`: 2023-12
* `Maven`: apache-maven-3.6.1
* `Tomcat`: apache-tomcat-9.0.86
* JDK: 11.0.18

本次的项目为制作一个简易登录系统，项目名称为 **login-system**

选择的项目骨架(archetype)是

| Group Id                    | Artifact Id            | Version |
| --------------------------- | ---------------------- | ------- |
| org.apache.maven.archetypes | maven-archetype-webapp | 1.0     |

第一次创建后的项目目录结构如下

~~~shell
login-system
├── pom.xml
├── src
│   └── main
│       ├── resources
│       └── webapp
│           ├── index.jsp
│           └── WEB-INF
│               └── web.xml
└── target
~~~

其中这个target目录没什么用，暂时不考虑

这些目录有什么作用呢，等会讲

其实这个骨架并不完整，我们还需要添加自己添加一些目录使其完整

~~~shell
login-system
├── pom.xml
├── src
│   └── main
│       ├── java
│       │    └── servlet
│       │        ├── HelloServlet.java
│       │        └── LoginServlet.java
│       ├── resources
│       └── webapp
│           ├── index.jsp
│           └── WEB-INF
│               └── web.xml
└── target
~~~

像这样，其实只是在`login-system/src/main/`目录下添加了一个`java`目录，在整个项目中这个目录会被自动标记为 **Source Folder** 也就是java编译的起点就是在这里（后面web.xml中配置\<servlet\>的\<servlet-class\>也是以这个目录为起点）。其实呢，如果不添加这个目录，`eclipse`还不让你新建 **Servlet**，它指明了需要在一个`java`目录下写 **Servlet**

(好像完整的骨架在 main目录下还有个 test目录，自己加一下吧)

> 在  ./src/main目录下新建的目录都会被自动标记为 **Source Folder**，此后这些目录下的 .java文件都被编译为 .class文件，然后存放在 tomcat/webapps/[webapp]/WEB-INF/classes目录下
> 当然，resources作为 **Source Folder**，其目录下的资源文件会被直接放在classes目录下

然后我还添加了一个`servlet`包，目的是为了分类，在这个包下的类都是 **Servlet**类

在了解整个项目各个目录，各个文件夹的作用前，我们需要先了解`tomcat`目录下各个文件夹的作用，因为整个项目最终是要部署在`tomcat`下的

~~~shell
tomcat
├── conf
├── logs
├── temp
├── work
└── webapps
    ├── ROOT
    └── login-system
~~~

这里为了更好的解释，我对整个`tomcat`的项目结构做了删改和顺序调整，`tomcat`下的一级目录以及各个作用这里我就不多做解释，我们这次的重点是`webapps`目录

在这个目录下除了自带的`ROOT`目录外还有刚刚创建的`login-system`项目，这里我要挑明的说

`tomcat`为什么称为容器，我觉得关键就在于这个`webapps`目录。我们写的项目可以说是一个**webapp**，这个 **webapp**需要依托在`tomcat`下才能展示出来，一个`tomcat`不可能只运行一个 **webapp**，那样太浪费资源了，所以从这里可以看出来，一个`tomcat`可以运行多个 **webapp**，在这里`ROOT`是一个 **webapp**，`login-system`也是一个 **webapp**，它们都被放在`tomcat`的`webapps`目录下。

这些 **webapp**被挂在了`tomcat`的`webapps`中，所以说`tomcat`是一个容器，它容纳了这些 **webapp**

让我们仔细看一下这个文件目录结构

~~~shell
webapps
├── login-system
│   ├── index.jsp
│   ├── META-INF
│   └── WEB-INF
│       ├── classes
│       │   └── servlet
│       │       ├── HelloServlet.class
│       │       └── LoginServlet.class
│       └── web.xml
└── ROOT
    └── WEB-INF
        └── web.xml
~~~

将其中的`login-system`模块与我们的源项目进行比较

~~~shell
login-system
├── pom.xml
├── src
│   └── main
│       ├── java
│       │    └── servlet
│       │        ├── HelloServlet.java
│       │        └── LoginServlet.java
│       ├── resources
│       └── webapp
│           ├── index.jsp
│           └── WEB-INF
│               └── web.xml
└── target
~~~

会发现极其的相似，这个转化过程是这样的。

我们写一个项目，总会有一个输出结果，而现在这个项目输出结果的重点就是 `webapp`目录

在转换为能够挂载在`tomcat`下的 **webapp**前，`eclipse`做的工作就是处理`java`目录（所有 **Source Folder**），将这里面的 .java文件全部编译为 .class文件后把这些字节码文件一起存放在一个新建的`classes`目录下，保持原先各个包的结构，然后将整个 `classes`目录移到与`main`同级的`webapp`目录下的`WEB-INF`目录下。然后将我们写的项目中的`webapp`目录提取出来，把这个目录名改为项目名。最后把这个`login-system`目录作为一个 **webapp**迁移到 `tomcat`的`webapps`目录下

这是项目的最终结构

~~~shell
login-system/
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   ├── pojo
    │   │   │   ├── UserDB.java
    │   │   │   └── User.java
    │   │   ├── servlet
    │   │   │   ├── IndexServlet.java
    │   │   │   ├── LoginServlet.java
    │   │   │   └── LogoutServlet.java
    │   │   └── xmlParse
    │   │       ├── UserXMLHandler.java
    │   │       └── XMLParser.java
    │   ├── resources
    │   │   └── UserDB.xml
    │   └── webapp
    │       ├── css
    │       │   ├── index.css
    │       │   └── login.css
    │       ├── index.jsp
    │       ├── js
    │       │   └── index.js
    │       ├── login.html
    │       └── WEB-INF
    │           └── web.xml
    └── test
        ├── java
        └── resources

~~~

最后挂载到tomcat

~~~shell
webapps/
├── login-system
│   ├── css
│   │   ├── index.css
│   │   └── login.css
│   ├── index.jsp
│   ├── js
│   │   └── index.js
│   ├── login.html
│   ├── META-INF
│   │   ├── MANIFEST.MF
│   │   └── maven
│   │       └── com.kyzen
│   │           └── login-system
│   │               ├── pom.properties
│   │               └── pom.xml
│   └── WEB-INF
│       ├── classes
│       │   ├── pojo
│       │   │   └── User.class
│       │   ├── servlet
│       │   │   ├── IndexServlet.class
│       │   │   ├── LoginServlet.class
│       │   │   └── LogoutServlet.class
│       │   ├── UserDB.xml
│       │   └── xmlParse
│       │       ├── UserXMLHandler.class
│       │       └── XMLParser.class
│       └── web.xml
└── ROOT
    └── WEB-INF
        └── web.xml
~~~

