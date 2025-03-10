0x01 前言
=======

某次项目中碰见了ueditor编辑器的net版本，存在任意文件上传的漏洞，也同样和其他版本一样存在ssrf的问题。  
但是经过测试后发现，ueditor因为魔改过无法识别到版本信息，且回包为空不返回图片地址，还删除了listimage的action，导致任意文件上传无法突破。  
最后使用了比较骚的姿势，利用get型的盲ssrf打内网的solr的log4j成功反弹shell。  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-74dc2875a298edb8921d13e12644eaf9a1ca83b2.png)

0x02 原理
=======

下载源码包分析，搜索版本号可以发现在\\gbk-php\\下面存在一个js文件中含有ueditor的版本信息，我们可以通过这个来判断是否存在一些漏洞，web中对应的链接便是 net\\ueditor.all.js 文件查看即可看见UE.version，值得注意的是只有显示大版本号。  
存在漏洞版本：  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-7c95047a1ee2de3abb11868b7fe095904852ff69.png)

```php
net  =1.3.6 || =1.5.0  || <=1.4.3 存在任意文件上传，也存在下面的漏洞
php  <=1.4.3 存在盲ssrf、存在xml上传导致xss漏洞
jsp    <=1.4.3 存在盲ssrf、存在xml上传导致xss漏洞
```

我们看已修复ssrf漏洞的版本，在抓取源的时候，会先去判断IP，进行过滤  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-a091f0571f39f3bae41e0fa676d892dfd4e796e3.png)  
但是在小于1.4.3版本的时候，是没有进行过滤的，进而造成了ssrf漏洞。  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-e20973a8f59c02795919b74170a201464fb8e912.png)

0x03 环境搭建
=========

使用windows2016开启iis环境，在目录中拖入代码，文章篇幅有限，这里就不多赘述了，自行百度  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-c6e7ebc0ee03b46673b61153a4d57c373ba541cb.png)  
右键文件夹转换为应用程序即可  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-d78632ef2579b442d0907efd9c73a2592b9fcbb7.png)  
下面建立一个图片马让ueditor抓取  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-0ea6d37bc92efee85e359fd64aa4dd0321a3134a.jpg)

感觉这里网上的文章大多都有错误，实测是建一个图片马就ok了，文件名也不用名称成乱七八糟的 1.gif?.ashx，因为url解析中?后面是get传参

也就是说这个漏洞根本不需要出网直接往服务器传一个图片马就ok了。（为什么需要图片马是因为会对图片进行校验）  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-19eadf2ddf0043ab66b1437b7c42cd57c611429d.png)

命名成1.gif?.ashx，利用python起web服务是打不成功的。路由会404  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-2b3348ef04f718b8f235a7b4b1ccd3a7da0caf26.png)

0x04 任意文件上传 bypass waf
======================

T00ls上的利用手段，抄了一下  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-f5f8d864b91126de7310bf34fdb8ea5f1bcda999.png)

0x05 骚思路 Ueditor SSRF+内网solr log4j 反弹shell
==========================================

想必各位有经验的师傅，这个编辑器在hvv的目标中出现的次数想必是数不胜数了，而且这个ssrf漏洞 只在 1.4.3.1 1.4.3.2 1.4.3.3 的小版本的更新了修复。  
仔细查阅代码发现 net 版本 1.5.0版本 是没有修复 ssrf漏洞的，而其他版本均做了修复  
1.5.0版本的net依旧存在ssrf  
但是一直是一个鸡肋的ssrf漏洞，该漏洞不支持302跳转，且php版本检测http开头  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-73769eb013c89a10c14bd3f05cf054af92b74bcb.png)  
这时又想到了一个比较骚的思路 探测内网中存在log4j影响的apache服务的默认端口  
运用于了实战中 成功反弹了shell

```php
受log4j漏洞影响的apache服务如下
Apche OFBiz
影响版本
OFBiz < v18.12.03

Apache Solr
影响版本
v7.4.0 <= Solr <= v7.7.3v8.0.0 <= Solr < v8.11.1

Apache Druid
Apache JSPWiki
影响版本
JSPWiki = V2.11.0

Apache Filnk
影响版本
四个系列：< v1.14.2, < v1.13.5, < v1.12.7, < v1.11.6

Apache SkyWalking
影响版本
SkyWalking < v8.9.1

poc就不放这里了，又兴趣的可以看大神的公众号
https://mp.weixin.qq.com/s?src=11&timestamp=1641703846&ver=3547&signature=2lMGQil4y52dVorugQJChxXYc3RU4yBzhxroqI8MTNQ5T8EDbYRYjDVcltsPEG6eDzM49*eWrOB6pq3wtKUgauIvhUcqzjiUkxKKvZNCTbJSpdVd2KRz-MUg*if19OWN&new=1
```

0x06 利用思路
=========

ueditor探测 ip+端口 存在即返回200 不存在 即 返回500  
例如：  
source\[\]=[http://10.10.10.1:80](http://10.10.10.1) 200  
source\[\]=<http://10.10.10.1:81> 500

使用bp可以轻易的进行端口扫描  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-691ac3888f443d4417a96528bfb18585171360d1.png)

探测到内网存在 solr的默认端口，直接一个poc打上去，成功反弹shell  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-0f1a97a869e440e6a89c7e8a5f5198eea2cc3334.png)  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-ae331cd266d54550f841c29e5bc12c2e66a316ed.png)  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-8ee4c162122181594d76c69c85e43bfd75d5d866.png)

0x07 笔者的一些失误记录
==============

这边一直犯了一个错误（痛失shell），POST请求中需要有content-type字段才可以被服务器接收到 post数据，之前都是用bp抓的服务器get请求，手动改方法导致，老是报错 {"state":"参数错误：没有指定抓取源"}

可以使用hackbar来快速发包测试，下面的catchimage就是一个正常的action，默认存在回显  
![](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-c7f704436c4fba51ddc5704c8ddbc8a22e4cf1d6.png)