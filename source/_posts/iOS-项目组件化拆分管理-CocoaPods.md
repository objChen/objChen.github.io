---
title: iOS 项目组件化拆分管理 - CocoaPods
date: 2018-07-07 00:07:27
tags: Refactoring
---



### 背景

在执行`pod update`之前，可以执行`pod outdated`来查看有哪些pods有新的版本。

当你执行`pod update PODNAME`时，CocoaPods会尝试发现`PODNAME`的可更新的版本，而不会去关注`Podfile.lock`中的版本。它会把依赖更新到最新的版本。

### 使用

- CocoaPods安装及使用 

  为了控制篇幅，简短精炼，CocoaPods的简介、安装、使用及常见错误不在本篇文章赘述，有需要请移步[CocoaPods的使用](objchen.cn)

- 新建podspec

  你的项目已经建好了：在项目根目录执行 `pod spec create '项目名'`

  你还没有建项目：执行 `pod lib create '项目名'`

   需要填写信息，如下：

  ```
  // 使用的平台
  What platform do you want to use?? [ iOS / macOS ] 
   > iOS
  // 使用的语言
  What language do you want to use?? [ Swift / ObjC ] 
   > ObjC
  // 是否在你的目录中新建一个Demo
  Would you like to include a demo application with your library? [ Yes / No ]   
   > Yes
  // 是否需要测试框架
  Which testing frameworks will you use? [ Specta / Kiwi / None ] 
   > None
  // 是否需要UI测试
  Would you like to do view based testing? [ Yes / No ] 
   > No
  // 你的类前缀
  What is your class prefix? 
   > SK

  ```


- 添加到远程Git仓库： `git remote add origin 远程仓库地址`

- xxx.podspec 配置

  ```ruby
  Pod::Spec.new do |s|  
    s.name             = 'SooKit'  # 名称，pod search搜索的关键词
    s.version          = '0.1.0'  # 版本号
    s.summary          = 'A short description of SooKit.' # 简介，pod search搜索的关键词
    s.description      = 'SooKitSooKitSooKitSooKitSooKit'  # 详细说明，这个根据库的功能来描述
    s.homepage         = 'https://github.com/“XXX”/SooKit'  # 主页地址，例如Github地址
    s.license          = { :type => 'MIT', :file => 'LICENSE' }  # 许可证
    s.author           = { '“XXX”' => '“peng_chen@soolife.com.cn”' }  # 作者
    s.source           = { :git => 'https://github.com/“XXX”/SooKit.git', :tag => s.version.to_s }  # Git仓库地址
    s.social_media_url = 'https://twitter.com/<XXX>'  # 社交网址
    s.ios.deployment_target = '8.0'  # 兼容版本
    s.source_files     = 'SooKit/Classes/**/*' # 需要包含的源文件，常见的写法如下
    s.dependency     "BPushSDK", "1.4.1"  # 依赖库，不能依赖未发布的库
    s.dependency     'AFNetworking', '~> 2.3'  # 依赖库，如有多个可以这样写
    s.requires_arc     = true  # 是否要求ARC
    s.public_header_files = 'Pod/Classes/**/*.h'  # pch 文件
    s.frameworks       = 'UIKit', 'MapKit'   # 需要的framework
    s.resources        = "XXX/Images/*.png"  # 需要包含的图片等资源文件
    s.resource_bundles = {    # 需要包含的图片等资源文件
      'SooKit' => ['SooKit/Assets/*.png']
    }
  end
  ```

  - s.source_files常见写法

  ```
  "Directory1/*"
  "Directory1/Directory2/*.{h,m}"
  "Directory1/**/*.h"
  ```

  1. "*"表示匹配所有文件*
  2. "*.{h，m}"表示匹配所有以.h和.m结尾的文件
  3. "**"表示匹配所有子目录

  ​

  - s.source常见写法

  ```
  s.source = { :git => "https://github.com/xxx/xxx.git", :commit => "68defea" }
  s.source = { :git => "https://github.com/xxx/xxx.git", :tag => 1.0.0 }
  s.source = { :git => "https://github.com/xxx/xxx.git", :tag => s.version }
  ```

  1. commit =>“68defea”表示将这个Pod版本与Git仓库中的某个commit绑定
  2. tag => 1.0.0表示将这个Pod版本与Git仓库中的某个版本的commit绑定
  3. tag => s.version表示将这个Pod版本与Git仓库中相同版本的commit绑定

- 检查.podspec 执行 `pod lib lint 项目名.podspec`

- ​

- ​

**// TODO：未完，忙完再续。**

### 规则

基础库 - SooKit (Private Pods)    中类名前缀为SooKit首字母缩写SK

### 最后

