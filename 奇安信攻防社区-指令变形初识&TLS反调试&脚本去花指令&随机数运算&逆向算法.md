### main函数真的是最早执行的函数吗

![image-20220817151422340.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-cb3f127570cb8dd8775f06ed5bd724137f783b37.png)

先写一个简单的代码，生成后来进行观察

放到od中先出现的是oep

![image-20220817151859982.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-6164137a9f353f199f7c8b7a24a259654262fc76.png)

往下找才能找到main函数

![image-20220817152139779.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-e6a51ed6e26455550f7b41b3d547589a24084658.png)

而在main函数前还有很多的执行代码，只观察这一个小程序就可以看到这句话是错误的

还有个更直观的方法

先设置一个全局变量，然后观察是先生成全局变量还是main函数

![image-20220817152322002.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-6a9732d7c66166da1352a25adf82cd019945a11e.png)

运行程序后

![image-20220817152534383.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-43f29711cd5e3f7dfa33f1df701f0d18dbc18c7d.png)

先弹出的消息是“我是构造函数”

![image-20220817152630080.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-40cc877afef4b1debbad63e1323005121516f6af.png)

然后再弹出“我是main函数”

再使用od进行动态调试，进行观察

弹出“我是构造函数”消息框的是使用的这个函数

![image-20220817152922333.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-edf7343d9141853438c00e3ccd8402ec538f3ab5.png)

![image-20220817152810145.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-55e8802e47d61e21ad6c2177120a085a32e9ef27.png)

在这个函数下面才用到了main函数

而比OEP还早的功能是`TLS(线程局部存储)`

TLS
---

### 这个功能的设计初衷是为了方便线程访问全局变量的

若是想要在程序中使用这个功能，需要添加

```c++
//我要用到TLS这个东西了，在文件上方
#pragma comment(linker,"/INCLUDE:__tls_used")

//新建一段数据，放到TLS这个目录表里面
#pragma data_seg(".CRT$XLX")
PIMAGE_TLS_CALLBACK pTLS_CALLBACKs[] = { TLS_CALLBACK,NULL };
#pragma data_seg()
```

使用了这个功能后会出现一个TLS目录表

![image-20220817160710300.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-4337322e45485ed8714faf17113bff90fa451163.png)

tls目录描述了tls结构体

![image-20220817171357599.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-0f71be3504110891bfe014141d4b8a1f94f1acb9.png)

这个结构体主要的是`AddressOfCallBacks`这个值，这里面存放的是代码中`PIMAGE_TLS_CALLBACK pTLS_CALLBACKs[] = { TLS_CALLBACK,NULL };`的数据，是函数指针数组

```c++
// TLS反调试.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//

#include 
#include 
#include "MINT.h"

#pragma comment(linker,"/INCLUDE:__tls_used")

DWORD isDebug = 0;
void NTAPI TLS_CALLBACK(PVOID DllHandle, DWORD Reason, PVOID Reserved)
{
    if (Reason == DLL_PROCESS_ATTACH)
    {
        MessageBoxA(0, "TLS函数执行", 0, 0);
        //NtSetInformationThread(GetCurrentThread(), ThreadHideFromDebugger, 0, 0); // 禁止调试
        NtQueryInformationProcess(GetCurrentProcess(), ProcessDebugPort, (PVOID)&amp;isDebug, sizeof(DWORD), NULL); // 检查是否被调试
    }
}

int main()
{
    MessageBoxA(NULL, "Main函数执行", "提示", MB_OK);
    system("pause");
}

#pragma data_seg(".CRT$XLX")
PIMAGE_TLS_CALLBACK pTLS_CALLBACKs[] = { TLS_CALLBACK,NULL };
#pragma data_seg()

```

使用这个程序来进行演示和验证

生成程序后，是先执行了TLS，再执行的main

![image-20220817172507622.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-2bbaa51451af760153a13d7f3a5aa25d691db4bf.png)

![image-20220817172520353.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-87f66ba1df114126d0376387f3de4346e1d65e85.png)

后面还有联系到花指令，所以介绍一下

1. 花指令的作用
2. 花指令设计思想(构成恒成立的EIP跳转，插入无效数据)
3. 花指令的优缺点(防静态分析，放不住动态调试)
4. 手动去除花指令
5. 用工具自动去除花指令
6. 模糊匹配，编写脚本去除花指令

**因为只要构成恒成立的EIP跳转就会产生花指令，所以下面这个程序就可以在没有出现`jmp`的指令下也构造了花指令**

放入od中查看到这样一个操作

![image-20220817174529892.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-61c8d4c71058079740669fc50b7017684a812280.png)

这个步骤call了自己下面的地址，call了这个地址会将这个返回地址压入堆栈

然后将这个地址加了`0x17`，返回值变成了`00402022`，并且下面的指令为`retn`，那就会直接跳到上面计算后的返回地址

所以在00402013-00402021，这些地址上的指令都没有做，那就证明这一段都是花指令，没有用的指令，若是将这些指令保留，则会影响反调试

所以可以将这一大段的代码都修改为`nop`

![image-20220817175836118.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-55488d7dbb3116edc55f2f40177b36c0b0a529ea.png)

00402021这个地址可以数据窗口中跟随，然后将第一个字节修改为`90`

![image-20220817175709701.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-badea61d374f9fee9d07973b49481f967164a6a6.png)

![image-20220817175749785.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-8c57d4a49ef8ed0d9dbeb9bc938194e4c064c574.png)

再接着向下进行分析

![image-20220817180454816.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-0fb0857912ec48e1f2c871d87694f5ad500b979e.png)

往下又出现一个`call 00402032`这个指令，而在00402032前还有一个00402031的数据，那这个数据又没有用到，又是一个无效数据，直接进行`nop`

![image-20220817180707016.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-c135831c002cc4ce006c075abf4bcd8bf3d315d5.png)

call指令完成后，又会将地址压入栈中

![image-20220817180830922.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-1c6690b434c7a2b820247e9cbe17680e96fb905b.png)

接着又将地址加了`0x6`，地址计算后变成了`00402037`，就会在`retn`后返回到`00402037`的地址上

所以从0040202C-00402036期间又是一段花指令，进行`nop`填充

![image-20220817181535735.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-33ddcf55b120dfe79bf8bf969c38cad6a259e64d.png)

再往下分析，发现都是一样的规律

![image-20220817183610195.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-75302c1977b90db7088cd453b8fc69832310d77f.png)

又是call了一个地址，那00402043的指令又没有用了，进行`nop`填充

![image-20220817183808578.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-0ffdb4ac26c14664023a502fd78675d0c04e444c.png)

然后有将地址进行加法`0x6`，地址更改为00402049，那就证明0040203E-00402048的数据又是无效数据，进行`nop`填充

![image-20220817184042425.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-e59e51acc26a35e9fc4d42e98fc26a7b026b0dbb.png)

之后我们就观察到当数据以`E8 01000000`开头，`C3`结尾的这一段都是花指令

![image-20220817184327945.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-9a995f4293acc2ee72bb252d3b99c709a5dc3ecc.png)

这样类似的花指令还有很多

再继续向下进行分析

![image-20220817231612301.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-1d90a56ab23c4f1ac00a19ad3dafe58cfb12ea19.png)

call  
这里就调用了TLS里的`NtSetInformationThread()`函数

&gt; 若是这个函数里的第二个参数设置为`ThreadHideFromDebugger`就不会接收到内核反馈给我们的程序的调试信息了，那这个程序就无法再进行调试了

函数其中第一个参数为`GetCurrentThread()`线程句柄

也就是对应上图中的第一个参数`eax`

而下一个参数为`ThreadHideFromDebugger`，将这个参数的值设成了`0x11`

而在这个参数中的选项中10进制的17转为16进制则为11

![image-20220817232348280.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-4bb497714fd8caffe77f0785ee5eefc7886c9d01.png)

刚好为设置的`ThreadHideFromDebugger`，而这个值就是用来禁止调试的

所以这个程序就是在这里进行了**`调试器逃逸`**

![image-20220817232446631.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-429bc0697f0e7b3ee20446c58c98b8eec69687e3.png)

再继续向下进行分析，在分析时也会遇到一些花指令的干扰，还是按照之前的额方法进行`nop填充`即可

![image-20220817233704444.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-0d96a7146711b5a516ba9f51b671e1a38fa53446.png)

再次读取当前进程句柄后就调用了另一个函数`NtQueryInformationProcess`

![image-20220817233435485.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-e54d2bfc3c0a0f5cab4a3ece14c0e13eded77893.png)

&gt; 这个函数是查询程序的调试端口，

第一个参数为当前进程句柄，也就是上图中对应的`ebx`的值

第二个值为`0x7`也就是对应选项中的第七位`ProcessDebugPort`

![image-20220817234118471.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-30f87566adfa3fc16ee13dedaa7c6ab8ddb5dd9a.png)

这个函数需要检查的地方就是这里

再向下进行分析

接着将查询到的值，放在`0x7`的下一的参数中

![image-20220817234451737.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-a8b75ac3ef42d36ed5afc6bae975ddcf258566c5.png)

接着比较存储到的值eax是否等于`0`

若是判断为不是`0`，这个程序就会跑崩了

我们可以直接将值进行更改

接着向下进行分析

![image-20220817235033034.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-ac3a99d011a66db09da19d92fca451446f190b2d.png)

接着让一个全局变量与`0x0`做比较，其实也就是代码中这个参数的设置

![image-20220817235123551.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-0129001a37e7f38ff58a64303af78ec4eeb4cd36.png)

若是这个值不是`0`，程序也会崩

![image-20220817235332515.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-a88ce18ec0b2bf31208dbb3ceb5fd09cf67f762f.png)

这个指令代表若是上面的比较判断为失败，则会将esi的值给edi，也就是证明反调试给检测到了

![image-20220817235634841.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-00e4e9debca49c4af2f57eb7ed47e58ac83df172.png)

而此时`esi`的值为`00000000`，那`edi`的值则变为`00000000`

那再往下分析就会看到edi的值会给到`0x00404036`

![image-20220818155810530.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-b00e44c67718a079b2156590999e198107889485.png)

再往回看一步，00404032

![image-20220818155941740.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-fdb06918aef842cb667001182a0b2583b6d40221.png)

![image-20220818160023238.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-74956d71936a9334bc1db1dedec64d8056b0293b.png)

而这个地方存放的地址为程序运行的入口

![image-20220818160107784.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-83c0aa2a34c94196827e1090f1ea7a6f90411fbb.png)

&gt; **整合下来这个程序相当于在动态Patch TLS回调函数**zai

再接着向下分析

![image-20220818160550465.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-9f3b131a9aa4855dad65f36b9a0606c0295afcf4.png)

在00404036这里存储的地址就代表着下一个回调函数的地址，接着向下执行就会跳到这里

![image-20220818160711389.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-7a6547432e206f34cbb72182215a808c2c1eed9a.png)

而这个TLS检测会调用无数次，再继续分析

![image-20220818164742517.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-64a72b3199d1ec5c31c4169c3d470699cf63ce67.png)

上图中的`0040406C`是读取到的是否未调试的位置(也就是上面说的全局变量)，然后再跟`0x0`进行运算

所以要是一直进行检测，则若是这个程序填入了正确的注册码，还是依然不能激活

那我们需要先找到这个常量

![image-20220818180945432.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-2f9af4d03b88b93e8efcdce4fad1699b62a8ce0f.png)

![image-20220818182735029.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-5d89cabea0eae80a356fb68de889df61038ea3b1.png)

![image-20220818182804489.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-bf63efdb57322a25ab6fc188462f1b5b78abb745.png)

这里有一个异或的操作，所以需要将这个反调试给破掉

**这里的TLS就是演示和大概的演示，我是用的程序都有反调试的功能所以就没有实际的演示**

### 脚本去除花指令

花指令的固定格式

&gt; E8 01 00 00 00 C2 83 04 24 06 C3

经过观察发现，这个花指令的结构为，前面为`E8 01 00 00 00`开头，并且结尾一定为`C3`

而od这个程序支持模糊匹配，所以我们可以根据这个功能来写一个脚本

我们可以在这个花指令格式中把会发生改变的字节用`??`来代替

&gt; E8 01 00 00 00 ?? ?? ?? ?? ?? C3

所以直接在od中使用`ctrl+b`将上面一段hex进行查找，就可以一个个的去找到花指令，但是要将所有的花指令都去除掉也需要花费很多时间，但是我们找到了花指令的模板，就可以直接写脚本来自动清除花指令

### **去除花指令脚本**

```php
//从EIP的位置查找第一个特征码
find eip, #E80000000081042417000000C3576174636820757220737465702100#
//判断是否找到
cmp $RESULT, 0
//如果没有找到就跳出到结束

je exit
//如果找到就填充成NOP
mov [$RESULT], #90909090909090909090909090909090909090909090909090909090#
//从EIP的位置查找下一个特征码
find eip, #E80000000081042425000000C354686520666C616720626567696E7320776974682022666C61677B2200#
//判断是否找到
cmp $RESULT, 0
//没找到就退出
je exit
//找到就填充为NOP
mov [$RESULT], #909090909090909090909090909090909090909090909090909090909090909090909090909090909090#
//定义一个循环的标签，因为下面这个特征码不止一处，所以需要循环多次进行查找
loop:
    //从EIP的位置查找
    find eip, #E801000000??????????C3#
    //判断是否找到
    cmp $RESULT, 0
    //如果找不到了就退出
    je exit
    //找到就填充为NOP
    mov [$RESULT], #9090909090909090909090#
//继续循环
jmp loop
exit:
MSG "complete! \r\n"
ret

```

若是想要执行脚本，在页面右键选择运行脚本的功能选中要运行的脚本就可以直接运行了

![image-20220820185838283.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-9708b75a6b74b06c12e95b81538e2222fd87bc5f.png)

若是想要看到运行的过程可以选中下面的脚本功能中的脚本运行窗口，然后按`Tab`键一步步运行调试

![image-20220820190003951.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-510b2ccda31605b288f99253556c023da3b5887e.png)

![image-20220820190030322.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-a19cbba687836e6681f0e22664bb7fb2033d1c12.png)

![image-20220820190053858.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-5eaf6d7ecfa8a480d68c6e67387fb721fc0657cf.png)

这样就可以动态的看到去除花指令的过程

![image-20220820190132481.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-b44783a0208c76cfc4e16a9b76bae1745da14434.png)

![image-20220820190143590.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-651d58f73b70ad68cf2c3de38a5e153db3c14754.png)

完成后会进行弹窗提示

### 逆向算法

分析完这些要就可以去看看算法了，先把程序拖到IDA中

![image-20220820192309810.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-37a5cac771e4e658a87a468e64b782d5dba17fcf.png)

首先会看到有一个随机数，那第一步需要将种子算出来

并且还给了提示

![image-20220820192211101.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-8e97beaca22f4d0d64608b3b0fd08fa8bf8a6e45.png)

说了flag的第一个字符是`f`，这就相当与一个突破点，可以先把v6的条件满足就能知道随机数是多少，然后再根据随机数暴力破解种子，种子是 isdebug ^ keySum 计算得来，isdebug是在第二个TLS回调函数里设置的，固定就是 0x31333359（可以看上面分析TLS时说的），有了种子就可以破解出所有的字符

但是要注意IDA F5反汇编的代码有错误

.text:004011D8 mul edx  
.text:004011DA div ecx  
.text:004011DC mov eax, edx  
IDA F5插件无视了这三行

因为上面提示了说第一个字符是`f`，那其实我们再接着往下分析就可以知道前五个字符为`flag{`，那就可以根据这个特征来计算

```php
// 如果知道 key 的累加和，就可以得到随机数种子
// 要想计算得到 key 的累加和，只能通过 v6 = (unsigned __int8)key[i] * rand(); 反推，反推的方法是暴力枚举srand种子
// 已知key以"flag"开头，则key[0] - key[3] 都是确定的，那么只要找到一个种子，使前4个字符计算后和dword_4030B4[i]相等
void CalcKeySum()
{
    char key[5] = "flag";
    // 爆破keySum
    for (unsigned int keySum = 0; keySum &lt; 255*42; keySum++)
    {
        srand(keySum ^ isdebug);
        int i;
        for (i = 0; i &lt; 4; i++)
        {
            unsigned int v6 = (unsigned __int8)key[i] * rand();
            unsigned int v7 = v6 * (unsigned __int64)v6 % 0xFAC96621;
            unsigned int v8 = v7 * (unsigned __int64)v7 % 0xFAC96621;
            unsigned int v9 = v8 * (unsigned __int64)v8 % 0xFAC96621;
            unsigned int v10 = v9 * (unsigned __int64)v9 % 0xFAC96621;
            unsigned int v11 = v10 * (unsigned __int64)v10 % 0xFAC96621;
            unsigned int v12 = v11 * (unsigned __int64)v11 % 0xFAC96621;
            unsigned int v13 = v12 * (unsigned __int64)v12 % 0xFAC96621;
            unsigned int v14 = v13 * (unsigned __int64)v13 % 0xFAC96621;
            unsigned int v15 = v14 * (unsigned __int64)v14 % 0xFAC96621;
            unsigned int v16 = v15 * (unsigned __int64)v15 % 0xFAC96621;
            unsigned int v17 = v16 * (unsigned __int64)v16 % 0xFAC96621;
            unsigned int v18 = v17 * (unsigned __int64)v17 % 0xFAC96621;
            unsigned int v19 = v18 * (unsigned __int64)v18 % 0xFAC96621;
            unsigned int v20 = v19 * (unsigned __int64)v19 % 0xFAC96621;
            unsigned int v21 = v20 * (unsigned __int64)v20 % 0xFAC96621;
            unsigned int v22 = v21 * (unsigned __int64)v21 % 0xFAC96621; // 注意，IDA的F5插件漏了这行，大坑
            // 还有一个坑，(unsigned __int64)v6 不要省略强转，IDA怎么生成就怎么抄
            // 因为在vs里，64位的求余根本没用div指令，不强转就算不出来了
            if ((unsigned __int64)v6 % 0xFAC96621 * (unsigned int)v22 % 0xFAC96621 != dword_4030B4[i])
            {
                break;
            }           
        }
        if (i == 4)
        {
            printf("keySum = %X\n", keySum);
            seed = keySum ^ isdebug;
            break;
        }       
    }
}

```

爆破得到keySum后，马上就能算出前42次随机数的值

```php
void SetRand()
{
    srand(seed); // 爆破得到的种子
    for (int i = 0; i &lt; 42; i++)
    {
        Rand[i] = rand();
    }
}

```

接着就可以爆破密码

```php
// 1.cpp : 定义应用程序的入口点。
//
#define _CRT_SECURE_NO_WARNINGS
#include 
#include 
#include 
using namespace std;

DWORD dword_4030B4[42] = {
    0x63B25AF1, 0x0C5659BA5, 0x4C7A3C33, 0x0E4E4267, 0x0B611769B,
    0x3DE6438C, 0x84DBA61F, 0x0A97497E6, 0x650F0FB3, 0x84EB507C,
    0x0D38CD24C, 0x0E7B912E0, 0x7976CD4F, 0x84100010, 0x7FD66745,
    0x711D4DBF, 0x5402A7E5, 0x0A3334351, 0x1EE41BF8, 0x22822EBE,
    0x0DF5CEE48, 0x0A8180D59, 0x1576DEDC, 0x0F0D62B3B, 0x32AC1F6E,
    0x9364A640, 0x0C282DD35, 0x14C5FC2E, 0x0A765E438, 0x7FCF345A,
    0x59032BAD, 0x9A5600BE, 0x5F472DC5, 0x5DDE0D84, 0x8DF94ED5,
    0x0BDF826A6, 0x515A737A, 0x4248589E, 0x38A96C20, 0x0CC7F61D9,
    0x2638C417, 0x0D9BEB996
};

DWORD isdebug = 0x31333359; // 过掉反调试后得出的值，用于计算种子
DWORD seed; // 随机种子
DWORD Rand[42]; // 伪随机数组，因为种子已经爆破得到，所以42个随机数已经确定

BOOL __stdcall CheckKey(char* key)
{
    int keySum; // edi
    unsigned __int8 c; // al
    char* ptr; // esi
    unsigned int i; // ebx
    unsigned int v6; // eax
    unsigned int v7; // edx
    unsigned int v8; // edx
    unsigned int v9; // edx
    unsigned int v10; // edx
    unsigned int v11; // edx
    unsigned int v12; // edx
    unsigned int v13; // edx
    unsigned int v14; // edx
    unsigned int v15; // edx
    unsigned int v16; // edx
    unsigned int v17; // edx
    unsigned int v18; // edx
    unsigned int v19; // edx
    unsigned int v20; // edx
    unsigned int v21; // edx
    unsigned int v22; // IDA 漏了这句
    if (strlen(key) != 42)
    {
        MessageBoxA(0, "密码长度!=42", "", MB_OK);
        return FALSE;
    }
    keySum = 0;
    c = *key;
    ptr = key + 1;
    while (c)
    {
        keySum += c;
        c = *ptr++;
    }
    srand(isdebug ^ keySum);                      // isdebug == 0x31333359

    for (i = 0; i != 42; ++i)
    {
        v6 = (unsigned __int8)key[i] * rand();
        v7 = v6 * (unsigned __int64)v6 % 0xFAC96621;
        v8 = v7 * (unsigned __int64)v7 % 0xFAC96621;
        v9 = v8 * (unsigned __int64)v8 % 0xFAC96621;
        v10 = v9 * (unsigned __int64)v9 % 0xFAC96621;
        v11 = v10 * (unsigned __int64)v10 % 0xFAC96621;
        v12 = v11 * (unsigned __int64)v11 % 0xFAC96621;
        v13 = v12 * (unsigned __int64)v12 % 0xFAC96621;
        v14 = v13 * (unsigned __int64)v13 % 0xFAC96621;
        v15 = v14 * (unsigned __int64)v14 % 0xFAC96621;
        v16 = v15 * (unsigned __int64)v15 % 0xFAC96621;
        v17 = v16 * (unsigned __int64)v16 % 0xFAC96621;
        v18 = v17 * (unsigned __int64)v17 % 0xFAC96621;
        v19 = v18 * (unsigned __int64)v18 % 0xFAC96621;
        v20 = v19 * (unsigned __int64)v19 % 0xFAC96621;
        v21 = v20 * (unsigned __int64)v20 % 0xFAC96621;
        v22 = v21 * (unsigned __int64)v21 % 0xFAC96621; // IDA漏了这句
        if (v6 % 0xFAC96621 * (unsigned __int64)v22 % 0xFAC96621 != dword_4030B4[i])
            break;
    }
    if (i &gt;= 0x2A)
        return TRUE;
    else
        return FALSE;
}

// 如果知道 key 的累加和，就可以得到随机数种子
// 要想计算得到 key 的累加和，只能通过 v6 = (unsigned __int8)key[i] * rand(); 反推，反推的方法是暴力枚举srand种子
// 已知key以"flag"开头，则key[0] - key[3] 都是确定的，那么只要找到一个种子，使前4个字符计算后和dword_4030B4[i]相等
void CalcKeySum()
{
    char key[5] = "flag";
    // 爆破keySum
    for (unsigned int keySum = 0; keySum &lt; 255 * 42; keySum++)
    {
        srand(keySum ^ isdebug);
        int i;
        for (i = 0; i &lt; 4; i++)
        {
            unsigned int v6 = (unsigned __int8)key[i] * rand();
            unsigned int v7 = v6 * (unsigned __int64)v6 % 0xFAC96621;
            unsigned int v8 = v7 * (unsigned __int64)v7 % 0xFAC96621;
            unsigned int v9 = v8 * (unsigned __int64)v8 % 0xFAC96621;
            unsigned int v10 = v9 * (unsigned __int64)v9 % 0xFAC96621;
            unsigned int v11 = v10 * (unsigned __int64)v10 % 0xFAC96621;
            unsigned int v12 = v11 * (unsigned __int64)v11 % 0xFAC96621;
            unsigned int v13 = v12 * (unsigned __int64)v12 % 0xFAC96621;
            unsigned int v14 = v13 * (unsigned __int64)v13 % 0xFAC96621;
            unsigned int v15 = v14 * (unsigned __int64)v14 % 0xFAC96621;
            unsigned int v16 = v15 * (unsigned __int64)v15 % 0xFAC96621;
            unsigned int v17 = v16 * (unsigned __int64)v16 % 0xFAC96621;
            unsigned int v18 = v17 * (unsigned __int64)v17 % 0xFAC96621;
            unsigned int v19 = v18 * (unsigned __int64)v18 % 0xFAC96621;
            unsigned int v20 = v19 * (unsigned __int64)v19 % 0xFAC96621;
            unsigned int v21 = v20 * (unsigned __int64)v20 % 0xFAC96621;
            unsigned int v22 = v21 * (unsigned __int64)v21 % 0xFAC96621; // 注意，IDA的F5插件漏了这行
            //(unsigned __int64)v6 不要省略强转，IDA怎么生成就怎么抄
            // 因为在vs里，64位的求余根本没用div指令，不强转就算不出来了
            if ((unsigned __int64)v6 % 0xFAC96621 * (unsigned int)v22 % 0xFAC96621 != dword_4030B4[i])
            {
                break;
            }
        }
        if (i == 4)
        {
            printf("keySum = %X\n", keySum);
            seed = keySum ^ isdebug;
            break;
        }
    }
}

void SetRand()
{
    srand(seed); // 爆破得到的种子
    for (int i = 0; i &lt; 42; i++)
    {
        Rand[i] = rand();
    }
}

int main()
{
    CalcKeySum(); // 爆破种子（0xE61）
    SetRand(); // 设置伪随机数组

    // 爆破密码
    for (int i = 0; i &lt; 42; i++)
    {
        for (unsigned char ch = 0; ch &lt; 0xFF; ch++)
        {
            unsigned int v6 = (unsigned __int8)ch * Rand[i];
            unsigned int v7 = v6 * (unsigned __int64)v6 % 0xFAC96621;
            unsigned int v8 = v7 * (unsigned __int64)v7 % 0xFAC96621;
            unsigned int v9 = v8 * (unsigned __int64)v8 % 0xFAC96621;
            unsigned int v10 = v9 * (unsigned __int64)v9 % 0xFAC96621;
            unsigned int v11 = v10 * (unsigned __int64)v10 % 0xFAC96621;
            unsigned int v12 = v11 * (unsigned __int64)v11 % 0xFAC96621;
            unsigned int v13 = v12 * (unsigned __int64)v12 % 0xFAC96621;
            unsigned int v14 = v13 * (unsigned __int64)v13 % 0xFAC96621;
            unsigned int v15 = v14 * (unsigned __int64)v14 % 0xFAC96621;
            unsigned int v16 = v15 * (unsigned __int64)v15 % 0xFAC96621;
            unsigned int v17 = v16 * (unsigned __int64)v16 % 0xFAC96621;
            unsigned int v18 = v17 * (unsigned __int64)v17 % 0xFAC96621;
            unsigned int v19 = v18 * (unsigned __int64)v18 % 0xFAC96621;
            unsigned int v20 = v19 * (unsigned __int64)v19 % 0xFAC96621;
            unsigned int v21 = v20 * (unsigned __int64)v20 % 0xFAC96621;
            unsigned int v22 = v21 * (unsigned __int64)v21 % 0xFAC96621; // IDA漏了这句
            if (v6 % 0xFAC96621 * (unsigned __int64)v22 % 0xFAC96621 == dword_4030B4[i])
                putchar((char)ch);
        }
    }

    //char key[] = "flag{wh3r3_th3r3_i5_@_w111-th3r3_i5_@_w4y}";
    //if (CheckKey(key))
    //{
    //    printf("密码正确\n");
    //}
    //else
    //{
    //    printf("密码错误\n");
    //}
    return 0;
}

```

![image-20220820194844656.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-7307c38de1f178baf208c36d46de0232cd2381c8.png)

![image-20220820195618483.png](https://shs3.b.qianxin.com/attack_forum/2022/08/attach-3caddbc9ea0dd72cc379793124e9df2ac15c00ab.png)

这个演示题目还是较难的，对于我这种差不多刚入门的学了很久才搞明白，其中一些知识有参考NCK课程的内容还有一些博客