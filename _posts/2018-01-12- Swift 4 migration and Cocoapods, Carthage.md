---
layout: post
title: "Cocoapods,Carthage构建的工程迁移到Swift 4"
author: "wx"
---

# Cocoapods

### 先决条件
在之前XCode版本中编译通过的Swift 3工程

### 迁移方法

先不升级Pod,先把工程迁移到Swift 4，没有问题的话再把Pod迁移到Swift 4。步骤: <!--more-->

1. **更新Podfile,设置Pod的swift version 为3.2**
~~~ruby
post_install do |installer|
 installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
     config.build_settings['SWIFT_VERSION'] = '3.2'
    end
end
end
~~~

2. **转换工程到Swift 4**
设置Xcode当前swift版本为4.0。点击Xcode中的`Edit` -> `Convert` -> `Convert to current swift syntax`。然后解决掉所有issues。保证工程可以编译成功。

3. **升级Podfile**
检查用到的Pods，去他们主页查看是否已经支持Swift 4,对已经支持Swift 4和尚未支持的要在Podfile中区分

~~~ruby
swift_32 = ['RxSwift', 'RxCocoa']
swift4 = ['PalaverKit']

post_install do |installer|
  installer.pods_project.targets.each do |target|
    swift_version = nil

    if swift_32.include?(target.name)
      swift_version = '3.2'
    end

    if swift4.include?(target.name)
      swift_version = '4.0'
    end

    if swift_version
      target.build_configurations.each do |config|
        config.build_settings['SWIFT_VERSION'] = swift_version
      end
    end
  end
~~~

**参见:**
[Swift New compatibility modes](https://swift.org/blog/swift-4-0-released/)
[migration-guide-swift4-Using Carthage/CocoaPods Projects](https://swift.org/migration-guide-swift4/)
[Support multiple Swift versions on a per pod basis](https://github.com/CocoaPods/CocoaPods/issues/6791)


# Carthage

### 先决条件
在之前XCode版本中编译通过的Swift 3工程

### 迁移方法
先不升级Carthage工程,先把工程迁移到Swift 4，没有问题的话再把Pod迁移到Swift。

- **转换工程到Swift 4**

设置Xcode当前swift版本为4.0。点击Xcode中的`Edit` -> `Convert` -> `Convert to current swift syntax`。然后解决掉所有issues。保证工程可以编译成功。


- **升级Cartfile**

检查用到的Framework，去他们主页查看是否已经支持Swift 4,在Cartfile中指定对应的具体版本

~~~ruby
github "ninjaprox/NVActivityIndicatorView" == 3.6.0 //Using Swift 3
github "Alamofire/Alamofire" ~> 4.5 //Using Swift 4
~~~

Over ~
