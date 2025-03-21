---
title: 初识ssh
date: 2020-09-09 10:52:02
tags:
---

ssh前言
最近在写自动化脚本，需要登录多台服务器操作。由于频繁输入密码，进行配置免密登录(公钥登录)，在此记录细节。

ssh只是一种计算机之间机密登录的协议，它相对于telnet和rsh的明文传输，
提供了加密、校验和压缩，使得我们可以很安全的远程操作，
而不用担心信息泄露(当然不是绝对的，加密总有可能被破解，只是比起明文来说那是强了不少)。

<!--more-->

# ssh基本用法
远程登录主机host
```
$ ssh user@host
```

如果本地用户名与远程用户名一致，登录时可以省略用户名
```
$ ssh host
```

ssh的默认端口22，使用p参数修改端口
```
$ ssh -p 2222 user@host
```

# 加密简介
加密的意思是将一段数据经过处理之后，输出为一段外人无法或者很难破译的数据，除了指定的人可以解密之外。
一般来说，加密的输入还会有一个key，这个key作为加密的参数，而在解密的时候也会用一个相关联(有可能是相同)的key作为输入。
粗略来说是下面的流程：

```
# 加密方
encrypted_data = encrypt(raw_data, key)
# 解密方
raw_data = decrypt(encrypted_data, key1)
```

目前主流的加密算法一般分为下面两类：
私钥(secret key)加密，也称为对称加密
公钥(public key)加密，也称非对称加密

## 私钥加密
私钥加密，加密方和解密方用的是同一个key，这个key对于加密和解密双方来说都是保密的。
一般来说是加密方先产生私钥，然后通过一个安全的途径来告知解密方这个私钥。

## 公钥加密
公钥加密，解密方生成一对密钥(公钥和私钥)，公钥对外发布，私钥自己保存。用公钥加密的数据，只能私钥进行解密。
加密方首先需要获取到公钥，然后利用这个公钥进行加密，把数据发送给解密方，解密方用私钥进行解密获取数据。
如果解密的数据在传输的过程中被第三方截获，也不用担心，因为第三方没有私钥，没有办法进行解密。

![Lena](/images/ssh02.png)

但是公钥加密还是会有风险，有人冒充解密方发送一对伪造的密钥，那么他就能解出加密方所上送的数据。
那么加密方很难辨别真伪。因为不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，也就是说，都是自己签发的。
这种封建就是著名的"中间人攻击"(Man-in-the-middle attack)。所以公钥加密里面比较重要的一步是身份认证。

![Lena](/images/ssh03.png)

一般的私钥加密都会比公钥加密快，所以大数据量的加密一般都会使用私钥加密，而公钥加密会作为身份验证和交换私钥的一个手段。

## 身份校验
上面讲到公钥加密的风险，那么ssh协议是如何应对的呢？

如果你是第一次登陆对方主机，系统会出现下面提示
```
[para@para2 ~]$ ssh para@47.114.0.16
The authenticity of host '47.114.0.16 (47.114.0.16)' can't be established.
ECDSA key fingerprint is da:eb:96:f4:79:05:fb:3a:2a:34:4f:04:c1:a6:29:39.
Are you sure you want to continue connecting (yes/no)? 
```
这代表，无法确认host主机的真实性，只有他的公钥指纹(ECDSA key fingerprint)，问还想继续连接吗

"公钥指纹"，公钥长度较长（这里采用RSA算法，长达1024位），很难比对，所以对其进行MD5计算，将它变成一个128位的指纹。
上例中是da:eb:96:f4:79:05:fb:3a:2a:34:4f:04:c1:a6:29:39，再进行比较，就容易多了。
然而远程主机必须在自己的网站上贴出公钥指纹，以便用户自行核对。

用户决定接受这个远程主机的公钥。
```
Are you sure you want to continue connecting (yes/no)? yes
```
系统会提示host主机已经得到认可

```
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '47.114.0.16' (ECDSA) to the list of known hosts.
para@47.114.0.16's password: 
```
如果密码正确，就可以登录了。

当远程主机的公钥被接受以后，它就会被保存在文件$HOME/.ssh/known_hosts之中。
下次再连接这台主机，系统就会认出它的公钥已经保存在本地了，从而跳过警告部分，直接提示输入密码。

每个SSH用户都有自己的known_hosts文件，此外系统也有一个这样的文件，通常是/etc/ssh/ssh_known_hosts，
保存一些对所有用户都可信赖的远程主机的公钥。

## 数据完整性
数据一致性说得是如何保证一段数据在传输的过程中没有遗漏、破坏或者修改过。一般来说，目前流行的做法是对数据进行hash，
得到的hash值和数据一起传输，然后在收到数据的时候也对数据进行hash，将得到的hash值和传输过来的hash值进行比对，
如果是不一样的，说明数据已经被修改过；如果是一样的，则说明极有可能是完整的。

目前流行的hash算法有MD5和SHA-1算法。

# ssh连接过程

## 口令登录
(1)身份校验：用户同意后进行登录操作
(2)远程主机收到用户的登录请求，把自己的公钥发给用户。
(3)用户使用这个公钥，将登录密码加密后，发送回来。
(4)远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

## 公钥登录
口令登录每次都要输入密码，非常麻烦，ssh提供了公钥登录，省去了输入密码的步骤

公钥登录原理，用户将自己的公钥存在远程主机上。登录的时候，远程主机会向用户发送
一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先存储的公钥进行解密，
解密成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

![Lena](/images/ssh01.png)

操作步骤：
1.ssh-keygen -t rsa 命令生成一对密钥对，放在 $HOME/.ssh/目录下，分别为id_rsa.pub和id_rsa。
id_rsa是私钥(保存)，id_rsa.pub是公钥(给服务器保存)

2.输入命令  ssh-copy-id user@host,将公钥发送至远程host上

3.检查远程主机的配置文件 /etc/ssh/sshd_config 是否开启
```
　RSAAuthentication yes
　PubkeyAuthentication yes
　AuthorizedKeysFile .ssh/authorized_keys
```
修改后重启ssh服务
```
ubuntu系统 service ssh restart 
debian系统 /etc/init.d/ssh restart
```

4.修改远程主机配置文件
chmod 600 $HOME/.ssh/authorized_keys
chmod 700 $HOME/.ssh/

# authorized_keys和known_hosts文件
当新连接一台远程主机，都会进行身份校验，公钥指纹保存在$HOME/.ssh/known_hosts文件中。
远程主机将用户的公钥，保存在登录后的用户主目录的$HOME/.ssh/authorized_keys文件中。

因此可以直接将客户端的公钥追加到远程主机的authorized_keys文件中
命令如下
```
　$ ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```


