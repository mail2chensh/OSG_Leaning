# MMeTokenDecrypt实践操作

* 本文实践操作基于[MMeTokenDecrypt](https://github.com/manwhoami/MMeTokenDecrypt)项目。
* 查看翻译请访问：[MMeTokenDecrypt翻译](https://github.com/mail2chensh/OSG_Leaning/blob/master/002_MMeTokenDecrypt/MMeTokenDecrypt%E7%BF%BB%E8%AF%91.md)
* 实践者： Chensh



## 前言背景

在MacOS系统的深处，有一把神秘的小钥匙。这把密钥由44个随机字符组成，它的作用非常大。

有多大？它可以用来解开我们存储在系统里面的授权Token！

我们都知道（未读原文前我根本就不知道，😆），在MacOS里，用户的授权Token都存储在**/Users/\*/Library/Application Support/iCloud/Accounts/**这个目录下，以用户的**iCloud账号**（邮箱）为名的文件，其存储格式是**DSID**，这个文件是加密过的。

那平常系统需要用到这个文件里面存储的token的时候，是如何解密的呢？

这里涉及另外一个密钥，这个密钥存储在用户的**keyChain**里面，一个名为iCloud的条目。

系统需要解密这个DSID文件时，就会从keyChain里面拿出解密密钥，进行**base64编码**，作为**Hmac算法**的**消息体**输入，然后和上面提到的神秘小钥匙进行了**MD5加密**，得到一把新的钥匙。

而这个DSID文件，是采用128位AES的CBC加密模式加密的。CBC加密模式就是密码分组链接模式，这种模式是先将明文切分成若干小段，然后每一小段与初始块或者上一段的密文段进行异或运算后，再与密钥（也就是上面**新**产生的那把钥匙）进行加密。当然，这个过程还混合了一个空的初始化向量进行计算。

所以，根据上文的分析，只要我们找到系统的神秘小钥匙和拿到keyChain里面iCloud的存储条目，那我们就可以尝试解密DSID这个文件，拿到用户的Token。

而这里的神秘小钥匙已经被原文作者找到了并提供出来，作者在原文里主要阐述的是**keyChain里面存在的bug，导致解密密钥容易被泄露**。

那么接下来，我们就开始复现作者的思路。



## keyChain的不安全性

本来keychain作为用户存储密码的载体，应该是做到很安全的，但这里有一个原文作者认为的bug，恶意用户可以根据这个bug来绕过钥匙串密码从而拿到keychain里面存储的条目。当然，这个绕过方式是有条件的，以下分别介绍两种情况。

#### 当用户没有设防

我们首先打开keychain应用(Spotlight里面输入Keychain Access进行搜索)。

然后在左侧**种类**栏目**密码**一项进行选中，右侧就会出现存储在本机上的所有安全信息存储。这里进行筛选，找到我们的试验目标：**Chrome Safe Storage**。这个条目存储了谷歌Chrome浏览器的安全存储信息解密密钥。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fhb0jtvyiqj31kw087400.jpg)

那要如何拿到这个解密密钥呢？上文我们已经提及到了，这里打开命令行。输入以下命令：

```Shell
$ security find-generic-password -ga 'Chrome'
```

这里使用到了 security命令，它是非常强大命令行安全工具，之前我们[重签](http://chensh.top/2017/06/30/Patching-and-ReSigning-iOS-Apps/)的那篇文章里面也使用到了这个命令去查找有效的证书。

这里的参数解释如下：

* **find-generic-password** 命令参数，是使用“查找密码”的功能。
* **-a**，这个参数是匹配类型为“账户”的，用于过滤。
* **-g**，这个参数是将查询到的密码显示出来。

运行这个命令后，系统马上就会弹出一个提示框，如下：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fhb0zu1parj30ok0badhp.jpg)



那么到这里，只要用户点击了“允许”按钮，那么就可以马上得到存储在keychain里面的密钥了。如下：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fhblsp9swej30zg0ngtd0.jpg)

这里security请求钥匙串访问的权限，用户点击“允许”后，就能读取到相应的password。

假如有人未经你允许擅动你电脑，这个时候就可以轻而易举的窃取了你密码。

当然，如果是远程的恶意黑客，那么有可能会无限次弹出弹窗提示，直至逼到你点击“允许”按钮为止。

那我们该如何防范这种情况呢？请往下看。



#### 当用户开启了“询问钥匙串密码”选项

我们回到刚刚keychain 里面那个chrome safe storage的条目，右键选择**显示简介**。

切换到**访问控制**的面板，将**询问钥匙串密码**选中。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fhblzkxuz7j30ty0jkgnl.jpg)



这里有另外一个小bug，就是keychain里面的更改需要重复两次操作，才能真正的修改成功，当我们勾选了这个选项，切换到*属性*然后再切换回来的时候，发现这个选项又没有被选中，需要再重复操作一遍才行。

然后我们回到命令行，重新输入上面security的命令，这一次我们发现弹窗出来不一样了。这里需要我们输入钥匙串密码。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fhbm32ymorj30oe0b8tao.jpg)

所以当我们开启了这个选项，就算别人偷偷用你的电脑，没有你的钥匙串密码，也无法得到其中的安全信息。这里就多了一道防护。



以上的操作步骤，我们主要对是否开启钥匙串密码的安全性做一点讨论。接下来我们要使用脚本来自动获取密码，并且结合系统的神秘小钥匙，用来解密我们iCloud条目里面的Token。



## MMeToenDecrypt.py 解析

先引入所需要的库。

```python
import base64, hashlib, hmac, subprocess, sys, glob, os, binasicc
from Foundation import NSData, NSPropertyListSerialization
```

* base64用来加密iCloud的钥匙。
* hashlib用于计算md5
* subprocess用于fork一个子进程，并运行一个外部程序。



按照上面的方法，使用security命令获取iCloud条目的密码。

```Shell
security find-generic-password -ws 'iCloud'
```

* -w 参数是用于仅显示密码项。
* -s 参数是用于指定类型为server的条目，并匹配后面的关键词。

在python里面，我们可以是subprocess库来运行外部程序，将shell环境下命令运行的结果返回到本程序。如果没有获取到，则打印报错信息，再退出程序。

```python
iCloudKey = subprocess.check_output("security find-generic-password -ws 'iCloud' | awk {'print $1'}", shell=True).replace("\n", "")
if iCloudKey == "":
    print "Error getting iCloud Decryption Key"
    sys.exit()
```

得到iCloud的密码后，我们将其进行base64编码

```python
msg = base64.b64decode(iCloudKey)
```

接下来上场的是我们那把神秘的小钥匙了。

它在所有的Macos版本里面都用于生成Hmac哈希。它是用于解密的关键。

在位于**/System/Library/PrivateFrameworks/AOSKit.framework/Versions/A/AOSKit**路径下的系统库，执行了下面的方法，调用了CCHmac来生成一个Hmac，用于解密秘钥。

```swift
KeychainAccountStorage _generateKeyFromData:
```

```python
key = "t9s\"lx^awe.580Gj%'ld+0LG<#9xa?>vb)-fkwb92[}"
```

将上面得到的iCloudKey和系统神秘钥匙使用hmac进行md5计算。

并将我们得到的哈希值进行16进制转换。

```python
hashed = hmac.new(key, msg, digestmod=hashlib.md5).digest()
hexedKey = binascii.hexlify(hashed)
```

我们知道用户的授权Token都存放在**~/Library/Application Support/iCloud/Accounts/** 下，所以要遍历一下该文件夹下面的文件，判断是否为我们所需要的DSID文件。

```python
mmeTokenFile = glob.glob("%s/Library/Application Support/iCloud/Account/*" % os.path.expanduser("~"))
for x in mmeTokenFile:
    try:
        int(x.split("/")[-1])
        mmeTokenFile = x
    except:
        continue
if not isinstance(mmeTokenFile, str):
    print"Could not find MMeTokenFile. You can specify the file manually"
  	sys.exit()
else:
    print "Decrypting token plist -> [%s]\n" % mmeTokenFile
```







