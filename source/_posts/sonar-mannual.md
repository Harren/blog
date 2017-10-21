---
title: Sonar代码质量分析使用
date: 2017-10-17 18:47:09
tags:
---
## Sonar概述
Sonar是一个用于代码质量管理的开放平台。通过插件机制，Sonar 可以集成不同的测试工具，代码分析工具，以及持续集成工具。

与持续集成工具（例如 Hudson/Jenkins 等）不同，Sonar 并不是简单地把不同的代码检查工具结果（例如 FindBugs，PMD 等）直接显示在 Web 页面上，而是通过不同的插件对这些结果进行再加工处理，通过量化的方式度量代码质量的变化，从而可以方便地对不同规模和种类的工程进行代码质量管理。

![ffmpeg](http://wx3.sinaimg.cn/mw690/78d85414ly1fknfudniaej21kw0uan49.jpg)
<!-- more -->

在对其他工具的支持方面，Sonar 不仅提供了对 IDE 的支持，可以在 Eclipse 和 IntelliJ IDEA 这些工具里联机查看结果；同时 Sonar 还对大量的持续集成工具提供了接口支持，可以很方便地在持续集成中使用 Sonar。
此外，Sonar的插件还可以对 Java 以外的其他编程语言提供支持，对国际化以及报告文档化也有良好的支持。

## Sonar安装
本文主要介绍 Sonar 的使用方法，直接到[Sonar官网](https://www.sonarqube.org)下载最近的发型包即可，本文使用的为最新的版本为6.5(推荐使用最新版)，其源代码可以参考[github地址](https://github.com/SonarSource/sonarqube)。

下载zip包后，直接解压，然后根据应用服务器环境启动 bin 目录下的脚本即可


```
bin/linux-x86-64/sonar.sh -h          // 显示所有命令
bin/linux-x86-64/sonar.sh start       // 启动，默认为9000端口
```

然后在浏览器中访问 http://localhost:9000 即可, 初始化用户名和密码为: admin/admin



