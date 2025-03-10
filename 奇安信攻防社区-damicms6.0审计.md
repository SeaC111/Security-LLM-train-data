### 环境

damicms，为了兼容更多的用户，所以建议使用php5来进行搭建，最新的版本支持php7，用php7时能够进入到安装界面完成安装，但是界面无法显示也没有报错信息，需要更换php版本

### 远程代码执行

根据github上的issue，确定了代码触发点为整个cms中使用的模板文件，为了便于触发issue中用的是`head.html`,在任意页面都能够触发写入的恶意代码。  
抓包确认调用的方法

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-ced6da3220d87359f3e17066ccd00edbec07b90e.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-ced6da3220d87359f3e17066ccd00edbec07b90e.png)

`Admin/Lib/Action/TplAction.class.php#89`

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-34e1f4b645616d70c0df32d19a5c862c4f164a71.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-34e1f4b645616d70c0df32d19a5c862c4f164a71.png)

通过`stripos`来限制文件的后缀，该函数不区分大小写，所以大小写没办法绕过；之后用`htmlspecialchars_decode`将文件内容中特殊的HTML实体转换回普通字符；`stripslashes`反引用字符串，会去除转义反斜杠；接着调用`write_file`方法，跟进  
`Admin/Common/common.php#45`,清除了文件的缓存，之后就会将内容写入到文件当中，并未对内容进行任何的校验

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-b27e11c0e4eb3e71c87f043b15a0a9783694260f.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-b27e11c0e4eb3e71c87f043b15a0a9783694260f.png)

常规思路来看，这里写入的只是html文件，审过的cms这一类head.html文件只能够解析html标签，最多就是能够构造一个存储型XSS罢了，这里居然能够解析php，那就需要去找一下为什么。全局搜索来看一下有包含或者其他方式用到head.html的地方  
`Web/Lib/Action/PublicAction.class.php#70`

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-7772f2cbcddd06309c12a2b4367a3a92c3ba9973.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-7772f2cbcddd06309c12a2b4367a3a92c3ba9973.png)

跟进调用`assign`类中的方法

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-d833c2fc6479436943dea8670d47aecabc0e96ff.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-d833c2fc6479436943dea8670d47aecabc0e96ff.png)

反溯`$this->view`变量  
[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0e9afefd580ad7fd981bf776a9421d081689e1fa.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0e9afefd580ad7fd981bf776a9421d081689e1fa.png)

这里会调用`Think`类中的`instance`方法实例化`View`类,然后赋值给`$view`参数，那么上面调用的也就是`View`视图类中的`assign`方法，跟进，发现只是对模板变量进行了赋值

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0b1885a5ad13c82e9efb732bf1f22cbec54f0f96.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0b1885a5ad13c82e9efb732bf1f22cbec54f0f96.png)

在视图类中有一个`display`方法用于加载模板和进行页面的输出，这个方法应该才是写入的恶意代码的触发点

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-f16b067b4807586049366b7308ad1e96ef27a20b.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-f16b067b4807586049366b7308ad1e96ef27a20b.png)

直接跟进`display`方法中调用到的类中`fetch`方法，根据注释会进行模板解析并且获取其中的内容

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0b5e3048e76646f37b4520cdc588f136a6f75359.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-0b5e3048e76646f37b4520cdc588f136a6f75359.png)

这里根据注释会使用PHP原生模板，`$content`不为空，根据三元运算符，会进行内容的拼接，猜测最终的触发点就是这里，通过`eval`会使得写入的恶意代码能够被执行。

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-d06d510a6335dd94c505a241540cb7f827318d2e.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-d06d510a6335dd94c505a241540cb7f827318d2e.png)  
[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-faa0c5f9bc4ed399aaede981d17f639deefccc68.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-faa0c5f9bc4ed399aaede981d17f639deefccc68.png)

### 任意文件读取

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-70fc03d770d5cd740b992d488852ef5b4c392a21.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-70fc03d770d5cd740b992d488852ef5b4c392a21.png)

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-b16169cda542423ff348093f83df262393c497d1.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-b16169cda542423ff348093f83df262393c497d1.png)

传入的id参数，去除首尾空白字符，再会进行一个替换，之后会调用`dami_url_replace`方法，跟进  
`Admin/Common/common.php#94`  
根据issue，跟进具体的调用方法  
`Admin/Lib/Action/TplAction.class.php#76`

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-f1862d41e39f10007a9ee04abf3ab5200a3551f6.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-f1862d41e39f10007a9ee04abf3ab5200a3551f6.png)  
这里上面调用该方法的时候只有传入经过处理的id参数，所以$order默认为`asc`，这里会对获取到的id参数值进行替换，所以给定的poc中会那么构造文件路径

```php
|=》/
@=》=
#=》&
```

接着调用read\_file方法，直接读取文件内容,再后面的代码部分就不用去看了，最终肯定是调用视图类中的方法进行文件内容的打印输出

```php
function read_file($11)
{
    return @file_get_contents($11);
}
```

poc

```php
?s=Tpl/Add&id=C:|windows|win.ini
```

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-a2a22b796159b24f13dffa1949c15cfbf708daa5.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-a2a22b796159b24f13dffa1949c15cfbf708daa5.png)

### 任意文件删除

`Admin/Lib/Action/TplAction.class.php#119`

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-2fdbb165cb7f6d060f779ac496d7bc9c4565940e.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-2fdbb165cb7f6d060f779ac496d7bc9c4565940e.png)

有任意文件读取一般都会有任意文件删除漏洞的存在，可以看到代码部分，前面的任意文件读取中已经分析过了是怎么处理获取到的`$id`参数的，所以这里只需要指定需要删除的文件路径就可以了，这里我在根目录下创建了一个`1.php`文件，下面就进行一下删除操作

[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-14365552a7e9c655ac36aa2beb9ebdbb816de3de.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-14365552a7e9c655ac36aa2beb9ebdbb816de3de.png)

### 总述

这几个漏洞都需要后台权限，可能会觉得比较鸡肋，但是这个cms实际上它的管理员权限是能够进行获取的，默认的 damiCMS 管理员用户的 cookie：  
[![](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-5d5fdfb58dd2e09bc66a5933b59ec69638bda344.png)](https://shs3.b.qianxin.com/attack_forum/2021/11/attach-5d5fdfb58dd2e09bc66a5933b59ec69638bda344.png)  
在`BkGOp9578O_1535522538 = czoxOiIxIjs％3D`时，管理员的登录会被更新  
`BkGOp9578O_`是COOKIE\_PREFIX默认情况下，`1533522538`很明显是时间戳，该cookie会保持 3小时登录状态，这意味着我们可以得到管理员用户的cookie，只需枚举最多 10800 次即可获得权限，可以在这个基础上来进行上面几个漏洞的利用。  
最新版的在这里：<https://www.damicms.com/Down>\# ，v7.0.0,联系了技术支持最新版已经修复漏洞了。