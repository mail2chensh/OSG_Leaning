# 对iOS Apps打补丁以及重签名

##### 原著地址：[Patching and Re-Signing iOS Apps](http://www.vantagepoint.sg/blog/85-patching-and-re-signing-ios-apps)

##### 作者： Bernhard Mueller

##### 翻译： Chensh



## 前言

在没有越狱的设备上运行经过修改的iOS二进制文件，听起来是一个不错的主意，特别是当你的越狱机子变砖后，你想把机子更新为非越狱的iOS版本的时候（尽管我和我身边的人重来没遇到过这种事情）。

你可以使用这种技术对app进行动态分析，假如你在非洲，你可以做一个虚假的GPS定位来欺骗锁区的Pokemon，又不想被越狱检测。不管如何，你可以跟着下面的教程来修改一个app并且重签，然后让它运行在你未越狱的设备上。注意这里的技术实现的前提条件是需要先砸壳，特别是来自App Store的App。



由于苹果有代码签名系统和配置描述的规定，使得重签一个app不是一项简单的挑战。如果一个app的配置描述文件和代码签名头部不完全一致，则会被iOS系统拒绝运行。这就要求你需要学习许多相关的概念——证书、包名、应用ID、团队标识符以及如何使用苹果的构建工具将他们绑定在一起。完全可以说，不使用Xcode这种默认方式去构建一个系统可以运行的特定的二进制文件，是一个巨大的挑战。



我们将要使用的工具包括[optool](https://github.com/alexzielenski/optool),苹果的构建工具以及一些Shell命令。这个方法中的重签脚本灵感来源于 [Vincent Tan's Swizzler project](https://github.com/vtky/Swizzler2/wiki)。另外的重打包来源于[NCC group](https://www.nccgroup.trust/au/about-us/newsroom-and-events/blogs/2016/october/ios-instrumentation-without-jailbreak/)。



要复现以下步骤，请先下载一个app:[UnCrackable iOS App Level 1](https://github.com/OWASP/owasp-mstg/blob/master/Crackmes/iOS/Level_01/UnCrackable_Level1.ipa),来自OWASP移动测试指南的分支。我们的目标是使得 UnCrackable 这个app在启动的时候加载 FridaGadget.dylib 这个动态库，这样我们就能使用Frida来进行分析。



## 获取一份开发者配置文件和证书

配置文件是由你的一个或多个设备上的代码签名证书集成的一份plist格式的文件，由苹果进行白名单签名验证。就是说，苹果明确的要求你的应用运行在某一特性的环境里，例如在选定的设备上进行调试。配置文件也列出了你的应用拥有的权限，而代码签名证书里面包含了将要用于真实签名的私钥。

根据你是否已经注册为iOS开发者，你可以选择一下两种方式之一去获取证书和配置文件。



####  使用iOS开发者账号：

如果你之前曾使用Xcode开发和部署过iOS应用，那么你已经拥有了一个代码签名证书。使用**security**工具可以列出你当前存在的签名标志：



```shell
$ security find-identity -p codesigning -v
1) 61FA3547E0AF42A11E233F6A2B255E6B6AF262CE "iPhone Distribution: Vantage Point Security Pte. Ltd."
2) 8004380F331DCA22CC1B47FB1A805890AE41C938 "iPhone Developer: Bernhard Müller (RV852WND79)"
```



注册开发者账号能够从苹果开发者网站获取到配置文件，这里有份指南，可以指导第一次创建相应的证书和配置文件，[戳这里](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html)。对于重打包来说，并不特定要求你选择了什么应用ID，你甚至可以重用一个已经存在的ID。最重要的事情是拥有一份匹配的配置文件。确保你创建了一份开发环境的配置文件，而不是一份发布描述文件，这样你才能对app进行调试。



在以下的shell命令列表中，我将会使用我公司的开发团队关联的个人签名身份。并创建了一个应用id为"sg.vp.repackaged",以及一将配置文件取名为"AwesomeRepackaging"，这样生成出来的配置文件就是"AwesomeRepacaging.mobileprovision"——这里需要更换成你自己的id名和配置文件名。



#### 使用常规的iTunes账号：

虽然你不是一个付费开发者，但苹果还是提供了一个免费的开发者配置文件。你可以在Xcode里面使用你的Apple账号获取这个配置文件，只需简单的编译一个空的iOS工程，然后从app资源包里面提取出这个**embedded.mobileprovision**，具体可以参考[NCC blog](https://www.nccgroup.trust/au/about-us/newsroom-and-events/blogs/2016/october/ios-instrumentation-without-jailbreak/)的这篇博文。



当你获取到这个配置文件后，你可以使用**security**这个工具来查看它的内容。除了许可证书以及设备外，你还能在这个描述文件找到app的权限授权表。在之后的代码签名里面，你需要这个表，所以下面将演示如何将他们提取出来到一个plist格式的文件里。也可以看看文件的内容，看看是否如预期般。

```shell
$ security cms -D -i AwesomeRepackaging.mobileprovision > profile.plist
$ /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > entitlements.plist
$ cat entitlements.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>application-identifier</key>
<string>LRUD9L355Y.sg.vantagepoint.repackage</string>
<key>com.apple.developer.team-identifier</key>
<string>LRUD9L355Y</string>
<key>get-task-allow</key>
<true/>
<key>keychain-access-groups</key>
<array>
<string>LRUD9L355Y.*</string>
</array>
</dict>
</plist>
```



注意到我们的App ID是由团队标识符（LRUD9L355Y）以及包名(sg.vantagepoint.repackage)组合而成的。这个配置文件只对携带这样特定的app id才有效。“get-task-allow”这个键非常重要，当它为true时，像debugging server这样的其他进程才能够附加到这个app。可想而知，在发布描述文件里面，这个键的值肯定是false的。



#### 其他准备工作

为了在我们的app启动的时候加载一个附加的库，我们需要插入一条额外的加载命令到主程序的Mach-O头中。这里我们使用[optool](https://github.com/alexzielenski/optool)工具来自动化这个工程：

```shell
$ git clone https://github.com/alexzielenski/optool.git
$ cd optool/
$ git submodule update --init --recursive
```



我们也将使用到[ios-deploy](https://github.com/phonegap/ios-deploy)这个工具，这个工具能够在不使用Xcode的情况下发布或者调试iOS。（npm是Nodejs的包管理器，如果你还没有安装[Nodejs](https://nodejs.org/en/)，你可以使用homebrew来安装，或者到官网直接下载安装包）

```shell
$ npm install -g ios-deploy	
```



另外，在教程开始前，你还需要先把 FridaGadget.dylib 这个动态库下载下来。

```shell
$ curl -O https://build.frida.re/frida/ios/lib/FridaGadget.dylib
```



除了以上提到的工具，我们还需要使用一些原生系统工具和Xcode的编译工具，所以确保你已经安装了Xcode命令行开发工具。



#### 打补丁，重打包以及重签名

来不及了，快上车吧！

如你所知，IPA文件其实一种ZIP压缩格式，所以我们可以使用zip工具来解压缩。然后将**FridaGadget.dylib**拷贝到文件目录下。并且使用**optool**工具将一条加载命令插入到**UnCrackable Level 1**这个二进制文件内。

```shell
$ unzip UnCrackable_Level1.ipa
$ cp FridaGadget.dylib Payload/UnCrackable\ Level\ 1.app/
$ optool install -c load -p "@executable_path/FridaGadget.dylib" -t Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Found FAT Header
Found thin header...
Found thin header...
Inserting a LC_LOAD_DYLIB command for architecture: arm
Successfully inserted a LC_LOAD_DYLIB command for arm
Inserting a LC_LOAD_DYLIB command for architecture: arm64
Successfully inserted a LC_LOAD_DYLIB command for arm64
Writing executable to Payload/UnCrackable Level 1.app/UnCrackable Level 1...
```



这种对主程序明显的篡改将会使得它的代码签名失效。所以这个无法再非越狱机子上面运行。所以你将需要替换掉配置文件，并使用描述文件中列举的证书对主程序和FridaGadget.dylib进行重新签名。

第一步，将我们的自己的配置文件拷贝到资源包里面：

```shell
$ cp AwesomeRepackaging.mobileprovision Payload/UnCrackable\ Level\ 1.app/embedded.mobileprovision\
```



第二步，我们需要确保在**Info.plist**中的包名是否跟我们的描述文件里面指定的一致。因为在重签的过程中，**codesign**会检查我们的Info.plist文件里面的包名，如果不匹配则会返回一个错误值。

```shell
$ /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier sg.vantagepoint.repackage" Payload/UnCrackable\ Level\ 1.app/Info.plist
```



最后，我们需要使用代码签名工具来重签所有的二进制文件：

```shell
$ rm -rf Payload/F/_CodeSignature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938 Payload/UnCrackable\ Level\ 1.app/FridaGadget.dylib
Payload/UnCrackable Level 1.app/FridaGadget.dylib: replacing existing signature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938 --entitlements entitlements.plist Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Payload/UnCrackable Level 1.app/UnCrackable Level 1: replacing existing signature
```



#### 安装并运行应用

现在你可以开始部署并运行修改后的app了，如下操作：

```shell
$ ios-deploy --debug --bundle Payload/UnCrackable\ Level\ 1.app/
```

如果一切顺利的话，app应该会附加**lldb**并以调试模式运行在设备上。Frida现在也附加到应用上了，可是使用**frida-ps**命令来验证一下：

```shell
$ frida-ps -U
PID Name
--- ------
499 Gadget
```

现在你可以使用Frida来正常调试你的应用了！



#### 故障排除

如果发生什么错误，有可能是配置文件和代码签名头不匹配，这种情况建议阅读一下[官方文档](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html)，了解一下整个系统是如何运行的。另外还可以参考[苹果授权故障排除](http://https//developer.apple.com/library/content/technotes/tn2415/_index.html)这个页面。



#### 关于这篇文件

这篇文章是[Mobile Reverse Engineering Unleashed](safari-reader://www.vantagepoint.sg/blog/83-mobile-reverse-engineering-unleashed)的系列之一。你可以访问[这里](http://www.vantagepoint.sg/blog/categories/17-mobile-reverse-engineering)来查看更多相关文章。

