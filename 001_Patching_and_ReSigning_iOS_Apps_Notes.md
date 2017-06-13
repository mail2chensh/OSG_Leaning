# iOS Apps打补丁及重签名实战操作笔记

**原文：** [Patching and Re-Signing iOS Apps](https://github.com/mail2chensh/OSG_Leaning/blob/master/Patching_and_ReSigning_iOS_Apps.md)

以下笔记内容基于上面👆的翻译文章后的实战操作，没有阅读过原文的请先阅读一遍。



## 工欲善其事必先利其器

#### 1. 安装包及Frida动态库

请先下载以下素材：

* App安装包：  [UnCrackable_Level1.ipa](https://github.com/OWASP/owasp-mstg/blob/master/Crackmes/iOS/Level_01/UnCrackable_Level1.ipa)
* Frida动态库：[FridaGadget.dylib](https://build.frida.re/frida/ios/lib/FridaGadget.dylib)



#### 2. mobileprovision 文件

拥有开发者账号的，可以直接登录苹果开发者官网，然后新建一个App ID。

这里我创建的App ID是： **com.osg.uncrack**

根据文章要求，进而创建一个provisioning文件： **COM_OSG_UNCRACK.mobileprovision**



#### 3. 创建 entitlements.plist文件

创建一个权限文件，名为：**entitlements.plist**，然后填入一下内容注意需要将其中的标识符改为你自己的账号标识符。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>application-identifier</key>
<string>P683E34A2C.com.osg.uncrack</string>
<key>com.apple.developer.team-identifier</key>
<string>P683E34A2C</string>
<key>get-task-allow</key>
<true/>
<key>keychain-access-groups</key>
<array>
<string>P683E34A2C.*</string>
</array>
</dict>
</plist>
```



#### 4. 下载并生成optool工具

根据原文的操作，我们在github上面克隆optool的仓库，并更新子模块。

![](https://ww1.sinaimg.cn/large/006tNbRwgy1fgjwrnkteyj30uw06otae.jpg)

![](https://ww1.sinaimg.cn/large/006tNbRwgy1fgjwsvftckj31kw03wabr.jpg)



更新完毕后，我们用Xcode打开工程，运行一下，就会生成optool工具了。在工程目录下的Produce里面，右键选择**Show in Finder**，然后在打开的文件夹里面，将文件拷贝到目录**/usr/bin/**下。

![](https://ww2.sinaimg.cn/large/006tNbRwgy1fgjwwzryj2j30jq0zyak4.jpg)



这样就可以在终端里面直接使用optool这个命令了。

![](https://ww2.sinaimg.cn/large/006tNbRwgy1fgjwypdkr2j31ja0lwwj5.jpg)



#### 5. 查询本地的证书

使用**security**命令查询本地钥匙串里面的证书。

![](https://ww1.sinaimg.cn/large/006tNbRwgy1fgjx0d7gsaj31hk054q4m.jpg)

这里我们将要用到的是第二个，也就是developer开发环境下的证书。我们把唯一标识码记录下来，一会重签名的时候会用到。



#### 6. 安装ios-depoy

```shell
npm install -g ios-deploy
```



#### 7. 安装Frida

```shell
sudo pip install frida
```



到这里，基本上需要的东西都已经准备齐全了，我们新建一个文件夹**resign**，将上面所有的东西都放在一起，接下来就要开始打补丁和重签名了。

![](https://ww2.sinaimg.cn/large/006tNbRwgy1fgjx3vv497j30rg07adhd.jpg)



## 拆包清洗



#### 1. 将ipa包加压缩

```shell
unzip UnCrackable_Level1.ipa
```

可以看到解压出一个Payload文件夹。

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fgjxawbhnjj311407omz4.jpg)



#### 2. 拷贝动态库到资源包里面

```shell
$ cp FridaGadget.dylib Payload/UnCrackable\ Level\ 1.app/
```

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fgjxmtger6j313010c15o.jpg)



#### 3. 额外加载动态库，使用optool插入加载命令到二进制文件

```shell
$ optool install -c load -p "@executable_path/FridaGadget.dylib" -t Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
```

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fgjxomhgj6j31kw09bn04.jpg)

其中 **-c** 为指定 **load_command** 命令，**-p** 指定动态库的路径， **-t**指定目标文件。



#### 4. 拷贝描述文件，并改名为embedded.mobileprovision

```shell
$ cp COM_OSG_UNCRACK.mobileprovision Payload/UnCrackable\ Level\ 1.app/embedded.mobileprovision
```



#### 5. 将info.plist文件里面的包名更改为你自己的包名

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fgjxs4z6ckj30tq0y87cn.jpg)



#### 6. 使用codesign来进行代码签名

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fgjxtd3i1gj31kw0cf0x1.jpg)



## 安装运行

到这里基本上打补丁和重签名就完成了，接下来可以使用ios-deploy这个工具来安装应用到手机，也可以直接重新压缩为zip格式并改为ipa后缀来安装。

```shell
$ ios-deploy --debug --bundle Payload/UnCrackable\ Level\ 1.app/
```

可以看到启动成功，并安装到手机上面，而且还进入lldb的调试模式：

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fgjxhgrv55j31kw097tef.jpg)



接下来我们来验证一下我们的frida是否成功插入app里面。

运行下面的命令：

```shell
$ frida-ps -U
```

可以看到对应的应用：

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fgjxjm8zv5j30bw04kq34.jpg)



至此，我们的重签名步骤也全部完成了。✌️