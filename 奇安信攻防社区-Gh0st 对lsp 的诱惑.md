你看，健康上网多重要
==========

来一段介绍
-----

首先，这个东西我是无意发现的，那是一个平静的一天，我打开一个正常的网站，发现了这个让众多lsp 不会平静的病毒诱饵。

![1642555929325.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-77698103a23abcac481ede704d376d06c115b66d.png)

然后，本着无聊就简单分析一下的心态，我下载了它。PS：真的是无聊才下载的。

咳咳，为了方便过审，我重命名这个样本为test.exe。

| 文件名 | MD5 |
|---|---|
| test.exe | 21a08a30b02b395c375616936e756a1d |

扯皮结束，开始分析
---------

拿到样本还是要分析一下的。首先拿到这个样本时，通过样本名称就大致知道这个样本多半会伪装为视频或者播放器之类的，解压后，嘿，雀食。

![1642556007592.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-7559da1fd07aafa67715058094d45efdeaf600ca.png)

通过图标可以发现，这里利用一个类似视频播放器的图标进行伪装，隐藏的一点都深。

使用DIE 查看样本，发现时UPX 加壳，不多说，脱它。

![1642556025132.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-2f410109abdd015c28c02e762ea4230b398a10b8.png)

UPX 的壳可以使用UPX 官网下的程序进行脱，也可以手动脱，这个壳比较简单，这里就不详细说明怎么脱了。

![1642556046712.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-e5f5dea9b2e078ae4825d3b94ee1cc1bd33b4670.png)

脱完后，OEP 定位到0x40F80C 并dump 出文件，修复IAT 后，就可以使用IDA 进行静态分析。

首先，打开IDA 并将dump 的文件（这里叫test\_dump.exe）打开进行静态分析。

首先是可以看到，样本会打开注册表HKEY\_LOCAL\_MACHINE\\SOFTWARE\\ODBC 并在返回成功后进入接着执行后续代码。

在之后的代码中，使用了大量形如一下代码结构的花指令干扰IDA 分析

```php
call    _CxxThrowException
```

```php
mov     eax, next_eip_value
ret
```

该花指令通过先跳转到异常处理函数并在执行异常处理函数的代码后回到下一条地址处接着执行，在IDA 使用反编译时，会因此受到干扰，无法正确反编译代码。直接匹配特征去花：

```c
#include <idc.idc>

static main()
{
    auto head, op, code,i;
    head = NextHead(0x00000000, 0xFFFFFFFF);
    while ( head != BADADDR )
    {
        op = GetMnem(head);
        code = GetOpnd(head,0);
        Message("--> %8x : %s %s\n",head, op, code);
        if(code == "_CxxThrowException")
        {
            //SetColor(currEA, CIC_ITEM, 0x010187);
            for(i = 0; i<5;i++) 
            {
                PatchByte(head+i, 0x90);
            }
        }
        if(code == "eax" && op == "mov")
        {
            auto asmCode = GetDisasm(head + 0x5);
            if(asmCode == "retn")
            {
                //SetColor(currEA, CIC_ITEM, 0x010187);
                for(i = 0; i<6;i++) 
                {
                    PatchByte(head+i, 0x90);
                }

            }
        }
        head = NextHead(head, 0xFFFFFFFF);
    }

}

```

当然，这个脚本是针对这个样本随便写的，遇到类似的样本或者其他的花指令，不能使用这个去，要不然出大问题。

出来这种静态去花之外，也可以直接动态调试，只是在理解整个代码的所要完成的工作的时候，较于直接分析比较打脑壳。不过小孩子才做选择，我都要【doge】

配合动态调试和IDA 静态查看，可以看到样本后续会通过解密一段PE 数据并加载到内存后定位到函数Shellex 处接着执行。

解密PE数据

![1642556078574.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-e64f5348fdd3dbc062096a433bd5bb33f31fbad0.png)

解密算法如下：

![1642556100317.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-96e5f7f15518f0e9f72fe3196a0db86c990de3a4.png)

在解密出PE 数据后，我们dump 出一份数据到桌面并命名为dump.bin

| 文件名 | MD5 |
|---|---|
| dump.bin | 21bea12a54bc431003ebc3e7b5b7d75a |

这个文件先暂时放一边，接着分析test\_dump.exe.

在解密出dump.bin 并将其加载到内存空间后，通传入的Shellex 函数名字符串获取该函数地址并调用。

![1642556115882.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-66d8d64cf0eb8ea7b018e430b368aeb080004aad.png)

之后，在调用完成后，退出程序。

自此，test\_dump.exe 程序功能分析完成，该程序主要就是一个加载器，通过解密自身携带的被加密的PE 结构数据并在解密完成的数据加载到内存后，调用指定函数执行后续功能。

接着就分析刚刚解密获得的dump.bin 文件，同样的，将文件拖入IDA 查看，可以知道这个文件是一个DLL 文件，并且导出上面提到的Shellex 函数以及DLL 函数入口函数。入口函数未实现任何功能，功能都在Shellex 函数内进行实现，所以后面就是从Shellex 函数作为入口点开始分析。

首先进入到Shellex 函数后，程序会初始化一些字符串到指定的内存，这些字符串包括所创建的服务名称、用户界面标识服务字符串、被加密的诸如版本号和C2 地址的字符串等。

![1642556143980.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-4ebc951036af610d4a21c708898168bcfdcb4de9.png)

```php
"tLPQtLS60L230LW1vaY=" // 23.224.97.119
"pg=="
"6gkIBfkS+qY=" // Default
"tdC2pg==" // 1.0
"PhXDxphx Qiyqh"
"MeumeuDR Nevnevne Wnfwnfwn Gwog"
"SkcsDRkbsk Ctlctkctk Duldtld Umeumdum Evn"
"62970fcf3ea2b1319dd5c8e9116d7060"
"%ALLUSERSPROFILE%\\Application Data\\Storm\\update\\"
"PhxphFVX64.exe"
```

接着，创建互斥体，互斥体名称为Global\\\[module\_path\]，其实，在这里就可以大致可以确定该样本家族：Gh0st.

![1642556167319.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-7e090ddb6434f10ec3079b26dbcee71be110a2e1.png)

之后获取当前程序执行所处时间并格式化写入到注册表中供之后代码使用。

![1642556186448.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-270a784b2d0d3db62bc832808d0dd0cc92e75c81.png)

此外，在分析中还发现该RAT 程序包含多种启动方式，如注册表启动、通过命令启动、通过服务启动、通过开始菜单启动等。在RAT 重新执行中，通过检测配置信息，对启动方式进行设置。如判断命令行内是否存在字符串“-acsi”。

![1642556203307.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-67cccf331c6d8af8dddba2946e6d6d6b692aacd7.png)

通过判断是否存在对应服务以及判断服务是否正在执行进行服务启动。

![1642556222139.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-65948337f30645ab8cde506e71c05d74e5246ebb.png)

如果上面两种情况都不满足，程序便会创建对应的服务并且复制当前模块文件到update 路径下，之后结束当前程序的执行，启动所创建的服务实现程序的启动。

![1642556235040.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-9e5f179bacc9ceb3a91f42335a17b4c64f666ff4.png)

此外，样本会以多种方式实现启动，包括使用服务启动、开始菜单进行启动、注册表启动以及文件绿色启动。

![1642556252716.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-63866a66251d3112fb59ee3eb76c6cebd8dafaa8.png)

程序在运行中，还会通过匹配大量杀软进程的名称检查杀软。

![1642556265694.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-80594de0442ad0a71511646d7fbed49b0d76907c.png)

![1642556297271.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-1d6ef2532f07d5b366c4e4f4be0c4cc517feca11.png)

该程序通过解密之前设置的字符串，得到C2 地址并与之创建链接实现通信。

![1642556313692.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-f06195c2b2e93a64a2451906c5dc47b3680e967a.png)

解密函数如下：

![1642556330561.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-d810f7164021dd23c87fd31f1d0613eea502b034.png)

在获取C2 地址23.224.97.119 后与之建立连接，实现通信。

![1642556351497.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-235b7e4c67712233da36d471cd4b32e16c5657bb.png)

通过接受服务器回传指令实现，通过分析，得到指令列表大致如下：

| 指令 | 功能 |
|---|---|
| 1 | 记录计算机磁盘和文件信息 |
| A | 获取计算机服务状态和服务配置信息 |
| B | 检测指定进程是否运行 |
| C | 检测指定窗口是否存在 |
| 0x10 | 实现屏幕远程 |
| 0x27 | 运行指定程序 |
| 0x28 | 结束指定进程的执行 |
| 0x29 | 清除IE 信息 |
| 0x2A | 获取Chrome 数据 |
| 0x2B | 获取Skype 数据 |
| 0x2C | 获取Firefox 数据 |
| 0x2D | 获取360 浏览器数据 |
| 0x2E | 获取QQ 浏览器数据 |
| 0x2F | 获取搜狗浏览器数据 |
| 0x30 | 向服务器传输获取的浏览器数据 |
| 0x3F | 获取键盘操作记录 |
| 0x45 | 向服务器传输实时录音数据 |
| 0x48 | 获取计算机系统信息 |
| 0x5A | 显示消息弹窗 |
| 0x5D | 远程退出、注销、重启系统 |
| 0x5E | 打开驱动相关服务 |
| 0x5F | 停止并删除驱动文件 |
| 0x60 | 获取系统日志 |
| 0x64 | 实现远程控制系统 |
| 0x72 | 远程下载文件并执行 |
| 0x74 | 关闭相关进程并删除文件 |
| 0x75 | 设置分辨率 |
| 0x76 | 创建插件文件并执行 |
| 0x7F | 打开3389相关服务 |

![1642556369172.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-c5f619b20efbdb966d1688d675ffd37bf757c9da.png)

溯源
--

该样本在互斥体创建、数据解密、行为逻辑上与Gh0st 远控高度相似。

![1642556381521.png](https://shs3.b.qianxin.com/attack_forum/2022/01/attach-06a534a17c7bb6185e60aea953257c34f580bc7f.png)

总结
--

洁身自好，没事别瞎下载一些奇奇怪怪的文件。