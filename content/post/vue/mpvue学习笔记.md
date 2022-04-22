---
typora-root-url: ../../../static
title: "mpvue学习笔记"
date: 2020-07-27T19:51:36+08:00
draft: false
categories: ["Vue"]
tags: ["mpvue"]
---

## 介绍
mpvue是美团开源的一个用于开发小程序的框架，在用法上和vue高度一致。

## 环境
- node.js：运行在服务端的 JavaScript。
- vue-cli：一个vue专用的项目脚手架工具。
- 微信开发者工具。

## mpvue项目结构
- package.json文件：项目的主配置文件，包含了mpvue项目的基本描述信息，项目所依赖的各种第三方库及版本信息以及可执行的脚本信息。例如 `script` 部分就有 `dev` , `start` , `build` , `lint` 等可执行脚本命令。
- project.config.json文件：用于管理微信开发者工具的小程序的配置文件，主要记录了小程序的appid、代码主目录以及编译选项等信息。
- static目录：存放小程序本地静态资源，如图片、文本文件等，可以使用相对路径或绝对路径进行访问。
- build目录：存放一些用于编译打包的node.js脚本和webpack配置文件，一般无需改动这些文件。
- config目录：包含了用于开发和生产环境下的不同配置，dev.env.js用于开发环境，prod.env.js用于生产环境。可以编写不一样的信息，然后在代码中以变量的形式进行引用。
- src目录：是进行小程序功能编写的地方。利用脚手架生成的基于mpvue的demo代码为我们创建了components、pages和utils目录和App.vue、main.js和app.json三个文件。main.js+App.vue是两个入口文件，相当于原生小程序框架中的app.json和app.js的复合体。

## 注意事项
- 安装完node.js后，最好将npm的源更改为国内的淘宝镜像源： `npm set registry https://registry.npm.taobao.org/` 。
- 创建mpvue脚手架项目： `vue init mpvue/mpvue-quickstart firstapp` 后需要进入项目运行 `npm install` 来安装依赖库，此时会发现多了node_modules目录。
- `npm run dev` 启动开发模式，将自动将src下的代码编译为微信小程序代码从而运行，编译后的代码在dis目录下。
- 新增的页面需要重新 `npm run dev` 来进行编译。