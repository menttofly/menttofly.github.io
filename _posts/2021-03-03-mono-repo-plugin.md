---
title: mono-repo插件
date: 2021-03-03 21:14:15
categories: [组件化, 容器化]
tags: [cocoapods] 
---

## 背景介绍

`monorepo` 作为组件化架构中的一种源码组织方案，不仅可以提高团队协同效率，统一发布、测试工作流，同时也保留了模块间相对的隔离性。由于`CocoaPods` 并未提供官方的 `monorepo` 支持，因此工程早期通过在 `Podfile` 中使用 `:path` 语法来声明依赖关系：

```bash
pod 'ModuleA', :path => '../modules/ModuleA'
...
```

但是这种方式不仅低效，同时也无法复用组件自身依赖项，原因如下所述：

- `podspec` 不能通过 `:path` 选项指定本地组件，缺失解决自身依赖问题能力
- `Podfile` 需穷举所有依赖，丢失会导致 `pod install` 报错，引入无关组件则造成冗余
- 当组件路径发生变化时，需要调整全部声明依赖项的位置

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

本插件发布在 `Ruby Gems` 上，直接安装即可：

```bash
➜ gem install cocoapods-monorepo
```

在 `Podfile` 中引入插件并指定读取目录 `:path`，然后执行 `pod install` ：

```ruby
plugin 'cocoapods-monorepo', :path => 'path/to/monorepo-directory'
```

一切就绪！

## 技术原理

### 1. cocoapods plugin

`CocoaPods` 允许开发者使用和创建自己的插件，因为 `Ruby` 本身具有非常强的动态特性。插件开发类似 `objc category` 和 `swift extension` ，可以很方便地对 `class/module` 进行拓展，并且为其添加甚至重写方法、属性等。

