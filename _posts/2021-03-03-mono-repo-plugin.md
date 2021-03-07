---
title: mono-repo插件
date: 2021-03-03 21:14:15
categories: [容器化]
tags: [cocoapods] 
---

## 背景介绍

在组件化架构非常流行的今天，如何有效地隔离拆分子模块，并提高协同发布效率成为了重中之重。`multi-repo` 在版本管理、统一工作流等方面的不足，促使我们决定将组建化架构调整为基于 `cocoapods` 的 `monorepo` 方案。由于`cocoapods` 并未提供天然的 `monorepo` 支持，因此早期是通过在 `Podfile` 中使用 `:path` 语法来解决依赖问题：

```bash
pod 'GDWind', :path => '../../modules/GDWind'
```

但是这种方式不仅低效，对新环境的搭建也非常不友好，在组件的规模比较大时很难管理，原因如下：

- `podspec` 不支持 `:path` 语法，无法使用 `cocoapods` 自身解决依赖项的能力
- 需要在 `Podfile` 穷举所有依赖，容易引入无关组件，且缺失依赖会导致各种错误

所以结合 `cocoapods` 提供的插件能力，开发了 `cocoapods-monorepo` 插件，用于解决上述相关问题。

## cocoapods-monorepo插件 

### 1. 核心功能

本插件提供 `Podfile` 和 `podspec` 本地依赖识别能力，无需指明组件所在路径，特点如下：

- 基于AOP编程，不侵入 `cocoapods` 执行逻辑，不破坏 `xcode` 增量编译能力
- 以 `ruby gem` 包形式发布，团队成功和CI流程集成几乎零成本

### 2. 主要构成

|           文件            |                             功能                             |
| :-----------------------: | :----------------------------------------------------------: |
|   `cocoapods_plugin.rb`   | 注册 `cocoapods` 钩子，在 `install` 和 `update` 前导入 `resolver` 模块 |
| `pod_spec_local_cache.rb` |    缓存指定目录下所有本地组件 `podspec` 解析后的描述信息     |
|       `resolver.rb`       |  核心类，将组件指定为外部源并注入 `sandbox` ，成为本地依赖   |

### 3. 接入方式

先将工作目录定位到插件所在位置，然后执行命令安装插件：

```bash
➜ work-directory/tools/cocoapods-plugins
➜ gem install cocoapods-monorepo-0.0.1.gem --user-install
```

在 `Podfile` 中添加 `plugin 'cocoapods-monorepo'`，并执行 `pod install` ：

```ruby
source 'https://github.com/CocoaPods/Specs.git'
platform :ios, '10.0'
plugin 'cocoapods-monorepo'
```

一切就绪！

## 技术原理

### 1. cocoapods plugin

`CocoaPods` 允许开发者使用和创建自己的插件，其核心原理为基于AOP编程能力。插件开发类似 `objc category` 和 `swift extension` ，可以很方便地对 `class/module` 进行拓展，并且为其添加甚至重写方法、属性等。

