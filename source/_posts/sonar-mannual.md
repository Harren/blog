---
title: Sonar代码质量分析使用
date: 2017-11-17 18:47:09
categories:
- 技术杂谈
tags:
- 代码质量分析
---
## Sonar概述
Sonar是一个用于代码质量管理的开放平台。通过插件机制，Sonar 可以集成不同的测试工具，代码分析工具，以及持续集成工具。

与持续集成工具（例如 Hudson/Jenkins 等）不同，Sonar 并不是简单地把不同的代码检查工具结果（例如 FindBugs，PMD 等）直接显示在 Web 页面上，而是通过不同的插件对这些结果进行再加工处理，通过量化的方式度量代码质量的变化，从而可以方便地对不同规模和种类的工程进行代码质量管理。

![ffmpeg](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwouel8vmj21kw0ua0zk.jpg)
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

## Sonar数据库配置
Sonar 默认使用的是 Derby 数据库，但这个数据库一般用于评估版本或者测试用途。商用及对数据库要求较高时，建议使用其他数据库。Sonar 可以支持大多数主流关系型数据库（例如 Microsoft SQL Server, MySQL, Oracle, PostgreSQL 等，本文以 MySQL 为例说明如何更改 Sonar 的数据库设置:

```bash,monokai
mysql> CREATE USER sonar IDENTIFIED BY 'sonar';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'sonar'@'localhost' IDENTIFIED BY 'sonar' WITH GRANT OPTION;
```
配置好数据库权限后，修改 sonar.properties 文件配置如下(数据库用户名密码为：sonar/sonar)
![sonar_config](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwoukblqyj20nr07s404.jpg)
配置后重新启动sonar即可，此次因为需要创建数据库，重启较慢，重启成功后会在数据库中生成sonar相关的表。

## 使用Sonar进行代码质量管理
由于本人主要使用 Java 作为开发工具，主要介绍对 Java 代码代码质量管理，sonar默认是不需要登录权限认证就可以上传代码监测报告的，在生产环境中需要打开用户权限，在[配置]->[通用配置]->[权限]中打开即可，如下图所示
![sonar_auth](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwowmxps5j20wi06mq3r.jpg)
### Maven集成Sonar
Maven 插件会自动把所需数据（如单元测试结果、静态检测结果等）上传到 Sonar 服务器上，需要说明的是，关于 Sonar 的配置并不在每个工程的 pom.xml 文件里，而是在 Maven 的配置文件 settings.xml 文件里，涉及到以下 maven 配置项目:

| 配置项 | 作用 | 默认值 |
|--------|---------|-------|
| sonar.host.url | sonar服务器地址文件 | http://127.0.0.1:9000|
| sonar.login | sonar用户名 | 用户或者token(如果利用token则不用密码，推荐这种方式登陆) |
| sonar.password | sonar密码 | admin |
#### sonar生成登陆token
为了强化安全，避免直接暴露出分析用户的密码，使用用户令牌来代替用户登陆,如下图
![sonar_auth](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwouqs7woj21d20vkk4l.jpg)

#### Maven配置文件修改
具体配置如下:

```bash
<profile>
        <id>sonar</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <sonar.host.url>http://localhost:9000</sonar.host.url>
            <sonar.login>93a87b9d138cd836b65c2c52fc5578fc71270707</sonar.login>
        </properties>
    </profile>
```
编译命令如下

```bash
mvn clean install
mvn sonar:sonar
```
将 Soanr 所需要的数据上传到 Sonar 服务器上之后，Sonar 安装的插件会对这些数据进行分析和处理，并以各种方式显示给用户，从而使用户方便地对代码质量的监测和管理，之后可以在sonar服务器得到上此次提交代码分析的结果信息，包括代码覆盖率等信息。
![sonar_auth](http://wx1.sinaimg.cn/mw1024/78d85414ly1fkwouu8uz2j21h00zugr6.jpg)


### Sonar配置Gitlab可持续集成
#### 自动化脚本集成
如果对项目有持续即成的需要，同时项目是利用gitlab进行托管，给项目配置好runner，则需要在项目目录下建.gitlab-ci.yml文件来自定义命令，具体参照[gitlab-ci使用](https://segmentfault.com/a/1190000006120164)简介 ，这样每次提交的时候都会自动运行脚本，并将生成的报告直接上传到服务器，下面提供一个参考脚本如下

```yml
# 定义 stages
stages:
 - review
 - analyze
# 定义 review
job1:
 stage: review
 script:
 - /usr/local/sbin/code_analyze --preview    #这条命令主要是将代码分析的信息输出到gitlab的Discussions，只会在分支上运行
 except:
 - master
# 定义 analyze
job2:
 stage: analyze
 script:
 - /usr/local/sbin/code_analyze             #这条命令主要是将代码分析的信息同步到sonar服务器，只针对master
 only:
 - master
```

code_analyze为脚本文件，主要是对git项目内容进行打包并将相应的代码分析报告上传到sonar服务器，其内容如下

```bash
#!/bin/bash
set -e
echo "test"
if [ "$1" = "--preview" ];then
    echo ${CI_BUILD_REF}
    echo ${CI_BUILD_REF_NAME}
    echo ${CI_PROJECT_DIR}
    echo ${CI_PROJECT_ID}
	sonar_prop="-Dsonar.issuesReport.console.enable=true -Dsonar.analysis.mode=preview  -Dsonar.preview.excludePlugins=issueassign,scmstats -Dsonar.gitlab.commit_sha=${CI_BUILD_REF} -Dsonar.gitlab.ref=${CI_BUILD_REF_NAME} -Dsonar.gitlab.project_id=${CI_PROJECT_ID}"
    if [ -f "gradlew" ]; then
	    ./gradlew clean check sonarqube $sonar_prop
    else
	    mvn --batch-mode clean verify sonar:sonar $sonar_prop
    fi
else
	sonar_prop="-Dsonar.preview.excludePlugins=gitlab"
	if [ -f "gradlew" ]; then
		./gradlew clean check sonarqube $sonar_prop
	else
        #mvn clean org.codehaus.mojo:cobertura-maven-plugin:2.7:cobertura -Dcobertura.report.format=xml -Dcobertura.aggregate=true
		mvn --batch-mode verify sonar:sonar $sonar_prop
    fi
fi

```

#### Sonar写入Gitlab Discussion
如果希望直接在 gitlab 的每次 Merge_requesrs 中在 gitlab 的 Discussion 中显示出此次代码分析的结果，效果如下
![sonar_auth](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwowereo8j21fg0j6dk1.jpg)

首先，需要gitlab给sonar授权，在 gitlab 中 ［User Settings］中生成 Access Tokens 
![sonar_auth](http://wx2.sinaimg.cn/mw1024/78d85414ly1fkwowhpph2j21kw0ubdnt.jpg)

然后在 sonar 的配置页写入token, 如下，由于申请的 token 的作用域为 api, sonar里面配置 scope 为 api
![sonar_auth](http://wx1.sinaimg.cn/mw1024/78d85414ly1fkwowk9xhsj20x006wwfb.jpg)
![sonar_auth](http://wx3.sinaimg.cn/mw1024/78d85414ly1fkwowmxps5j20wi06mq3r.jpg)

配置完后，在gitlab上之执行Merge Request时候会出发自动构建，同时生成相应的isscus。

## 参考资料 
1. [Sonar官方文档](https://docs.sonarqube.org/display/SONAR/Documentation)
2. [Sonar插件下载地址](https://docs.sonarqube.org/display/PLUG/Plugin+Library)



