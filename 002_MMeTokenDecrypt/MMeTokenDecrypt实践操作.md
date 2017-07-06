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



