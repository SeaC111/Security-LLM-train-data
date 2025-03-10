安卓机root抓包及绕过检测教程（上）
===================

1.搞机相关科普与机型的选购
--------------

### 1.1 机型选购指南

 本套教程以红米K20 Pro 系统版本MIUI 12.5.5稳定版，RFKCNXM为例，其它机型或版本可能有一定出入，请具体情况具体考虑。  
 以下机型为比较推荐的机型，更加建议选择安卓版本为11的机型，因为笔者所用的设备为安卓11，在一切奇怪的问题上比较容易解决。大力推荐K20、K20 Pro，但别买K20 pro尊享版。至于这张表格的来历，在后续的章节里会提到。  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a53f085247951063d47a86794f09074816e32b43.png)

### 1.2 搞机相关科普

#### 一、安卓基本分区

##### 1、bootloader（fastboot）

 Android 系统虽然也是基于 Linux 系统的，但是由于 Android 属于嵌入式设备，并没有像 PC 那样的 BIOS 程序。 取而代之的是 Bootloader —— 系统启动加载器。  
 一旦bootloader分区出了问题，手机变砖就很难救回了。除非借由高通9008或者MTK-Flashtool之类更加底层的模式来救砖。  
 一般刷机不动这个分区。

##### 2、recovery

 用于存放recovery恢复模式的分区，刷机、root必须要动的分区。里面有一套linux内核，但并不是安卓系统里的那个，相当于一个小pe的存在。现阶段的刷机、root基本都要用第三方rec来覆盖官方rec。

##### 3、boot

 引导分区，虽然说是引导，但实际是在bootloader之后，与recovery同级的启动顺序。里面装了安卓的linux内核相关的东西，magisk就是修改了这部分程序，实现的root权限的获取。

##### 5、userdata（data）

 用户数据分区，被挂载到/data路径下。内置存储器（内置存储卡）实际也存在这个分区里，用户安装的apk、app的数据都在这个分区。目前主流安卓版本中data分区通过fuse进行了强制加密，密码一般都是屏锁密码，且加密的data分区未必能在recovery下成功解密，所以有时刷机需要清除整个data分区。

##### 6、cache

 缓存分区，一般用于OTA升级进入recovery前，临时放置OTA升级包以及保存OTA升级的一些临时输出文件。

#### 二、安卓系统的启动流程

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f23a29252f5e2b9ac32946994dd532c22b37d813.png)  
 一般来说，安卓系统的刷机都是在fastboot和recovery进行的，因为此时安卓系统本身还没有启动，直接无视各种权限进行操作。  
 fastboot通常又被叫做线刷模式，PC端通过fastboot程序直接向手机刷写分区镜像文件，fastboot无法进行更加细致的操作，一刷就是整个分区，而不能对某个、某些文件进行特定的修改。  
 recovery模式下可以刷入带有脚本的zip包，可以实现对系统的增量修补，修改特定文件等更细致、强大的功能，但问题在于官方的recovery拥有签名校验，非官方的zip包无法通过官方的recovery刷入，于是我们需要刷入第三方的recovery来运行非官方签名的刷机包的刷入。于是我们就需要在fastboot下刷入第三方的recovery。  
 但是，fastboot的刷入也有限制，未解锁的fastboot（也就是bootloader锁、bl锁）不允许刷入非官方签名过的img镜像，所以我们需要对bootloader进行解锁，国内允许解锁bl锁的厂商就只有小米了，所以像搞机得用小米的机子。  
 通过解锁bootloader来刷入第三方的recovery，再通过第三方的recovery来刷入第三方的刷机包（如magisk或是其它的系统）达到直接或间接修改boot、system等分区的文件，这就是刷机的真谛。

 其中Magisk作用于boot.img的阶段。Magisk的安装实际是将boot分区导出然后进行patche后再重新刷入。magisk通过直接修改boot.img中的linux内核相关文件实现了root权限的获取，也是由于其在很靠前的位置，可以通过在system分区挂载时，额外挂载分区、文件、目录来实现对系统文件的替换、修改而不修改原始文件。

 Riru的Lsposed框架作用Zygote阶段，通过注入Zygote来实现对其创建进程的修改。

**How Riru works?**  
● How to inject into the zygote process?Before v22.0, we use the method of replacing a system library (libmemtrack) that will be loaded by zygote. However, it seems to cause some weird problems. Maybe because libmemtrack is used by something else.  
● Then we found a super easy way, the "native bridge" (ro.dalvik.vm.native.bridge). The specific "so" file will be automatically "dlopen-ed" and "dlclose-ed" by the system. This way is from here.

**参考文献**  
<https://source.android.com/devices/bootloader/system-as-roothttps://github.com/RikkaApps/Riruhttps://forum.xda-developers.com/t/magisk-the-magic-mask-for-android.3473445/>

### 1.3 如何开启开发者模式

本篇教程为操作指南性质，为后续部分教程的依赖。

1、进入“设置”  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-566abe3a0c839d2103dd86e072bc5a783d8b341c.jpg)  
2、进入“我的设备”  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-6c3d7d899df20b1f6ea947996d7f29bcfdcfad1a.jpg)  
3、进入“全部参数”  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-eab2e6cbe58b89f108e3f4de47a08a3174bafc95.jpg)  
4、多次点击“MIUI版本”  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-681bf45c5e477bd7e29d3ffbbfe74a22ea69eb78.jpg)  
5、出现“您已处于开发者模式，无需进行此操作”字样，至此开发者模式入口已打开  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-b0b7e3d187a67b834fa30f95e997d1e3de2f6697.jpg)  
6、那么开发者模式的入口在哪？  
在设置中的“更多选项”  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-64a4b2c5d4ce706a9c7564ee427d9076fcd6703f.jpg)  
7、最下方的开发者选项  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a638051434564d774cec1a0532fea1166e8e46be.jpg)  
8、打开开发者选项  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5d04084ab47ece68e619df7460e15c67ef78c1f2.jpg)  
 至此开发者模式已打开，开发者模式中有很多功能，请按需开启。本篇教程作为操作指南使用，在需要的场景中会有引用提示。

### 1.4 刷机必备工具包

#### 1.4.1、小米解锁工具包

用于解锁小米手机的bootloader的工具包，如果你的手机有bl锁就需要进行解锁。  
bl锁解锁教程见“小米、红米手机解锁bootloader”

下载地址：<http://www.miui.com/unlock/download.html>

#### 1.4.2、fastboot与adb工具包

fastboot是用于刷入第三方recovery时必要的工具，其实是对应安卓fastboot模式下的，pc操作的客户端软件。  
adb是安卓调试桥，在需要对应用进行调试时使用。

下载地址：<https://developer.android.com/studio/releases/platform-tools#downloads>

2.如何解锁bootloader
----------------

### 2.1 小米、红米手机解锁bootloader

#### 2.1.1 解锁Bootloader的步骤

1. 在需要解锁的设备中登录已经具备解锁权限的小米账号，并进入“设置 -&gt; 开发者选项 -&gt; 设备解锁状态”中绑定账号和设备；
2. 绑定成功后，手动进入Bootloader模式（关机后，同时按住开机键和音量下键）；
3. 下载手机解锁工具(解锁工具官网)，在PC端的小米解锁工具中，登录相同的小米账号，并通过USB连接手机；
4. 点击PC端解锁工具的“解锁”按钮，根据提示信息等待指定时间后再次尝试或者立即解锁； #### 2.1.2 解锁Bootloader过程中可能遇到的问题
    
    Q：解锁工具提示“账号设备不一致”是怎么回事？  
    A：这是在解锁过程中没有通过账号与设备验证，解决办法是先将手机升级到最新的稳定版或者从稳定版卡刷到最新的开发版，在待解锁的设备和解锁工具上要登陆同一个账号，并进入“设置 -&gt; 开发者选项 -&gt; 设备解锁状态”中绑定账号和设备。

Q：解锁工具提示“无法获取手机信息”是怎么回事？  
A：这种情况一般是电脑上的设备驱动没有装好，可以尝试重插USB线或者换个USB接口或者换根USB线来等待电脑慢慢安装驱动，或在工具右上角驱动安装模块中主动安装驱动。

Q：解锁失败显示“账号与设备的绑定时间太短，xxx个小时后再解锁”  
A：在售的新机型一般需要等待，用户账号安全评分较低的需要等待，等待时间目前是7天起，如果本年度解锁手机数超过2台，等待时间会相应增长。

Q：解锁失败显示“此账号本月解锁次数达到上限”  
A：一个小米账号每月限制解锁一台设备。

Q：解锁失败显示“此账号本年累计解锁次数已达上限”  
A：一个小米账号每年限制解锁4台不同设备。

Q：解锁失败显示“账号权限不足或者账号受限”  
A：账号存在安全风险，无法处理解锁操作，建议更换账号。

Q：解锁失败显示“未知错误-1”  
A：网络异常，请更换时间段或更换网络进行解锁。

3.如何刷入第三方recovery
-----------------

 由于官方的recovery模式功能有限，且不允许刷入非官方签名的刷机包，于是我们需要刷入第三方recovery。  
 目前全球最大的主流第三方recovery是TWRP项目，其官网为https://twrp.me/

### 3.1 下载TWRP Recovery

点击Devices按钮，进入设备列表  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1f8207d85fae365eff93f7d6f37679824793a605.png)  
向下滑动找到自己的机型  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ad65c3104ea706868ea7fc2ddb212c815cb58d87.png)  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-bf076b9a4f153749ffa46d913e15717ce1f6d304.png)  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-0dcc3a142009d97f7a49a20aa888ebe0382830dc.png)  
找个最新的下载下来  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4734894726a253a5cf41987dda23918b6063a758.png)  
下载后得到twrp-3.6.0\_9-0-raphael.img  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-df8967ac7b45a7044dc668b49444346560090fbb.png)  
至此就以及准备好了刷入用的recovery文件了

### 3.2 刷入recovery

注：需要用到的fastboot.exe参见https://www.yuque.com/daerwendexiongdidaerka/ny2za8/itcfvi

手动进入Bootloader模式（关机后，同时按住开机键和音量下键），通过数据线连接到电脑。  
通过cmd或powershell打开fastboot工具，  
fastboot.exe devices用来查看已连接的设备  
**fastboot.exe flash recovery recovery.img的命令刷入recovery**  
其命令原型为 fastboot flash \[要刷入的分区\] \[要刷入的镜像文件\]  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d00ce2875672c9117036c5126684d7d0cd7a42ca.png)  
此时twrp已被刷入，但别急着重启，直接重启系统会导致系统将recovery还原，我们需要直接进入到twrp中。  
先按下音量加和电源键，在手机屏幕熄灭时松开电源键，手机显示小米logo并震动时松开音量键，等待片刻即可进入recovery。

### 3.3 进入Recovery

**注：进入recovery的按键组合是音量加与电源键长按**  
 首次进入twrp时系统会提示是否修改系统分区，**先去点击Change Language**，将语言改成中文。然后将**蓝色的条滑动到右侧**，表示允许修改。  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-07d8abb44131a0b14870edb88bc1e1828ae50a04.png)  
进入主界面  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-091d4130455b602ab89e40b0e5ddcbe7acfc036f.png)  
至此，TWRP Recovery刷入完成并成功进入。

4.如何刷入magisk
------------

### 4.1 下载Magisk

Magisk的官方下载连接在：<https://github.com/topjohnwu/Magisk>  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-8ad618f1e9caa7653d534466a5d637e80df9a13d.png)  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-2345708501f9716205f68ed22d47cbb0e617895d.png)  
下载后我们得到了Magisk-v23.0.apk文件  
**注意：Magisk的apk安装包经过特殊处理，扩展名改为zip即为recovery下使用的刷机包，apk即为安卓使用的magisk管理app。**

### 4.2 刷入Magisk

**警告：以下操作将会清空手机内所有用户数据**  
将手机进入到TWRP Recovery下，通过数据线连接电脑，电脑此时会弹出一个MTP设备，我们会发现文件夹和文件都是乱码的，这是因为高版本安卓下整个Data分区使用了fuse作为强制加密，Recovery下无法将其解密。但是我们必须要通过某种渠道，将刷机包放在Recovery能够访问的路径中，此时有两种选择，OTG插U盘读取刷机包，或者直接格式化整个Data分区，此处我们使用后者。  
点击TWRP的清除按钮、选择格式化Data分区  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-367bd66442f07e1f6b925c3b014e9ab48a144a79.png)  
 成功后点击重启按钮中的Recovery，手机将重启到TWRP Recovery中，此时电脑中的MTP设备就会变成一个空设备，这时我们在将Magisk-v23.0.apk改名为Magisk-v23.0.zip复制到手机中，点击安装按钮刷入该刷机包。我们需要到/**sdcard路径里去找这个包**  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-08a831e916947710a842a7985b5e69ad58efcb39.png)  
 选中后，滑动确认刷入，当出现done字样且控制台不再有输出，点击reboot system重启系统即可。  
此时magisk应该以及成功装入到boot分区了  
**注意：重启系统后，data分区将被安卓系统本身再次格式化、并进行强制加密。如果中间出了什么岔子，需要再到TWRP中刷入刷机包，需要再次手动格式化Data分区。**

### 4.3 打开Magisk

进入系统后会发现Magisk还需要进一步安装，此时需要将Magisk-v23.0.apk安装到手机中，之后Magisk会提示系统需要重启之类的，等待系统重启再次进入系统。  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-7ff7bcd39999d416d04ca984b568a1598ce467a3.jpg)  
 当此处能够正常显示版本号时，Magisk则安装成功了，当然你也可以去下载几个需要root权限的软件试试，比如MT文件管理器、RE文件浏览器之类的，去看看比如/data之类没有root访问不了的路径。  
 至此，Magisk安装成功。