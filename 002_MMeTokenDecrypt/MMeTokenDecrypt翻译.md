# MMeTokenDecrypt

**原文链接：** [MMeTokenDecrypt](https://github.com/manwhoami/MMeTokenDecrypt)

**作者：**manwhoami

**翻译：**Chensh



## 前言

这个程序利用了在**macOS**上授权访问钥匙串的流程中的一个缺陷，使得可以无需用户身份验证，即可解密或提取出所有存储在 macOS/ OS X/ OSX上面的授权令牌(Authorization Tokens)。

所有的授权令牌都存储在这个目录下：**/Users/*/Library/Application Support/iCloud/Accounts/DSID**。 DSID是每一个iCloud账户在苹果系统里的后端存储格式。

这个**DSID**格式文件使用了**128位AES的CBC模式**[^注1] 和一个空的**初始化向量**进行加密的。而针对这个文件的解密密钥则是存储在用户的**钥匙串**里面一个名为**iCloud**的服务条目下，名字是iCloud账户相关的邮件地址。

这个解密钥匙进行了base64编码，并作为**Hmac算法**[^注2]中的消息体，进行标准的**MD5哈希加密**。这里的问题是，Hmac算法里所需的输入钥匙，被藏在了**MacOS内核**深处。这个钥匙由44个随机字符组成，它是解密DSID文件的关键。本分支已经包含了这个钥匙，就我所知，到目前为止这个钥匙还未在网上被公开过。



## 意义

目前市面上的软件拥有类似功能的有一款名为**“Elcomsoft Phone Breaker”**[^注3]的取证工具。MMeTokenDecrypt是开源的，允许开发者包含这个解密iCloud授权的文件到他们的工程里。

苹果必须要重新设计钥匙串信息。因为本程序fork了一个苹果已经签名的二进制文件作为子进程，当用户看到钥匙串访问请求弹窗时，并没有意识到背后的危险。更进一步，攻击者可以重复弹出钥匙串弹窗给用户，直至用户允许了钥匙串访问为止，因为苹果并没有为拒绝选项设定一个超时。这将会使得iCloud授权令牌被盗窃，而这些令牌可以用来访问几乎所有iCloud的服务项目：**iOS备份，iCloud联系人，iCloud Drive， iCloud 图片库， 查找我的好友，查找我的手机（查看我其他的项目）**。



## 上报反馈历程

* 2016年10月17日开始联系了苹果，我非常详细的阐述了如何破坏用户钥匙串授权的方法，并在不同的macOS系统版本中重现。
* 直到2016年11月6日，没有得到苹果的任何反馈。(委屈😢失落的作者……)
* bug报告包含了破坏整个钥匙串访问。MMeTokenDecrypt只是这个bug的其中一个实现。查看我的其他项目，**OSXChromeDecrypt**是这个bug的另外一种实现。
* 报告中有个值得注意的摘录如下：

> 此外，假如我们是远程攻击者，当“询问钥匙串密码”的选项没有被勾选，那么我们本质上就是通过代码实现强制弹窗提示，迫使用户单击“允许”按钮，这样我们就可以获得密码。但是，如果“通过密码访问钥匙串”的选项被选中，并且用户点击了“拒绝”按钮，那么强制提示则会变得比较棘手。

* 我已经把bug的报告上传到本项目中，并将实时同步苹果的更新。



## 用法

运行python文件：

```shell
$ python MMeDecrypt.py
```

```Shell
Decrypting token plist -> [/Users/bob/Library/Application Support/iCloud/Accounts/123456789]

Successfully decrypted token plist!

bobloblaw@gmail.com Bob Loblaw -> [123456789]

cloudKitToken = AQAAAABYXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~

mapsToken = AQAAAAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~

mmeAuthToken = AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

mmeBTMMInfiniteToken = AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~

mmeFMFAppToken = AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~

mmeFMIPToken = AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~
```



## 注意

假如你是使用 homebrew 安装的python版本，那么你运行这个脚本的时候有可能会遇到以下错误：

```shell
user@system:~/code/MMeTokenDecrypt $ python MMeDecrypt.py
Traceback (most recent call last):
  File "MMeDecrypt.py", line 2, in <module>
    from Foundation import NSData, NSPropertyListSerialization
ImportError: No module named Foundation
```



想要解决这个问题，你可以手动指定一个你系统上默认的python版本的完整的路径。

```shell
user@system:~/code/MMeTokenDecrypt $ /usr/bin/python MMeDecrypt.py
Decrypting token plist -> [/Users/user/Library/Application Support/iCloud/Accounts/123413453]

Successfully decrypted token plist!

user@email.com [First Last -> 123413453]
{
    cloudKitToken = "AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~";
    mapsToken = "AQAAAAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~";
    mmeAuthToken = "AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=";
    mmeBTMMInfiniteToken = "AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~";
    mmeFMFAppToken = "AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~";
    mmeFMIPToken = "AQAAAABXXXXXXXXXXXXXXXXXXXXXXXXXXXXX~";
}
```



**已在Mac OS X EI Capitan 验证过**



## 注释

**注1：**  AES的CBC加密模式。即为密码分组链接模式（Cipher Block Chaining (CBC)），这种模式是先将明文切分成若干小段，然后每一小段与初始块或者上一段的密文段进行异或运算后，再与密钥进行加密。

**注2：** Hmac算法，是密钥相关的哈希运算消息认证码，HMAC运算利用哈希算法，以一个密钥和一个消息为输入，生成一个消息摘要作为输出。

**注3：** [Elcomsoft Phone Breaker](https://www.elcomsoft.com/eppb.html)是Elcomsoft公司的一款手机取证工具，具有解密、下载iCloud文件和备份，获取iCloud的Token，解析钥匙串等功能。