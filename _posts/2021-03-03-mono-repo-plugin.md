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
- 以 `RubyGems` 的形式发布，对于团队成员和 `CI` 环境接入几乎零成本

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

### 1. 插件机制

由于 `CocoaPods` 是一个由少数人员维护的社区项目，无法完全支持众多潜在有用的 `Xcode` 功能。所以通过增加插件体系架构，允许其他人拓展 `CocoaPods` 以支持社区主要发展目标之外的其它特性。至于插件能做什么，官方文档是这么描述的：

> **What can CocoaPods Plugins do?**
>
> - Add new commands to `pod`
> - Hook into the install process, both before and after
> - Do whatever they want, because Ruby is a very dynamic language

简单来说 `Ruby` 具备非常强的动态特性，不仅支持对现有的 `class & module` 进行扩展，甚至可以添加或重写方法和属性。举个例子，我们使用 `alias_method` 实现 `objc` 中常见的 `Mehod Swizzling` 效果，在调用 `find_cached_set` 前执行其它任务：

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

为了提高插件研发效率，我们需要分析 `pod install` 执行过程，进行一些必要调试工作。选择 `Bundler+VSCode` 研发工具链，然后安装调试 `Ruby` 时所需的环境依赖，同时 `VSCode` 中也要安装 `Ruby` 插件：

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

值得一提的是，插件和 `CocoaPods` 是支持同时调试的，我们可以验证插件行为是否符合预期。

### 3. 实现插件

理论上，我们的 `monorepo` 插件应实现以下核心功能：

- 支持设定组件读取目录，这是实用性的前提
- 自动为 `Podfile` 、 `podspec` 的本地组件添加 `:path` 选项

#### 3.1 自动处理Podfile本地组件

如何自动给 `Podfile` 的组件添加 `:path` 参数呢？我们不妨先了解依赖项解析后的最终产物，也就是 [Pod::Dependency](https://github.com/CocoaPods/Core/blob/master/lib/cocoapods-core/dependency.rb) ：

> The Dependency allows to specify dependencies of a `Podfile` or a `podspec` on a Pod. It stores the name of the dependency, version requirements and external sources information

有别于 `Pod::Source` 使用 `Git Repo` 托管所有 `podspec` 的方式， `external source` 通过特定 `podsepc` 文件去下载 `Pod` 依赖。目前为止，`Dependency` 支持以下类型外部源：

```ruby
Dependency.new('libPusher', {:git     => 'example.com/repo.git'})
Dependency.new('libPusher', {:path    => 'path/to/folder'})
Dependency.new('libPusher', {:podspec => 'example.com/libPusher.podspec'})
```

我们已经知道，组件含有 `:path` 参数会解析成 `Development Pods` ，本质是给 `external_source` 变量添加 `:path` 键值。而 [Pod::Resolver](https://github.com/CocoaPods/CocoaPods/blob/master/lib/cocoapods/resolver.rb) 在根据 `Target` 生成依赖列表时，使用 [PodfileDependencyCache](https://github.com/CocoaPods/CocoaPods/blob/master/lib/cocoapods/installer/analyzer/podfile_dependency_cache.rb) 作为 `Podfile` 依赖来源。所以，尝试在读取 `Podfile` 时给依赖指定外部源，验证能否实现使用 `:path` 选项的效果：

```ruby
def self.from_podfile(podfile)
  ...
  podfile.target_definition_list.each do |target_definition|
    deps = target_definition.dependencies
    deps.each do |dependency| 
      dependency.external_source = {}
      dependency.external_source[:path] = 'path/to/some/Module'
    end
    podfile_dependencies.concat deps
    dependencies_by_target_definition[target_definition] = deps
  end
  ...
end
```

执行 `pod install` 后和预期一样，即使没有设定 `:path` 参数，组件依然出现在 `Development Pods` 中。

#### 3.2 自动解析podspec本地组件

对于 `podspec` 声明的其它组件，要在依赖分析时提取 `podspec` 信息。我们发现 `pod install` 执行 `analyze` 过程中，会调用 `fetch_external_source` 方法将 `external source` 保存到 [sandbox](https://github.com/CocoaPods/CocoaPods/blob/master/lib/cocoapods/sandbox.rb) 中。然后在后续安装依赖时，根据 `sandbox` 查询本地组件：

```ruby
def fetch_external_source(dependency, use_lockfile_options)
  source = if use_lockfile_options && lockfile && checkout_options = lockfile.checkout_options_for_pod_named(dependency.root_name)
             ExternalSources.from_params(checkout_options, dependency, podfile.defined_in_file, installation_options.clean?)
           else
             ExternalSources.from_dependency(dependency, podfile.defined_in_file, installation_options.clean?)
           end
  source.fetch(sandbox)
end
```

因此，在适当的位置把 `Specification` 注入到 `sandbox` 中，才能影响 `Pod` 依赖安装结果。`Pod::Resolver` 执行依赖分析时，对于每个 `dependency` 相应 `Specification` ，都会通过 `find_cached_set` 返回满足 `requirements` 的结果集：

```ruby
def specifications_for_dependency(dependency, additional_requirements = [])
  requirement_list = dependency.requirement.as_list + additional_requirements.flat_map(&:as_list)
  requirement_list.uniq!
  requirement = Requirement.new(requirement_list)
  find_cached_set(dependency).
    all_specifications(warn_for_multiple_pod_sources, requirement).
    map { |s| s.subspec_by_name(dependency.name, false, true) }.compact
end
```

所以，`find_cached_set` 是一个对目标 `dependency` 添加 `external source` 的绝佳位置，同时需要兼容 `CocoaPods` 版本升级：

```ruby
alias_method :origin_find_cached_set, :find_cached_set
def find_cached_set(dependency)
  unless dependency.external_source
    name = dependency.root_name
    podspec_path = podspec_local_cache.local_podspecs[name]
    unless podspec_path.nil?
      dependency.external_source[:path] = podspec_path
      stored_to_sandbox_podspecs(name, dependency)
    end
  end
  origin_find_cached_set(dependency)
end
```

现在执行 `pod install` 或 `pod update` ，会发现所有本地组件都出现在 `Development Pods` 中。

#### 3.3 支持目录读取

从 `Podfile` 的 [DSL](https://github.com/CocoaPods/Core/blob/master/lib/cocoapods-core/podfile/dsl.rb) 源码可知，插件被调用时支持传递参数，例如：

```ruby
plugin 'cocoapods-keys', :keyring => 'Eidolon'
```

我们只需在插件 `pre_install` 注册时接收参数内容：

```ruby
Pod::HooksManager.register("cocoapods-monorepo", :pre_install) do |context, options|
  unless options.key?(:path)
    raise Pod::Informative, "require pass `:path` option"
  end
  Pod::Resolver.monorepo = options[:path] 
end
```

## 总结

在组件化架构向 `monorepo` 方案演进过程中， 我们发现 `CocoaPods` 本身无法很好地满足要求。但幸运的是，插件机制为支持 `monorepo` 特性提供了可能。最终在插件正式交付之后，很好地支撑了公司多条产品线运行，给我们带来十分可观的收益。

## 参考文档

- [How to use CocoaPods plugins](https://guides.cocoapods.org/plugins/setting-up-plugins.html) 
- [CocoaPods都做了什么](https://draveness.me/cocoapods/) 
- [CocoaPods历险 - 总览](https://www.desgard.com/ios/ruby/2019/03/15/cocoapods-1.html) 

