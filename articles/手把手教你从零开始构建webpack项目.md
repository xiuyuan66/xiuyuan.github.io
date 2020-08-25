![webpack](https://user-gold-cdn.xitu.io/2019/11/26/16ea7919f9df0ed4?imageView2/1/w/1304/h/734/q/85/format/webp/interlace/1)

# 手把手教你从零开始构建webpack项目

## 前言

对于入门选手来讲，webpack 配置项很多很重，如何快速配置一个可用于线上环境的 webpack 就是一件值得思考的事情。其实熟悉 webpack 之后会发现很简单，基础的配置可以分为以下几个方面： entry 、 output 、 mode 、 resolve 、 module 、 optimization 、 plugin 、 source map 、 performance 等，本文就来重点分析下这些部分。

## 1.什么是webpack?

本质上，`webpack` 是一个现代 `JavaScript` 应用程序的静态模块打包器(`module bundler`)。当 `webpack` 处理应用程序时，它会递归地构建一个依赖关系图(`dependency graph`)，其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个 `bundle`。

## 2.核心概念

在开始前你需要先理解`webpack`几个核心概念：

- `Entry` // 入口
- `Output` //输出
- `Loaders` //模块转换器，loader 可以将所有类型的文件转换为 webpack 能够处理的有效模块
- `Plugins` //扩展插件，从打包优化和压缩，一直到重新定义环境中的变量，在webpack构建流程中的特定时机注入扩展逻辑来改变构建结果或做你想要做的事情
- `Mode` //模式，通过选择 development 或 production 之中的一个，来设置 mode 参数，你可以启用相应模式下的 webpack 内置的优化

## 3.新建项目

在本地新建文件夹
``` 
  mkdir webpack-demo && cd webpack-demo
  npm init -y
  npm install webpack webpack-cli --save-dev
```

