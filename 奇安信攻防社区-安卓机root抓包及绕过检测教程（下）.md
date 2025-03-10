&lt;!--

- @Descripttion:
- @version:
- @@Company: 52i.xyz
- @Author: GLRpiz
- @Date: 2021-12-29 17:51:23
- @LastEditors: GLRpiz
- @LastEditTime: 2021-12-29 19:30:25  
    --&gt; 安卓机root抓包及绕过检测教程（下）
    ===================
    
    1.如何刷入lsposed框架
    ---------------
    
    Lsposed的刷入需要在Magisk中进行。  
    目前我们所使用的是Riru-Lsposed，也就是以Riru框架为前置，来实现Zygote进行注入的Lsposed框架版本。  
    **Magisk中内置了插件库，可以在线安装插件**
    
    ### 1.1 刷入Riru和Riru-Lsposed
    
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-7b5ff35c1f941e7df0ec530354d3ac9f10d27a38.jpg)  
    此处，我们需要先**安装Riru，再安装Riru-Lsposed**,顺序反了会报错，因为Riru-Lsposed依赖于Riru，检测不到会不让你安装。  
    通过右下角的放大镜可以进行搜索。  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f73b3366b9dcfd089f4e972d0c20b4ae8fd1a5b7.jpg)  
    点击安装按钮，Magisk会自动下载插件并安装  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-8ad5272f87f34bc42b59428f06c1d397d6d50dfb.jpg)  
    安装成功后，会有一个提示按钮，暂时可以先不用重启，等Riru和Riru-Lsposed两个都装完再一起重启。
    
    ### 1.2 打开Lsposed
    
    安装成功后，桌面会有Lsposed的图标  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1433b81f76f00a6f3e3704023aaf836390783f10.jpg)  
    点击打开显示已激活，则安装成功。  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-db1f808358f2f44e3108c1d37fbcf9c11b62f0df.jpg)  
    Lsposed内置了仓库，也就是插件市场，可以下载安装各种xp框架模块。  
    至此，Lsposed框架安装成功。
    
    2.如何过系统、app的root检测
    ------------------
    
    ### 2.1 过系统和普通app的root检测
    
    Magisk自身提供了MagiskHide的功能，点击进入MagiskHide中，勾选的应用即为被隐藏root的应用。  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-efa342d0f981cd164d9040746f6256e464308a25.jpg)  
    就我个人习惯来说，为了以防万一，我都会把看起来和root检测有关的app都对其隐藏root。下图为我个人的隐藏列表。  
    ![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-46220b135537ca218675e938a71a516a16d8ba83.jpg)

### 2.2 过SafetyNet及谷歌认证

 如果你的设备没有通过谷歌GTS测试，也就意味它没有进行SafetyNet验证，直接导致的问题就是无法出厂预装GMS；后期，即使用户后期自己手动安装了GMS，但是仍然无法下载一些应用。比如Netflix、一些游戏、应用等。它们会告诉你，**您的设备与此版本不兼容**。  
 SafetyNet Attestation API 提供采用**加密签名**的证明，用于评估设备的完整性。为了创建证明，**该 API 会检查设备的软件和硬件环境**，以查找是否存在完整性问题，并将相应数据与已获批 Android 设备的参考数据进行比较。生成的证明会绑定到调用方应用提供的 Nonce。该证明还包含生成时间戳以及发起请求的应用的元数据。

这些情况下，你的设备将无法通过SafetyNet验证检查：  
（1）你的手机在出厂时并未通过谷歌的GTS验证  
（2）你的手机通过了谷歌GTS验证，但是你解除了BootLoader锁  
（3）开启了root  
（4）刷入了非官方ROM  
（5）其他一些情况（Xposed、太极等）  
单纯的Magisk Hide无法绕过SafetyNet的检测，此时需要用到额外的Magisk插件

#### 强制小米设备通过谷歌CTS测试.

<https://github.com/yanbuyu/XiaomiCTSPass>  
此前的**小米推荐机型列表也是来源于这个项目**，就是在root后能够强制通过SafetyNet的机型、  
该项目中还提到  
若系统是2021年8月份后的MIUI版本，XiaomiCTSPass模块必须搭配Universal SafetyNet Fix模块使用  
<https://github.com/kdrag0n/safetynet-fix>

也就是说，我们还需要将额外的两个模块刷入到Magisk中，才能真正隐藏掉root的痕迹，无论是系统或是谷歌框架的校验。

这两个模块都是Magisk模块，下载后复制到手机中，通过从本地安装按钮安装模块即可。  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-30c65144e82d4f01ba9fea9df45c27da14c1fefc.jpg)  
真正的隐藏root  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-0be01cd60ad703bbd6f0f4b8d223ae77ca1c10de.jpg)  
**注意：通过SafetyNet的前提是开启了谷歌框架，且手机能够连上谷歌的服务器。**

3.如何强制抓取app的https包
------------------

我想这一章节才是你们真正的关注点吧

### 3.1 ssl抓包

 依照传统的流程，通过Fiddler、BurpSuit生成一个ssl证书，然后再把证书装到手机里，打开浏览器准备美滋滋的抓包，然后手机浏览器就提示“你这证书有问题啊”，打开app抓包发现，https还是https，它压根就没用你装的证书。这确实是一件让人头疼的事情。

#### 3.1.1 浏览器的问题出在哪了呢？

 安卓的证书被分为了“用户证书”和“系统证书”，用户证书是不被信任的，所以浏览器使用了我们自己装的证书然后报错了，因为证书不被信任。那么是不是通过root权限将证书作为系统证书安装就好了呢？可以解决一部分问题，但不能完全解决，这就要引出一个重要的东西——**SSL-Pinning**

### 3.2 SSL-Pinning抓包

#### 3.2.1 问题的成因

SSL-Pinning相关的文档可以参加安卓的开发文档https://developer.android.google.cn/training/articles/security-ssl?hl=zh\_cn

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-bdaae6a84400a2954ed3c079c675956de6d6165e.png)  
 基于证书固定说的就是SSL-Pinning，其问题在于一个app在开发的时候就已经写好了使用哪几个ssl证书作为加密使用，对于其它的证书根本就是不屑一顾。所以SSL-Pinning的问题没有办法单纯从证书的问题上解决。

#### 3.2.2 那么如何解决呢？

**直接注入Http请求库**!  
绝大多数的安卓app都使用了okhttp作为http（https）请求的库，就如同绝大多数服务器使用了Log4j2作为日志库一样。我们不管什么证书的事情，之前在app通过okhttp库发起请求时做手脚就好了。  
所以，我们需要用到一个xposed插件——TrustMeAlready  
<https://github.com/ViRb3/TrustMeAlready/releases>  
我们只需要把他安装  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-803c84a08331dec3886918e9183ae2c87bdc3c0f.jpg)  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-26f8d7a06e9b3366e3290305622072f524b033ec.jpg)  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a04fc5490175e8482f37f50938b14478ef775871.jpg)  
然后配个wifi代理，ssl证书都不用装，就快乐了。