---
title: Flutter工程化能力建设
date: 2022-03-15 15:31:15
categories: [工程化]
tags: [flutter] 
---

## 前言

本文主要介绍 Flutter 工程化能力相关建设与实践，包括 DevOps 流程规范、构建产物优化等设计细节。

## DevOps流程

### 1. 混合开发

采用 Flutter 和原生混合开发模式，将 Flutter 相关依赖打包发布到私有远程库，然后以 Framework 的形式集成到原生工程，Native 开发者无需了解或配置 Flutter 环境，如图所示：
![开发架构](/assets/img/flutter_infra.png){:height="65%" width="65%"}
原生端由于 `FlutterBridge` 被跨端复用，所以 `podspec` 中不能写死 Flutter 远程依赖。对于 `Flutter SDK`、`FlutterBoost` 等必要的二进制依赖，通过 `pod_target_xcconfig` 配置引入：
![xcconfig](/assets/img/flutter_xcconfig.png)
实际开发过程中发现，Xcode 进行编译时偶现抛出 `'flutter_boost/FBFlutterViewContainer.h' file not found` 之类报错：
![error](/assets/img/flutter_not_found.png)
Xcode 编译时根据 `Target Dependency` 等因素决定构建顺序，未显示声明 `dependencies` 会影响构建行为，Task 执行的不确定性就可能引起上述错误。

> CocoaPods `test_spec` 单测时遇到类似问题，ld 链接阶段报错：`Undefined symbols for architecture x86_64`
> ![ld_error](/assets/img/flutter_ld_error.png)
> 原因和上面相同，组件单测 Target 以本地组件为根入口，缺失 `dependencies` 会导致依赖组件未参与编译，静态链接处理符号解析与重定位时，未找到被引用符号外部定义而报错。

针对本地开发测试场景，可以修改 `FlutterBridge.podspec` 显示声明依赖，执行 `pod install` 后重新编译即可解决：

```ruby
# 添加 Flutter 远程库对应的版本依赖
s.dependency 'fc_flutter|design_platform|video_platform', '2.3.03185638.3bfcb1b'
```
> PS. 通过 `subspecs` 拆分 Flutter 远程库，并在不同应用组件按需引用也同样可行。

### 2. 构建打包流程

涉及的核心流程与关键角色：
![core_process](/assets/img/flutter_process.png)
未来将优化 SDK 发布流程，构建原生主工程时通过 `Restful API` 触发 TeamCity 打包发布 Flutter 依赖，避免人工干预版本更新降低回归测试成本。

### 3. 流程优化

为了提⾼ CI 可拓展性、可维护性，并减少了业务⽅对 CI 依赖与感知，符合开闭原则、控制反转等设计原则，我们做了如下流程优化：

#### 3.1 pre-build-hook钩子

CI 提供了 `pre-build-hook` 钩⼦能力，满⾜业务希望定制 Flutter 构建前置流程的诉求，以若⼲应⽤场景为例：
- `flutter pub get` 之后⽣成 `.ios` 打包 `Podfile`，不⽀持 CocoaPods 私有源，通过此 hook 实现动态注⼊私有源，解决了私有库下载问题
- 近期 Flutter 跨端 UI 库项⽬，也通过 hook 执⾏⾃定义脚本，实现 Pigeon ⾃动创建桥接层接口，且根据 YAML 配置⽣成 Dart ⽂件

这⾥要注意当前⼯作⽬录为 Flutter Module，即 `/GDFlutter/{module_design | module_gaoding | module_video}`：

```bash
.
├── module_design
├── module_gaoding
│   ├── gd_flutter.podspec
│ ├── lib
│   │   └── main.dart
│   ├── pubspec.yaml
│ └── script
│       └── pre_build_hook.sh
└── module_video
```

#### 3.2 版本依赖优化

早期升级 `Flutter SDK` 版本时，需要修改 TeamCity 提供的版本选项。为了解耦 Flutter 和 CI 之间的依赖，我们让业务在 YAML 中直接指定 Flutter SDK，CI 识别解析之后 checkout 到对应的版本：

```yaml
environment:
  sdk: ">=2.1.0 <3.0.0"
flutter: 2.2.0
```

这设种计有两个好处：
- 保证了 Flutter 开发环境⼀致性，版本不同时在 `flutter pub get` 阶段就会产⽣错误
- 业务⽅可以⾃由更改 Flutter 版本，对 CI 这边完全是透明⽆感的，实现了控制流程的逆转

## 容器化

常规 CI 构建打包收集的产物包括 `Flutter SDK`，体积较大极⼤影响原⽣端集成速度，因此进⾏了 Flutter 容器化升级，将 Flutter 引擎以 podspec 的形式分离出来：

- 构建产物拆分：
```bash
 # 构建 DEBUG 产物
flutter build ios-framework --cocoapods --no-profile --no-release --output=$build 
# 构建 RELEASE 产物
flutter build ios-framework --cocoapods --no-profile --no-debug --output=$build
# 推送 Flutter-Debug.podspec 或 Flutter-Release.podspec
pod repo push gaoding-gaoding-ios-gdspecs $flutter_podspec --allow-warnings --skip-import-validation 
```

- podspec原生集成：
```ruby
Pod::Spec.new do |s|
    s.name = 'fc_flutter'
    s.version = '2.3.24161459.e20810c'
    s.source = { :git => 'git@git.xxx.com:Gaoding-iOS/xxx.git', :commit => '8339d8f4d32a6f64
    if ENV['GD_Develop'] == nil || ENV['GD_Develop'] == '1' 
        $env = 'Debug'
    else
        $env = 'Release' 
    end
    s.dependency "Flutter-#$env", '2.5.3'
```
> PS. 使⽤环境变量来区分集成 Debug 或者 Release 的 Flutter 版本

实测下来发现，Debug+Release 包⼤⼩减⼩了约 500MB+，显著提⾼了 Flutter 远程库的下载速度。

## 版本检测

Flutter 在多⼈⽇常开发时，注意到存在⼀些问题：
- 并⾏开发过程中，版本号管理混乱，经常出现 Flutter 远程库被覆盖的问题
- App 正式发布时，容易忘记升级版本号，导致没有集成最新 Flutter 功能

所以为双端 PR 增加了**「Flutter 版本号检测」**Check，并接入 Fitness 质量函数 CI 构建时触发，限制更新版本号必须与 Flutter 仓库 master 分支最新 commit sha 匹配：
![lint](/assets/img/flutter_lint.png){:height="80%" width="80%"}
> Flutter 库使⽤ 4 段式版本号，以打包分⽀最新 commit sha 为结尾

如果遇到 Check 没有触发的情况，也可以使⽤ Postman ⼯具来⼿动更新状态：

```bash
https://jack.gdm.xxx.com/{orgs}/{repo}/checks/{commit_sha}
```

jackbot ⽆需进⾏鉴权：
![lint](/assets/img/flutter_bot.png){:height="100%" width="100%"}