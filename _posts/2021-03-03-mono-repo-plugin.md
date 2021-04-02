---
title: 组件化插件cocoapods-monorepo
date: 2021-03-03 21:14:15
categories: [组件化]
tags: [cocoapods] 
---

## 背景介绍 

`monorepo` 作为组件化架构中的一种源码组织方案，不仅可以提高团队协同效率，统一发布、测试工作流，同时也能保留组件间的相对隔离。由于`CocoaPods` 并未提供官方的 `monorepo` 支持，因此工程早期通过在 `Podfile` 中使用 `:path` 语法来声明组件依赖：

```bash
pod 'ModuleA', :path => '../modules/ModuleA'
...
```

但是这种方式不仅低效，同时也无法复用组件自身依赖关系，原因如下：

- `podspec` 不能通过 `:path` 选项指定本地组件，缺失解析自身依赖的能力
- `Podfile` 需穷举所有依赖，依赖丢失时 `pod` 报错，引入无关组件则造成冗余
- 当组件路径发生变化时，需要调整全部声明依赖项的位置

如果可以让 `podspec` 支持解析本地组件，所有问题就能迎刃而解。幸运的是，`CocoaPods` 提供了完整的插件机制。通过我们研发的 [cocoapods-monorepo](https://github.com/menttofly/cocoapods-monorepo) 插件，实现 `CocoaPods` 对 `monorepo` 特性的支持，解决了上述工程化问题。

## cocoapods-monorepo插件 

### 1. 核心功能

本插件可自动识别 `Podfile` 与 `podspec` 本地依赖，无需声明组件所在路径，并具备以下特点：

- 面向 `AOP` 编程，不侵入 `CocoaPods` 执行流程，不破坏 `Xcode` 增量编译能力
- 以 `ruby gem` 的形式发布，对于团队成员和 `CI` 环境接入几乎零成本

### 2. 主要构成

|           文件            |                             功能                             |
| :-----------------------: | :----------------------------------------------------------: |
|   `cocoapods_plugin.rb`   | 注册 `CocoaPods` 钩子，在 `install` 前导入 `resolver` ，并解析路径参数 |
| `pod_spec_local_cache.rb` |    缓存特定目录下所有本地组件 `podspec` 解析后的相关信息     |
|       `resolver.rb`       | 核心类，为本地组件指定外部源后注入 `sandbox` ，成为 `Development Pods` |

### 3. 接入方式

本插件发布在 [RubyGems](https://rubygems.org/gems/cocoapods-monorepo) 上，直接使用 `gem` 命令安装即可：

```bash
➜ gem install cocoapods-monorepo
```

在 `Podfile` 中引用插件并通过 `:path` 选项设定读取目录，然后执行 `pod install` ：

```ruby
plugin 'cocoapods-monorepo', :path => 'path/to/modules-directory'
```

## 技术原理

### 1. CocoaPods插件

由于 `CocoaPods` 是一个由少数人员维护的社区项目，无法完全支持众多潜在有用的 `Xcode` 功能。所以通过增加插件体系架构，允许其他人拓展 `CocoaPods` 以支持社区主要发展目标之外的其它功能。另外，官方对插件是这么描述的：

> **What can CocoaPods Plugins do?**
>
> - Add new commands to `pod`
> - Hook into the install process, both before and after
> - Do whatever they want, because Ruby is a very dynamic language

简单来说 `Ruby` 具备非常强的动态特性，支持对现有的 `class & module` 进行扩展，甚至可以添加或重写方法和属性。举个例子，我们使用 `alias_method` 实现 `objc` 中常见的 `Mehod Swizzling` 效果，在调用 `find_cached_set` 前执行其它任务：

```ruby
class Resolver
    alias_method :origin_find_cached_set, :find_cached_set
    def find_cached_set(dependency)
      # Do anything before original method
      origin_find_cached_set(dependency)
    end
end
```

### 2. 搭建调试环境

为了提高插件研发效率，我们需要分析 `pod install` 执行过程，进行一些必要调试工作。我选择了 `Bundler+VSCode` 工具链，接着安装调试 `Ruby` 时所需的环境依赖，同时 `VSCode` 中也要安装 `Ruby` 插件：

```bash
➜ gem install ruby-debug-ide
➜ gem install debase
```

将项目工程、插件及 `CocoaPods` 源码放入相同的目录中，同时新建一个 `Gemfile` 文件，然后运行 `bundle install` 命令：

```ruby
gem 'cocoapods', path: 'path/to/cocoapods'
gem 'cocoapods-monorepo', path: 'path/to/cocoapods-monorepo'
gem 'ruby-debug-ide'
gem 'debase'
```

在根目录下创建 `.vscode/launch.json` ，`args` 可以选择 `install` 或 `update` 等选项：

```json
{
  "configurations": [{
      "name": "Debug CocoaPods Plugin with Bundler",
      "showDebuggerOutput": true,
      "type": "Ruby",
      "request": "launch",
      "useBundler": true,
      "cwd": "${workspaceRoot}/path/to/Podfile",	// Podfile所在路径
      "program": "${workspaceRoot}/CocoaPods/bin/pod",
      "args": ["install"]
    }]
}
```

此外，插件和 `CocoaPods` 是支持同时调试的，我们可以验证插件行为是否符合预期。

### 3. 插件开发

理论上，我们的 `monorepo` 插件应实现以下核心功能：

- 支持设定组件读取目录，这是实用性的前提
- `Podfile` 声明的本地组件，`:path` 缺失可被识别为 `Development Pods` 
- `podspec` 依赖的本地组件，同样也能够解析为 `Development Pods` 

#### 3.1 解析Podfile声明的本地组件

从 `Pod::Dependency` 中可以知道，存在一种叫 `external_source` 的外部源，使用下面语法均会被认定为外部源：

```ruby
pod 'YYKit', :path => '../../YYKit'
pod 'YYKit', :podspec => '../../YYKit/YYKit.podspec'
pod 'YYKit', :git => 'https://github.com/ibireme/YYKit.git'
```

## 总结

收益

## 参考文档

- [CocoaPods历险 - 总览](https://www.desgard.com/ios/ruby/2019/03/15/cocoapods-1.html) 
- [CocoaPods都做了什么](https://draveness.me/cocoapods/) 

