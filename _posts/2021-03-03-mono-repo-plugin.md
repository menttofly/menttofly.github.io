---
title: mono-repo插件
date: 2021-03-03 21:14:15
categories: [容器化]
tags: [cocoapods] 
---

## 背景介绍

为提高团队沟通协同效率，减少版本更新维护成本，我们将项目组件化方案调整成基于 `cocoapods` 的 `monorepo` 的方案。由于`cocoapods` 并未提供天然的 `monorepo` 支持，因此早期是通过在 `Podfile` 中使用 `:path` 语法来解决依赖问题：

```bash
pod 'GDWind', :path => '../../modules/GDWind'
```

但是这种方式不仅低效，对新环境的搭建也非常不友好，在组件的规模比较大时很难管理，原因如下：

- `podspec` 不支持 `:path` 语法，无法使用 `cocoapods` 自身解决依赖项的能力
- 需要在 `Podfile` 穷举所有依赖，容易引入无关组件，且缺失依赖会导致各种错误

所以我们结合 `cocoapods` 提供的插件能力，开发了 `cocoapods-monorepo` 插件，解决上述相关问题。

## cocoapods-monorepo插件 

### 1. 核心功能

本插件提供 `Podfile` 和 `podspec` 本地依赖识别能力，无需指明组件所在路径，特点如下：

- 基于AOP编程，不侵入 `cocoapods` 执行逻辑，不破坏 `xcode` 增量编译能力
- 以 `ruby gem` 包形式发布，团队成功和CI流程集成几乎零成本

### 2. 主要构成

|           文件            |                             功能                             |
| :-----------------------: | :----------------------------------------------------------: |
|   `cocoapods_plugin.rb`   | 注册 `cocoapods` 钩子，在 `install` 和 `update` 前导入 `resolver` 模块 |
| `pod spec_local_cache.rb` |    缓存指定目录下所有本地组件 `podspec` 解析后的描述信息     |
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

`cocoapods` 中的插件开发

