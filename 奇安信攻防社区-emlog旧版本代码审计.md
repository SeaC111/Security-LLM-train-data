这个emlog CMS是一个使用范围听广泛的博客系统，并且更新的频率还挺高的，近期才完成了最新的pro版本的迭代，本文中设涉及的漏洞都在github的issue列表当中，并且CMS作者均已知悉并且做了修复，本文只是进行了对应漏洞的分析

### File delete

后台插件管理处  
`admin/plugin.php`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-fb97d230083f1debea042d91bed71fba2e381d03.png)

首先调用了`LoginAuth`类中的`checkToken`方法进行`token`校验，该方法是为了防御CSRF攻击，`Token`出错直接抛出相应的异常；下面会调用`Plugin_Model`类中的`inactivePlugin`方法，跟进

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-cf6b13b1828e2fc9b85a66362ff166aac76ad6f6.png)

开头就调用了`Option`类中的`get`方法，跟进

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ef8f806e6f67a14568836415c1c57965516f4d1f.png)

传入的值为`active_plugins`，返回为空，回到`inactiveplugin`方法中，因为返回为空，`$active_plugins`的值必定为空，那么传入的`$plugin`的值就不会在数组中，就会直接返回。  
接着经过正则替换之后`$pludir`为我们传入的文件路径，调用`emDeleteFile`方法进行删除，如果是单个文件直接删除，如果是文件夹则循环遍历删除文件，并且最后删除空文件夹

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-8a47e9b25f23a92ed8bf9c9f05e2d1bfd9b8a454.png)

poc

```php
/admin/plugin.php?action=del&plugin=filepath&token=
```

### upload getshell

后台插件管理处=&gt;安装插件  
`admin\plugin.php#87`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-dd359f61a99038c410f97e313dc938dae53520cc.png)

首先对上传的压缩包文件名进行了判断，之后对数组中的`error`中进行获取判断，如果`$zipfile`变量值为空或者`error`值不为0或者临时文件名为空，直接返回报错信息；并且这里对上传的文件后缀作了限制，只能够为`zip`，通过调用`getFileSuffix`方法来获取文件后缀；接着会调用`emUnZip`方法，跟进

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-32d9d28552eb2a9bacf6ac57911c39fd55b56026.png)

传入的`$type`为`plugin`；可以看到这里还是调用的`PHP`原生类`ZipArchive`来对压缩包进行处理，这里调用了`explode`来分割获取到文件名前面部分，然后`$dir=文件名/`,最后会调用原生类的`getFromName`方法来检查是否能够获取到指定路径下的文件名，如果能够获取就能够上传成功并且解压缩。  
最后的这个条件只需要将上传压缩包中的`shell`文件名和压缩包文件名统一即可

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5ffbbf6145952ff7ebb021ad11444fbb7b30f283.png)

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-36280c1613cc2843f3422e89a6b13cab3e76c5f1.png)

其他类似的几处文件上传getshell就不一一分析了。

### SQL Injection

`admin/widgets.php#133`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-eb8f1a1668f2ba002132d111a76e61bcfdba08c6.png)

`wgnum`传参时不进行赋值，三元运算符赋值为1，`widgets`是可控点并且是漏洞点，这里会对出传入的值进行序列化；接着调用`Option`类中的`updateOption`方法，跟进

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-c7cd236b43d3db844226723ea417fa6e7f74551b.png)

实例化`Database`类，`$isSyntax`未进行传参，默认为`false`，`$value`值就是前面POST传入的参数值，这里跟进`query`方法

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ad479238fcec093d4f07b515b6d8975764828521.png)

可以看到这里会输出报错信息，并且查询语句可控，那么就可以构造报错注入语句进行SQL注入攻击  
由于序列化存储数据的原因，所有这里用or或者and来进行连接，并且要注意语句的闭合，前后都需要单引号来进行引号的闭合，最后的poc如下

```php
' or updatexml(0x3a,concat(1,(select user())),1)and'
```

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-e3906dd0318a7ec919dea98c5733f361eccb1b78.png)

### 小记

审了很多的后台的洞了，感觉在实战的时候用处不是特别大，后面会同时审计JAVA之类的CMS，并且多尝试审审前台getshell的洞。