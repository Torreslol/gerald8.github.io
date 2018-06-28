---
layout: post
title: "使用 fastlane 实现自动化发布"
author: "Gerald"
---

## 简介

**fastlane** - 很多ruby脚本的工具集，可以极大的提升我们持续集成，交付的效率。
本文主要记录我在使用 fastlane 来实现 iOS 端第三方托管平台的自动部署 的过程和遇到的问题。主要使用的 actions 包括

<!--more-->
- 使用 *increment_build_bumber* 递增版本号
- 使用 *match* 管理证书和描述文件
- 使用 *gym* 进行打包
- 使用 *hockey* 部署到hockeyapp
- 使用 *mailgun*  发送通知邮件

## Requirements

本文记录的过程在依赖以下环境中
- macOS High Sierra 10.13.5
- ruby 2.5.1
- fastlane 2.98.0

除此之外，本文中还需要用到

- 一个 iOS 工程（废话一）
- 一个 hockeyapp 账号 （废话二）
- 一个 mailgun 账号 （废话三）
- 一个 enterprise 版本的 developer 账号，并且钥匙串中有发布用的证书（含私钥）
- 一个私有的 git 仓库 （用于存放 证书 和 描述文件 ）

## Get Start
cd 到 工程目录下，执行

~~~shell
$ fastlane init
~~~

按照步骤，输入 apple id ， 选择 InHouse/enterprise 类型的初始化，很快工程目录下会生成一个 Gemfile， 一个 fastlane 文件夹， fastlane 文件夹下有我们要编辑的 Fastfile

~~~shell
$ atom Fastfile
~~~

新建一个 `lane` 叫做 upload_to_hockey

~~~ruby
default_platform(:ios)

platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do

   end
end
~~~

## increment_build_number

下面开始我们的工作，打包之前我们需要更改版本号的，fastlane 提供了一个超级简单的 action，只需要一句话

~~~ruby
default_platform(:ios)

platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do
      increment_build_number
   end
end
~~~

默认是每次加1的，如果你想指定的话也很简单,带上参数即可

~~~ruby
increment_build_number(
build_number: "75" # set a specific number
)
~~~
这里要注意一下，使用 *increment_build_number* 需要先设置 Xcode 的 *Current Project Version*（如果没有设为 1） 和 *Version System* 属性如下图，否则会出错。
![](/assets/QA1827_Versioning.png)

## match

fastlane 推荐使用 match 命令而不是直接操作 cert 和 sigh。
match 是 fastlane 的一个功能组件, 采取了集中化方式来管理证书和 profile, 新建一个私有远程 git 库用来保存证书和 profile, 一个 team 的开发者共用同一套证书, 方便了管理和配置, 同时 match 在证书过期时还会自动从苹果官网下载新的证书并 push 到私有的 git 库中, 保证证书同步。先来说一下官方推荐的做法：

初始化

~~~shell
$ fastlane match init
~~~
输入你的私有git仓库地址，命令会生成一个 Matchfile，在 Matchfile 里面可以配置必要的信息

~~~ruby
git_url("https://github.com/fastlane/certificates")
app_identifier("tools.fastlane.app")
username("user@fastlane.tools")
~~~

如果你现在证书管理比较混乱，想用 match 来做统一管理，可以使用以下命令来清空当前所有证书和 profiles

~~~shell
$ fastlane match nuke development
$ fastlane match nuke distribution
~~~

然后生成新的

~~~shell
$ fastlane match development
$ fastlane match adhoc
$ fastlane match appstore
~~~

命令会生成新的证书和描述文件，存放到私有仓库。当团队来了新成员后一样初始化 match ，输入

~~~shell
$ fastlane match development --readonly
$ fastlane match adhoc --readonly
$ fastlane match appstore --readonly
~~~
就会到私有仓库下载证书和描述文件，安装到本地，这里使用readonly 参数指定使用仓库中已经存在证书。

在实际操作的时候，因为我们有很多项目组和开发人员还有上百个应用，为了规避风险，这里我们上传我们钥匙串现有的证书到 git 私有库， 而不是生成新的。

首先，我们需要知道当前 Certificates 的 Cert id ，下面要用到，获取 Cert id 的话 需要用到 *spaceship* 这个工具。

~~~shell
$ code list_certs
~~~
复制以下内容

~~~ruby
require 'spaceship'

Spaceship.login('这里是你的 app id')
Spaceship.select_team

Spaceship.certificate.all.each do |cert|
   cert_type = Spaceship::Portal::Certificate::CERTIFICATE_TYPE_IDS[cert.type_display_id].to_s.split("::")[-1]
   puts "Cert id: #{cert.id}, name: #{cert.name}, expires: #{cert.expires.strftime("%Y-%m-%d")}, type: #{cert_type}"
end
~~~

保存，执行

~~~shell
$ ruby list_certs
~~~
在证书列表里根据 type 和 expires 找到你的证书
~~~
Cert id: 2B7RTV3W77, name: iOS Distribution, expires: 2018-10-20, type: InHouse
~~~
然后记下 Cert id，下面会用到。

接下来要从钥匙串里导出证书和秘钥

右键单击证书，选择 Export... 将证书导出为 certificate.cer 。然后导出私钥并保存为 certificate.p12。记得输入私钥的密码。

下面使用 openssl 对证书和秘钥文件来进行加密, 第一行是把 private key 提取到 pem 文件中，

~~~shell
$ openssl pkcs12 -nocerts -nodes -out key.pem -in certificate.p12

$ openssl aes-256-cbc -k your_password -in key.pem -out cert_id.p12 -a
$ openssl aes-256-cbc -k your_password -in certificate.cer -out cert_id.cer -a
~~~

这里就生成了 2个文件 cert_id.p12 和 cert_id.cer。描述文件也一样

~~~shell
openssl aes-256-cbc -k your_password -in xxxx.mobileprovision -out xxxx.mobileprovision.enc -a
~~~
**这里一定要注意** 生成的 xxx.mobileprovision.enc 文件等会一定要重命名和你的 bundle_id 按规则关联起来，比如我要做的是 enterprise/InHouse 类型的证书，最后的文件名字一定要是 InHouse_yourbundleid。

接着来到我们的 git 私有库目录下新建certs， profiles文件夹，在对应的目录下，创建 enterprise 文件夹，把加密后的证书和描述文件放进去。

最后的目录结构应该是这样的
~~~
.
└── certs
└── enterprise
├── cert_id.cer
└── cert_id.p12
└── profiles
└── enterprise
└── InHouse_yourbundleid
~~~

确认无误后，提交到远程仓库

~~~shell
git add .
git commit -m "xxxxxx"
git push --all
~~~

添加 match 命令到 Fastfile

~~~ruby
default_platform(:ios)

platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do
      increment_build_number
      match(
        username: "xxxxx",
        app_identifier: "",
        type: "enterprise",
        git_url: "xxxxxxxxx",
        verbose: true,
        readonly: true
      )
   end
end
~~~
结束以后可以测试一下结果，如果出错再具体分析。

## gym

打包前，先进行项目配置在项目 PROJECT -> Info -> Configurations 中，点击 + 按钮，Duplicate “Release” Configurations，添加一个 enterprise，上一步 match 命令执行成功后，在项目 TARGETS -> General 中，Signing 取消勾选 Automatically manage signing，就能选到对应的 profile。

添加 gym 命令到 Fastfile

~~~ruby
default_platform(:ios)

platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do
      increment_build_number
      match(
        username: "xxxxx",
        app_identifier: "",
        type: "enterprise",
        git_url: "xxxxxxxxx",
        verbose: true,
        readonly: true
      )
      gym(
        clean:true,
        configuration:"enterprise",
        export_method:"enterprise",
        output_directory:"./fastlane/release"
      )
   end
end
~~~

## hockey
hockeyapp 是国外的一个类似于 蒲公英，fir.im 的平台，当然了它不需要我手持身份证上传来实名认证，所以我选择它。fastlane 的 hockey 命令封装了 hockeyapp 的一些 api，使用起来很简单

~~~ ruby
hockey(
  api_token: "your api token",
  ipa: "./fastlane/build/#{scheme}_#{get_build_number()}.ipa",
  notes: "Changelog"
)
~~~

注意下这里的 api_token 不是hockey dashboard 里的appid。你需要到 Account Setting -> Api Tokens 里去手动创建。这里有一篇图文并茂的教程我就不写了。

上一下这一步之后的 Fastfile

~~~ruby
platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do
      increment_build_number
      match(
        username: "xxxxx",
        app_identifier: "",
        type: "enterprise",
        git_url: "xxxxxxxxx",
        verbose: true,
        readonly: true
      )
      gym(
        clean:true,
        configuration:"enterprise",
        export_method:"enterprise",
        output_directory:"./fastlane/release"
      )
      hockey(
        api_token: "your api token",
        ipa: "./fastlane/build/#{scheme}_#{get_build_number()}.ipa",
        notes: "Changelog"
      )
   end
end
~~~
到这里我们就实现了自动导出 enterprise 类型的ipa，并上传到 hockeyapp。

## mailgun

Mailgun 是一个简便高效的邮件发送云服务，提供丰富的API接口，还有就是他提供每个月3000封免费邮件发送服务，邮件到达率非常高，完全可以满足我们使用。 使用 fastlane 的 mailgun 可以很快集成，而且还有一个好处就是我们不再依赖 jenkins 服务，目前我们使用了 fastlane + jenkins 来构筑 CI/CD 流程。其实 jenkins 我们也只使用了一个Pipeline工具，我们完全基于 fastlane 实现这一套流程以后可以很快的迁移到其他平台。

~~~ruby
mailgun(
  apikey: "xxxxxxxxxxxx",
  postmaster: "xxxxxxxxxxxxx",
  app_link: "xxxxxxxxxxx",
  to: “xxxxxxxxxx",
  success: true,
  message: "send from mailgun"
)
~~~


mailgun 参数与 Maingun 网页 Domain Information 的关系我截了张图


![](/assets/mailgun.png)

最后附上最后的 Fastfile

~~~ruby
platform :ios do
   desc "upload ipa to hockeyapp"
   lane :upload_to_hockey do
      increment_build_number
      match(
        username: "xxxxx",
        app_identifier: "",
        type: "enterprise",
        git_url: "xxxxxxxxx",
        verbose: true,
        readonly: true
      )
      gym(
        clean:true,
        configuration:"enterprise",
        export_method:"enterprise",
        output_directory:"./fastlane/release"
      )
      hockey(
        api_token: "your api token",
        ipa: "./fastlane/build/#{scheme}_#{get_build_number()}.ipa",
        notes: "Changelog"
      )
      mailgun(
        apikey: "xxxxxxxxxxxx",
        postmaster: "xxxxxxxxxxxxx",
        app_link: "xxxxxxxxxxx",
        to: “xxxxxxxxxx",
        success: true,
        message: "send from mailgun"
      )
   end
end
~~~


## 总结

总体来说说 fastlane 使用起来还是很方便的，但是毕竟是要依赖ruby环境的，提到 ruby 就恶心了，像 ruby bundle gem 这些 你很难都理解他们的运行原理。运气不好的话不是这里不对就是那里不对。出错的情况会有很多种，有的在 fastlane 的 github issues 上可以找到，有的很难解决，就比如我就遇到了 cert 命令始终找不到本地钥匙串的证书和私钥的问题。也难怪前段时间有人写了一篇文章里面就吐槽了fastlane的这些问题。 [传送门](https://medium.com/xcblog/five-options-for-ios-continuous-delivery-without-fastlane-2a32e05ddf3d)




