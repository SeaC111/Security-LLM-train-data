前言
--

这个POC来源于一个很强很努力的学弟，当时我手上的项目正需要去挖很多的洞，刚好当时学弟在挖洞，于是乎就有了这个poc并且给了我，但是由于意外情况当时没来的及去用这个poc，并且学弟直接反馈给了官方，所以也就搁置在了我的知识库当中，看了一下是个前台RCE，所以得空好好分析一下。

前置
--

在开始之前先来看了一遍xunruicms的二开文档，里面有挺多对于审计很有帮助的信息，这里主要是程序路由模式的格式，这里对控制器做了一些说明，可以看到通过`/index.php?s=xxx&c=xxx&m=xxx`的路由形式可以访问到具体的模块下的文件中的某个方法，并且还可以传入参数值

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-89878384cbb3c155ac4146f0341c9ecb27cf47da.png)

漏洞分析&amp;&amp;复现
----------------

根据poc确定到漏洞点，逐步来分析  
`dayrui\Core\Controllers\Api\Api.php#248`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-bf4c9dfa16f2b454712d1122b9653427c9f057a3.png)

这里的三个参数都是可控的，通过GET方法进行传参赋值，这里调用`dr_safe_filename`方法对传入的参数进行处理，跟进该方法  
`dayrui\Fcms\Core\Helper.php#2950`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d9b1b138ab5a4f331fdc1f433d0a1b5f0f92bbe2.png)

该方法对一些特殊字符进行了正则替换，可以想到这里是为了防止目录穿越调用其他目录下的文件，那么我们能够调用的也就只有当前文件夹下的文件了  
接着往下看

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d836bec112226c16e56e6781b75e4850ff0a4605.png)

调用了`Service`类中的`V`方法，跟进，可以看到返回的实例化的`View`类对象

```php
 public static function V() {
        if (!is_object(static::$view)) {
            static::$view = new \Phpcmf\View();
        }
        return static::$view;
    }
```

之后会调用`View`类中的`assign`方法，该方法对传入的数据进行处理，如果`$key`是数组，那么就将数组重新存入到`$this->_options`变量当中，如果不是那就将`$key`和传入的`$value`存入数组；这里获取到的参数实际上是GET传入的，也就是说该数组的值也是我们可控的

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-e028ed858a08a6a535c0b65e3e5fe15ab2eca90a.png)

接着调用了`ob_start`方法，打开输出控制缓冲，后面相应的要输出的内容被存在了缓冲区当中,写了个小`demo`加深理解；如下，开启输出控制缓冲之后，利用`ob_get_contents`来获取缓冲区存储的内容，之后调用`ob_end_clean`方法来清空缓冲区当中的内容，所以最后只输出了一次，如果未清空缓冲区，那么最后缓冲区的内容同样会被输出一遍，就会输出两次

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-47685bc44f765e9a11cfacf99911f215eac00d62.png)

接着来看开启缓冲区之后调用的`View`类中的`display`方法

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d02c4cad6ad9d4b2a348fb101dedc32ac60f35c0.png)

这里前面的几个`if`判断并不会直接返回，并且这几个都是可控的，不影响，直接看到断点处，`extract`这里，会进行变量覆盖，并且由于`extract_type`写定的是`EXTR_OVERWRITE`,看一下php手册中的说明

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-9e2e14879a9775a72b3c0f1225c18b9f39bddc70.png)

这里的1相当于标识符`EXTR_ORERWRITE`，可以看到数组中的键值为`joker`的`value`值被覆盖了，标识符2相当于`EXTR_PREFIX_SAME`，有冲突的话会加上前缀`wx`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-60a0a957ced129a8e314aeca452276973da68511.png)

回到代码，这里既然存在变量覆盖，那是可能利用模板文件内容可控来RCE的，接着往下看

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-b139a93271b89bc61a31d4342a7eec221677585d.png)

`$this->_filename`是最开始可控点之一`$name`，所以这里也是可控的，`$phpcmf_dir`默认为空，虽然前面调用`display`方法只传入一个参数，看似不可控，实则前面说到的`assign`方法处理时，传入的其实都是GET方法获取的东西，我们也就可以传入`$phpcmf_dir`来使得这个点可控，跟进`get_file_name`方法  
这个方法中的判断条件和属性值特别多，仔细梳理了一番，最后确定几种情况，当$dir传入空值的时候，看到红框的地方

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-597779cfd76b4b2103d3b16023a4c1bbf004dc03.png)

回溯几个变量，`TPLPATH=template/,$this->_tname=pc`

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ee9bec1719f4c1da58eaf50dca27688020300c2a.png)

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-2dad00f0a342a54a3db4d774866fbfdea4cf33d8.png)

所以没传入$dir参数的时候会找下图中的文件  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-558fa4347bde0fe55cba31cb25b11d1942be61db.png)

传入$dir=admin时，就会在下图中寻找模板文件  
![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-820920df3956e158c6809a918bb6124e6eb738dc.png)  
所以这里只要`$name-》$phpcmf_dir-》dir`,也就间接控制了返回调用的模板文件了，相当于可以指定某个模板文件进行调用  
这里的模板文件还是静态文件，没有渲染过，所以需要找对模板文件进行解析的方法，经过对模板文件的渲染定位到了`api_related.html`文件，具体原因下面会提到  
`dayrui\Fcms\Core\View.php#676`,`list_tag`会进行解析，并且在这个方法当中存在`call_user_func_array`函数，那么有几率能够命令执行

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d352474d8ca65d510e8a98d3ef2ac724764ebc7b.png)

这里需要进行一下参数的控制，来看到方法中一开始的处理，`$_params`是调用方法处理时传入的参数，这里填上面留下的坑，渲染`api_related.html`文件被解析时调用了如下方法

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5b60e68ee7bbae7fd359cdceea243ea4c6c8b7b7.png)

所以传入`list_tag`方法的参数就是括号里的这些，接着`list_tag`方法中调用`explode`进行了处理，以空格为分隔符存入了数组

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-7376e8aef807d8ee01925e33d49813a2b21369f1.png)

接着又遍历数组，`$var`为取到的键名，`$val`为取到的键值，按照前面解析传入的值，`action=module`，而只有让`action=function`，才会在处理之后让`$system[action] = function`，才能够调用到`call_user_func_array`方法达到命令执行；这里并且写明了只能出现一次，但是看似只能出现一次，但是实际上并不是如此，这里可以利用前面分析的变量覆盖，配合空格，再传入`action=function`;这里还可以写入最后的利用点所需要的`$param`参数

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-595fdf9384dd016fad70bf0c47f0c136c6f18fcf.png)

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-13657e89ac80c6b291d402d8ed02896fd0cbfca5.png)

回看如下代码，这里不能有缓存，而一开始的缓存并没有开启，这可以在上面配合空格传入`name=命令执行函数像是system`,那么就有了`$param['name'] = system`，结合下面遍历的逻辑，如果传入`param0=whoami`,经过处理之后会得到`$p[0] = whoami`,那么最后就能够顺利进行命令执行了

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-32869dae80159c5ff4f5a95b5336eed62b7509a7.png)

经过上面的分析，poc也就出来了

```php
?s=api&c=api&m=template&app=admin&name=api_related.html&phpcmf_dir=admin&mid=%20action=function%20name=system%20param0=calc.exe
```

由于前面提到的开启缓冲区，其他的结果会配合json进行返回，这里选择弹一个计算器吧

![](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-0d741b369db17f6db35db27ff4296918cb0b6331.png)

PS
--

该漏洞在当天就修复了，修复措施可以去gitee官网查看。