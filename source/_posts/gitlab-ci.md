---
title: Gitlab-CI持续集成
date: 2017-09-27 13:57:27
categories:
- 技术杂谈
tags:
- 持续集成
---
持续集成 (Continuous Integration) 是一种软件开发实践，即团队开发成员经常集成他们的工作，通过每个成员每天至少集成一次，也就意味着每天可能会发生多次集成。每次集成都通过自动化的构建（包括编译，发布，自动化测试）来验证，从而尽早地发现集成错误，同样在《Code Complete》里提到了，对于持续集成（在书中，Steve McConnell使用Incremental Integration的术语）有以下几点好处：

* 易于定位错误，当集成失败后很容易及时的找到问题所在
* 模拟生产环境的自动测试
* 与其它工具结合的持续代码改进，比如Sonar,findbug等
* 方便进行Code Review

## Gitlab-Runner
在Gitlab-CI中有一个叫 Runner 的概念, 按照官方定义, Runner一共有三种类型

* 本地Runner (优点:部署方便, 缺点:使用的是开发机器的资源， Runner服务无法持久化)
* 普通的服务器上的Runner (本文主要用的这种Runner)
* 基于Docker的Runner (没有较好的docker环境，如果存在docker集群的话推荐使用)

GitLab-Runner类似于一个用来执行软件集成脚本的东西，它负责将Git仓库的代码 Clone 到 Runner所在到服务器上，然后运行软件集成脚本，同时将脚本输出的内容写回Git,如下图所示

![gitlab-ci](http://upload-images.jianshu.io/upload_images/525728-4339103186d2b1c9.png?imageMogr2/auto-orient/strip%7CimageView2/2)

Runner 可以分布在不同的主机上，同一个主机上可以根据不同的项目注册多个 Runner。

## Gitlab-Runner安装
直接参考[官网教程](https://docs.gitlab.com/runner/install/)安装

```bash
# For MacOS 
brew install gitlab-ci-multi-runner

# For Debian/Ubuntu 
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-ci-multi-runner

# For RHEL/CentOS 
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.rpm.sh | sudo bash
sudo yum install gitlab-ci-multi-runner
```

## 使用gitlab-ci-multi-runner注册Runner
安装好gitlab-ci-multi-runner这个软件之后，我们就可以用它向GitLab-CI注册Runner了，Gitlab-Runner可以分为两种类型:

* Shared Runner: 共享型，这个需要gitlab管理员创建。
* Specific Runner: 指定型，拥有项目工程访问权限的人都可以创建(本文主要创建该类型Runner)

向GitLab-CI注册一个Runner需要两样东西：GitLab-CI的url和注册token。其中，token是为了确定你这个Runner是所有工程都能够使用的Shared Runner还是具体某一个工程才能使用的Specific Runner。

首先，在gitlab项目配置上选中 Runners, 然后会出现相应的ci绑定Runner的配置选项
![gitlab-runner](http://wx2.sinaimg.cn/mw1024/78d85414ly1fkwt8499g7j21kw0u5gwc.jpg)
根据项目设置中的 url 和 token , 在服务器上注册Runner，

```bash
#然后启动Runner去和CI进行绑定
$ gitlab-ci-multi-runner register
Running in System-mode
Please enter the gitlab-ci coordinator Url:
#-->然后让你输入上图的CI URL
Please enter the gitlab-ci token for thus runner:
#-->然后让你输入上图的Token
Please enter the gitlab-ci description for this runner:
#-->然后随便给Runner命名
Please enter the gitlab-ci tags for thus runner(comma separated):
Please enter the executor:
#-->然后类型的话， 请务必选 Shell
#-->完毕

$ gitlab-ci-multi-runner list   // 查看是否注册成功
$ gitlab-ci-multi-runner start  // 把Runner当成Service启动
```
如果注册的Runner和Gitlab连接上则会出现绿色Runner证明可用
![gitlab-runner](http://wx2.sinaimg.cn/mw1024/78d85414ly1fkwubaypawj21kw0iodm5.jpg)

## Runner使用
在git项目根目录下创建文件 .gitlab-ci.yml 脚本

```bash
build:
	script: "pwd & mvn test"
```
在 Pipelines 中运行，即会自动运行脚本进行构建并输出构建结果，之后每次提交代码都会进行集成测试，保证每次的提交都是正确的
![gitlab-ci](http://wx2.sinaimg.cn/mw1024/78d85414ly1fkwujg6ti7j21kw0l00z6.jpg)


## 参考资料
1. [Gitlab-Runner](网站https://docs.gitlab.com/runner/install/)
2. [GitLab CI持续集成配置方案](http://www.cnblogs.com/newP/p/5735366.html)


