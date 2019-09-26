---
layout:     post
title:      PivotalCloudFoundry(一)
subtitle:   Deploy App To CloudFoundry
date:       2019-03-06
author:     LANY
catalog: true
tags:
    - Pivotal
    - PAAS
    - Cloud Foundry
---

# PivotalCloudFoundry-Deploy APP

## 先决条件


* [安装Cloud Foundry Command Line Interface](https://docs.pivotal.io/pivotalcf/2-4/cf-cli/install-go-cli.html)


## 部署过程

* 上传并存储应用程序文件
* 检查并存储应用程序元数据
* 为应用程序创建"droplet"(CloudFoundry执行单元)
* PCF选择合适的`Diege Cell`运行所创建的"droplet"
* 开始运行程序

## 通过CLI登陆到PCF

在push你的app到Cloud Foundry之前你需要知道：

* Cloud Foundry实例的API端点
* Cloud Foundry实例的账户名和密码
* 你想将你的应用程序部署到哪个Organization和Space。

## 决定部署选项

在你部署之前，你需要决定以下参数的选项：

* `Name`：你可以使用任意带有字母的字符作为你的应用程序的名称
* `Instances`: 你需要为你的运用运行的实例数量。通常来说，运行的实例数越多，那么宕机的时间越少。
* `Memory Limit`: 应用程序的每个实例可以消耗的最大内存量。如果实例超过此限制，Cloud Foundry将重新启动该实例。
* `Start Command`: Cloud Foundry用来启动应用程序的每个实例的命令。
* `Subdomain(host) and Domain`: 路由是子域和域的组合，必须是全局惟一的。无论是指定路由的一部分，还是允许Cloud Foundry使用默认值，这都是正确的。
* `Services`: 应用程序可以绑定到数据库、消息传递和键值存储等服务。应用程序被部署到应用程序空间中。应用程序只能绑定到目标应用程序空间中具有现有实例的服务。

## 定义部署的选项

你可以在命令行、清单文件或者两者之上同时定义部署的选项。

Cloud Foundry上传了除版本控制文件和文件夹之外的所有应用程序文件，这些文件的名称包括`.svn`、`.git`和`_darcs`。要从上传中排除其他文件，请在运行push命令的目录中的.cfignore文件中指定它们。

更多的信息请参考[在PUSH时忽略不必要的文件](https://docs.pivotal.io/pivotalcf/2-4/devguide/deploy-apps/prepare-to-deploy.html#exclude)

## 配置Pre-Runtime Hooks

要配置运行前钩子，需要在应用程序的根目录下创建名称为`.profile`的文件。如果目录下已经包含了`.profile`脚本，那么Cloud Foundry在开始应用程序之前会立即运行该脚本。因为`.profile`脚本在构建包之后执行，所以脚本可以访问由构建包创建的语言运行时环境。

你可以使用`.profile`脚本执行特定于应用程序的初始化任务，例如设置自定义环境变量。环境变量是在操作系统级别定义的键-值对。这些键值对提供了一种配置系统上运行的应用程序的方法。例如，任何应用程序都可以访问LANG环境变量，以确定错误消息和指令、排序序列和日期格式使用哪种语言。

`.profile`示例:

```
# Set the default LANG for your apps
export LANG=en_US.UTF-8
```

## PUSH应用程序

在不使用`manifest`文件的情况下运行下面的命令来PUSH应用程序：

```
cf push APP-NAME
```

如果你在`manifest`文件中提供了应用程序名称，你可以减少`CF PUSH`的参数。

详细文章 : [通过manifests部署应用程序](https://docs.pivotal.io/pivotalcf/2-4/devguide/deploy-apps/manifest.html)

