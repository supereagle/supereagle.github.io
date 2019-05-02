---
layout:     post
title:      "Jenkins Plugin开发指南"
subtitle:   "Jenkins Plugin Development Guide"
date:       2017-04-27
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Jenkins
---

## 环境准备
所有Jenkins plugin都是java开发的Maven项目，需要准备如下环境：
- 安装工具: [Maven](http://maven.apache.org/); JDK 6.0或以上版本，[Eclipse](http://www.eclipse.org/eclipse4/)
- 环境变量：PATH中添加mvn.bat路径；JAVA_HOME中添加SDK路径

## 导入项目
Jenkins plugin项目一般都是开源的，可以在他们的基础上进行定制化。
1. 从github上git clone项目
2. import到Eclipse
   - 打开Eclipse，选择“File”->“import”->“Maven”->“Existing Maven Projects”，选择需要导入的项目，忽略import过程中的warnings。
   - 在进行任何操作前，先`Run As` -> `Maven clean`，才能保证开发的plugin在Jenkins中可以被看见。

## 测试
在命令行窗口中plugin项目的目录下执行mvn hpi:run，Maven会在Jetty中启动一个测试Jenkins，并将新开发的plugin部署到Jenkins中。
在浏览器地址栏中输入http://localhost:8080/jenkins即可进入测试Jenkins。
在测试过程中，plugin并没有被打包成hpi文件，而是被打包成hpl文件。hpl文件是一个简单的文本文件，描述plugin相关的所有文件，而不是一个真正的package文件。

## 构建
Jenkins的所有plugin都是通过hpi的文件格式发布的，所以在发布前先要自己开发的plugin打包成hpi文件。
在命令行窗口中plugin项目的目录下执行mvn package，即可自动在target目录下的生成该plugin的hpi文件和jar文件。

## Reference
- [Jenkins插件开发入门 资料收集](http://plkong.iteye.com/blog/1780121)
- [Using Hudson's Plugin Development Framework to Build Your First Hudson Plug-in](http://wiki.corp.vipshop.com/download/attachments/52299596/Writing-first-hudson-plugin.pdf?version=1&modificationDate=1441190202000&api=v2)