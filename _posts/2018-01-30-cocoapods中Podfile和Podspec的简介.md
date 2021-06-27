---
layout: post
title: cocoapods中Podfile和Podspec的简介
date: 2018-01-30 00:00:00.000000000 +09:00
---

## Podfile

普通的Project，通过cd到根目录，执行`pod init`便会生成Podfile文件

一个简单的Podfile是这样的：

```ruby
target 'test' do
pod 'AFNetworking', '~> 3.1.0'
end
```



## Dependencies

### pod

第一行即指定Project中具体的target

第二行是 `pod 'AFNetworking'`

前一段表示指定依赖的库，这里即为AFNetworking

后一段`3.1.0`即为版本号

- #### Version

  中间这一段`~>`是一个符号，除了`~>`其实还有很多，如下

  - `= 0.1` 指定版本为0.1
  - `> 0.1` 大于 0.1，如果有很多，取最高版本
  - `>= 0.1` 大于等于 0.1，如果有很多，取最高版本
  - `< 0.1` 小于 0.1，如果有很多，取最高版本
  - `<= 0.1` 小于等于 0.1，如果有很多，取最高版本
  - `~> 0.1.2` 大于等于 0.1.2且小于0.2，如果有很多，取最高版本
  - `不写 `  始终取最高版本

- #### From a podspec in the root of a library repository.

  除了版本，还可以通过git的其他方式拉取

  To use the `master` branch of the repository:

  ```ruby
  pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git'
  ```

  To use a different branch of the repository:

  ```ruby
  pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :branch => 'dev'
  ```

  To use a tag of the repository:

  ```ruby
  pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :tag => '0.7.0'
  ```

  Or specify a commit:

  ```ruby
  pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git', :commit => '082f8319af'
  ```

- #### Build configurations

  默认情况下， 依赖项会被安装在所有target的build configuration中。当然指定具体的某个configuration如下

  ```ruby
  pod 'PonyDebugger', :configurations => ['Debug', 'Beta']
  ```

- #### Subspecs

  当你用一个名字装Pod的时候，它将安装所有定义在podspec里面的默认subspec

  你可以这样指定:

  ```ruby
  pod 'QueryKit/Attribute'
  ```

  也可以指定一个集合，像下面这样:

  ```ruby
  pod 'QueryKit', :subspecs => ['Attribute', 'QuerySet']
  ```

- #### Using the files from a local path

  如果你想要一个自己开发的本地Pod，你可以用```path```，如下

  ```pod 'AFNetworking', :path => '~/Documents/AFNetworking'```

  一般在做独立的库时，库的Example可以直接通过path引用自己的库，如下图中XXCommonExample的Podfile

  ![alt](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2018-03-13-远程私有仓库创建/远程私有仓库创建3.png)

  可以这么写：

  ```ruby
  pod 'XXCommon', :path => '../'
  ```




## Sources

Podfile检索的源，全局的，不存储在某个target定义里面

### source

指定specs的位置，如果有远程私有仓库，比如我们自己的gitlab，需要写上类似如下的源定义，而cocoapods 官方source是隐式的需要的，一旦你指定了其他source 你就需要也把官方的指定上

```ruby
source 'https://github.com/CocoaPods/Specs.git'
source 'http://gitlab.zhenai.com/ios/XXSpec.git'
```



## Hooks

Podfile提供了钩子用来在安装时被调用

Hooks也全局的，不存储在某个target定义里面

### plugin

使用plugin可以指定某个插件在安装过程中使用

如，指定用`slather` 和 `cocoapods-keys`插件

```ruby
plugin 'cocoapods-keys', :keyring => 'Eidolon'
plugin 'slather'
```

### pre_install

这个钩子允许你在Pods被下载后但是还未安装前对Pods做一些改变

例子:

让cocoapods对静态库支持

```ruby
pre_install do |installer|
    # workaround for https://github.com/CocoaPods/CocoaPods/issues/3289
    Pod::Installer::Xcode::TargetValidator.send(:define_method, :verify_no_static_framework_transitive_dependencies) {}
end
```

### post_install

这个钩子允许你在生成的Xcode project写入硬盘或者其他你想执行的操作前做最后的改动

例子:

给所有target自定义编译配置

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['GCC_ENABLE_OBJC_GC'] = 'supported'
    end
  end
end
```



## Podspec

如果要自己写一个库，就必须要了解与Podfile相对应的Podspec

一个Podspec规范描述了库的一个版本，它包含库的名称、版本、说明、要使用的文件等信息

通过cd到根目录，执行`pod spec create [name]`即可生成一个.podspec

一个简单的Podspec是这样的：

```ruby
Pod::Spec.new do |spec|
  spec.name         = 'Reachability'
  spec.version      = '3.1.0'
  spec.license      = { :type => 'BSD' }
  spec.homepage     = 'https://github.com/tonymillion/Reachability'
  spec.authors      = { 'Tony Million' => 'tonymillion@gmail.com' }
  spec.summary      = 'ARC and GCD Compatible Reachability Class for iOS and OS X.'
  spec.source       = { :git => 'https://github.com/tonymillion/Reachability.git', :tag => 'v3.1.0' }
  spec.source_files = 'Reachability.{h,m}'
  spec.framework    = 'SystemConfiguration'
end
```

## Root specification

七大必要元素:

### name

库的名字

```ruby
spec.name = 'AFNetworking
```

### version

库的版本

```ruby
spec.version = '0.0.1'
```

### authors

库的作者

```ruby
spec.version = '0.0.1'
```

### license

库的证书

```ruby
spec.license = 'MIT'
```

### homepage

库的首页

```ruby
spec.homepage = 'http://www.example.com'
```

### license

库的源

```ruby
spec.source = { :git => 'https://github.com/AFNetworking/AFNetworking.git',
                :tag => spec.version.to_s }
```

### summary

库的简述

```ruby
spec.summary = 'Computes the meaning of life.'
```



## Platform

库的支持平台

如iOS，8.0以上

```ruby
s.platform = :ios, '8.0'
```



## Build settings

### dependency

依赖的库

作为一个库，有时候需要依赖其他的库，如XXNetworking网络库会依赖AFNetworking，语法和pod差不多，不过后面只能跟版本号，其他依赖网络库的工程通过pod依赖了XXNetworking，下载时会同时把AFNetworking也下载过来

```ruby
spec.dependency 'AFNetworking', '~> 3.1.0'
```

### requires_arc

指明哪些文件使用ARC。默认是true，表明全部文件都是ARC，或者设置为false，指定所有ARC的文件

例：

```ruby
spec.requires_arc = true
```

或者

```ruby
spec.requires_arc = false
spec.requires_arc = 'Classes/Arc'
spec.requires_arc = ['Classes/*ARC.m', 'Classes/ARC.mm']
```

对于不是ARC的文件会自动添加`-fno-objc-arc` compiler flag



### frameworks

依赖的系统frameworks，多个framework,用逗号分开

例：

```ruby
spec.frameworks = 'QuartzCore', 'CoreData'
```

### weak_frameworks

如果在高版本的OS中调用新增的功能，并且在低版本的OS中依然能够运行,那么就要用到weak_frameworks.如果引用的某些类或者接口在低版本中并不支持，对于不支持的接口，可以在运行的时候判断，这样程序不会出错，如果不weak引用，程序在低版本下启动的时候就会崩溃掉

例:

```ruby
spec.weak_frameworks = 'Twitter', 'SafariServices'
```

### libraries

依赖的系统libraries，多个用逗号分开，去掉lib开头

例如

```ruby
spec.libraries = 'xml2', 'z'  #z表示libz.tdb,后缀不需要,lib开头的省略lib
```

### prefix_header_contents

类似于pch，有多个用逗号隔开

例：

```ruby
spec.prefix_header_contents = '#import <UIKit/UIKit.h>', '#import <Foundation/Foundation.h>'
```

或者

```ruby
s.prefix_header_contents = <<-EOS
 #ifdef __OBJC__
 #import "XXPrefix.h"    //XXPrefix包含了所有头文件
 #endif 
EOS
```

### pod_target_xcconfig

表示pod 本身被依赖时，修改的编译选项

比如framework的路径等

```ruby
s.pod_target_xcconfig = {
    'FRAMEWORK_SEARCH_PATHS' => '$(inherited) $(PODS_ROOT)/GTSDK $(PODS_ROOT)/Bugly $(PODS_ROOT)/ksyhttpcache/framework $(PODS_ROOT)/KSYMediaPlayer_iOS/framework/live',
    'OTHER_LDFLAGS'            => '$(inherited) -undefined dynamic_lookup -ObjC',
    'ENABLE_BITCODE'           => 'NO'
  }
```



## File patterns

### source_files

源文件

```ruby
spec.source_files = 'Classes/**/*.{h,m}'
spec.source_files = 'Classes/**/*.{h,m}', 'More_Classes/**/*.{h,m}'
```

### exclude_files

source_files中指定排除的文件

```ruby
spec.exclude_files = 'Classes/**/unused.{h,m}'
```

### vendored_frameworks

依赖的本地三方库路径

```ruby
spec.ios.vendored_frameworks = 'XXBaseBusinessModule/Classes/BaseModule/ZMCredit/ZMCreditSDK/*.framework'
```

### vendored_libraries

依赖的本地三方.a文件路径

```ruby
spec.ios.vendored_libraries = 'Libraries/libProj4.a'
```

### resource_bundles

资源文件，比如图片，音频视频等

```ruby
s.resources = 'XXBaseBusinessModule/Resources/**/*'
```

值得注意的是，通过pods导入的资源将不在MainBundle下，所有资源会生成在一个新的Bundle下

所以取资源的时候需要指定bundle来获取

```objective-c
[UIImage imageNamed:imageName inBundle:[NSBundle bundleForClass:[self class]] compatibleWithTraitCollection:nil];
```



## 参考文献

[Podfile Syntax Reference](https://guides.cocoapods.org/syntax/podfile.html#podfile)

[Podspec Syntax Reference](https://guides.cocoapods.org/syntax/podspec.html)