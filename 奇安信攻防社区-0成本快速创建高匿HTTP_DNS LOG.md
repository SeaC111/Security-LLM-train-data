0x0 前言
------

 经过log4j2漏洞的洗礼，相信很多人意识到自建DNSLog的重要性。笔者的工作流有两个，一个是类似P神Conote那种常规化的功能高度集成的WEB平台，另一个则是今天要与大家分享的，如何零成本快速创建一个高匿HTTP/DNS LOG服务的。

0x1 申请域名
--------

挂上代理访问:<https://www.freenom.com/zh/index.html?lang=zh>

![image-20211218130306410](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5e163cb70929af1aaf56fa75e1fd75b73b19eeaf.png)

输入一个域名进行获取即可，然后点击右上角的`check out`去结算。

![image-20211218130536805](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d6fc165104f00a3504af378f850e05299380e189.png)

申请域名结束后，进入域名管理，配置NameServer,指向CloudFlare。

```bash
darl.ns.cloudflare.com
gene.ns.cloudflare.com
```

![image-20211218130726605](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4129568387792e85d007d8556486a1888a79ad91.png)

CloudFlare的域名解析记录的速度是非常快的,后面添加到[CloudFlare](https://www.cloudflare.com/)即可。

![image-20211218131154513](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f48aa5f2205d3e0c1683e51979618dfa96972133.png)

0x2 配置NS记录
----------

分别申请两个DNS记录，NS记录代表子域名`log.log4j.ga`的DNS服务器为`ns.log4j.ga`。

然后我们设置`ns.log4j.ga`的A记录指向我们DNS服务器所在的地址即可。

![image-20211218143318306](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-8ce7297e11c5d2237292510a471668337fc930c5.png)

为了测试是否生效，可以用Python2写一个简单的DNS服务器返回指定的应答IP。

```python
#/usr/bin/python2
# -*- coding:utf-8 -*-
from twisted.internet import reactor, defer
from twisted.names import client, dns, error, server
record={}
class DynamicResolver(object):
    def _doDynamicResponse(self, query):
        name = query.name.name
        # 返回指定应答IP
        ip = "x.x.x.x"
        print(name+" ===> "+ip)
        answer = dns.RRHeader(
            name=name,
            type=dns.A,
            cls=dns.IN,
            ttl=0, # 这里设置DNS TTL为 0
            payload=dns.Record_A(address=b'%s'%ip,ttl=0)
        )
        answers = [answer]
        authority = []
        additional = []
        return answers, authority, additional
    def query(self, query, timeout=None):
        return defer.succeed(self._doDynamicResponse(query))
def main():
    factory = server.DNSServerFactory(
        clients=[DynamicResolver(), client.Resolver(resolv='/etc/resolv.conf')]
    )
    protocol = dns.DNSDatagramProtocol(controller=factory)
    reactor.listenUDP(53, protocol)
    reactor.run()
if __name__ == '__main__':
    raise SystemExit(main())
```

如图说明配置成功:

![image-20211218144731214](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4765d3fef99e449fce3194b3e90b1ab40d992f9f.png)

0x3 选择一 部署Knary
---------------

二进制文件部署:

```bash
wget https://ghproxy.com/https://github.com/sudosammy/knary/releases/download/v3.3.1/knary-3.3.1-linux-amd64
chmod +x knary-3.3.1-linux-amd64
mv knary-3.3.1-linux-amd64 knary
./knary
```

如果二进制不安全，可以考虑Docker 快速搭建:

```bash
export GIT_SSL_NO_VERIFY=0
git clone http://ghproxy.com/https://github.com/sudosammy/knary.git
# 编辑docker-compose.yaml的信息
vim docker-compose.yaml 
```

可以看到环境变量信息有两种配置方式，一种是利用当目前的`.env`文件，一种是直接配置环境变量`environment`,也可以混合来用，我这里先配置下`CANARY_DOMAIN`=`CANARY_DOMAIN=log.log4j2.ga`。

```dockerfile
version: "3.9"
services:
  knary:
    container_name: knary
    hostname: knary
    restart: always
    env_file:
      - .env # as commented below, you can also use the `environment` key to specify variables. This will overwrite anything in `env_file`
    # environment:
    #   - DNS=true
    #   - HTTP=true
    #   - BIND_ADDR=127.0.0.1
       - CANARY_DOMAIN=log.log4j2.ga
    # (etc. etc.)
    build: .
    ports:
      - "80:80"
      - "443:443"
      - "53:53/udp"
    volumes:
      - ./certs:/certs/                                                                                                                            
```

编辑一份环境变量:

```bash
vim .env
# 内容
# RENAME ME TO .env
DNS=true
HTTP=true
BIND_ADDR=0.0.0.0
CANARY_DOMAIN=log.log4j.ga
LETS_ENCRYPT=xxxxx@qq.com
EXT_IP=1x1.xxx.1xx.x3

DEBUG=true
LOG_FILE=knary.log
```

有两种方法，快速配置HTTPS证书:

1)自签名证书

当前目录创建一个cert文件夹

```bash
mkdir certs
```

openssl创建证书

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./certs/selfsigned.key -out ./certs/knary.crt
```

信息可以随意填，不重要。

![image-20211218195525230](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-6cf6d2f2a14ab6855cfb4fb31c70b55c4214f276.png)

2)Let's Encrypt 免费https，Kanry自带，只需要配置好邮箱即可，推荐使用这种!

docker-compose启动

```bash
# 卸载原先docker
apt-get remove -y remove docker
# 安装最新docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
# 安装docker-compose
sudo curl -L 
"https://ghproxy.com/https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
```

![image-20211219180606762](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-b43989aeee55fd65146f6f00b47eb3ab2e3a7929.png)

Ubuntu构建过程可能会报错,修改`/etc/resolv.conf`的值为这个即可。

```php
nameserver 8.8.8.8
nameserver 114.114.114.114
nameserver 127.0.0.53
nameserver 223.5.5.5
options edns0
```

为了兼容国内VPS还需要修改Dockerfile添加`Golang`代理,要不然`go get`会失败:

```bash
RUN export GOPROXY=https://goproxy.io && go get .
```

![image-20211219194825369](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f4dd7b8a78f3e90f3e143325bf9ba23d29758f99.png)

**效果展示**

强烈推荐直接二进制运行，非常方便!

![image-20211219204832896](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1b63284a5e0e5307bcbdd256f6fa71da5391ee44.png)

Docker运行界面:

![image-20211219211439135](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-54bed0ddef2a4f9594dce94be4107588271d28e3.png)

0x4 选择二 部署Hyuga
---------------

```bash
git clone https://github.com/Buzz2d0/Hyuga.git
cd Hyuga
```

修改`config.xml`文件

```bash
cd hyuga/
vim config.yaml
```

配置如图所示:

![image-20211219211816343](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5a2d754fc18b73a41efa1e7a56ba38db4d1d0126.png)

Docker快速组建:

```bash
docker-compose build
docker-compose up -d
```

如图所示，搭建成功！

![image-20211219215119823](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-1051b9557b9c54a8e4acf62d3bca79a68a4b7c4d.png)

访问:<http://log.log4j.ga/>

DNS记录:

![image-20211219215443401](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-f1df1c127b3bab8631054b34e22193433beea2af.png)

HTTP记录:

![image-20211219215531383](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-fc8ded2e1ea77da61546fde534a803de63aae30f.png)

设置DNS重绑定:

![image-20211219222400230](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-9b8a9243b8bbb92aeda65fcbe01ce916997a8011.png)

```php
for((i=1;i<=2;i++))                                                                                             
do
    sleep 0.5 && dig @8.8.8.8 r.amz3.log.log4j.ga;
done
```

![image-20211219222343647](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-0906bee51cd32e79c793b11fe3edc344559f6a4b.png)

0x5 选择三 部署Interactsh
--------------------

1\) 手工搭建

安装Golang 1.16.1环境:

```bash
sudo apt-get update
sudo apt-get -y upgrade
apt install  -y gcc

mkdir tmp
cd /tmp
wget https://dl.google.com/go/go1.16.1.linux-amd64.tar.gz
sudo tar -xvf go1.16.1.linux-amd64.tar.gz 
sudo mv go /usr/local/

echo "export GOROOT=/usr/local/go" >>  ~/.profile
echo "export GOPATH=$HOME/go" >>  ~/.profile
echo "export PATH=$GOPATH/bin:$GOROOT/bin:$PATH" >> ~/.profile
source ~/.profile
```

安装Interachsh Server

```bash
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-server@latest
```

2\) docker搭建

```bash
docker run projectdiscovery/interactsh-server:latest -domain log.log4j.ga
```

用法:

```bash
interactsh-server -domain  log.log4j.ga -auth
```

![image-20211219232717497](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-d21c3056392af6e142b587a7a8211d97d93fedfa.png)

获取到Client Token,打开https://app.interactsh.com/#/

![image-20211219232940758](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-a5b5e693de038788b3425620b075a871d467419a.png)

结果界面显示:

![image-20211219233310416](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5d1971d5496e6203c645c1bda077c3ace71ff8d1.png)

0x6 分析三者优缺点
-----------

Knary的优点在于支持单文件部署，支持国外的信息推送、支持域名过滤，支持DNS、HTTP/HTTPS，缺点则是缺乏可视化界面、不支持生成随机域名，且运行的时候并不是很稳定，内存占用有点高。

Hyuga是国内开发者开发的，可以说很符合笔者的需求，有良好、简洁的WEB界面，支持DNS、HTTP、支持DNS重绑定、支持API，安装过程也很简单，但是很可惜缺乏信息推送和支持HTTPS请求，但是目前来说这个工具能满足大部分的需求了。

Interactsh安装过程也并不复杂，支持DNS、HTTP/HTTPS等多种协议，有可视化界面，支持生成随机域名，可通过与projectdiscovery的notify工具联动来进行信息推送，缺点不支持国内信息推送流、生成的随机域名太长，容易被判定为恶意网址。

笔者针对三者做了个简单的对比表格:

|  | [Knary](https://github.com/sudosammy/knary/) | [Hyuga](https://github.com/Buzz2d0/Hyuga) | [Interactsh](https://github.com/projectdiscovery/interactsh/) |
|---|---|---|---|
| DNS | √ | √ | √ |
| HTTP | √ | √ | √ |
| HTTPS | √ | × | √ |
| WEB | × | √ | √ |
| 信息推送 | √ | × | √ |
| API | × | √ | × |
| 支持随机子域 | × | √ | √ |
| 安装过程 | 非常简单 | 简单 | 简单 |
| 使用难易 | 较难 | 简单 | 有点难 |

0x7 问题汇总
--------

下面由于安装环境的差异，可能会出现这些问题，提供给读者进行参考。

1\) 53端口占用

`vim /etc/systemd/resolved.conf`,如下图配置，关闭监听。

![image-20211219201959506](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-5cb4da62b34c5a9ebb199b7f05f75bd0754ade1c.png)

接着重启服务,`sudo systemctl restart systemd-resolved`,发现53端口不被占用了。

![image-20211219202200260](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-9d3b7dda427c63d078f0d722de4f85fd4d38bed9.png)

如果出现这个错误，VPS一直都是依赖本地的DNS服务器，因为关闭了53端口导致错误。

`vim /etc/resolv.conf`,分别指定两个公共DNS服务器即可。

![image-20211219203009526](https://shs3.b.qianxin.com/attack_forum/2021/12/attach-4a090fb87dcc40f4dac292b5902ee0024ac04f1a.png)

2)域名没有备案，针对国内的VPS。

有两种解决方案:

1)不要用80,8080,443等端口，但这样会限制使用性,不推荐。

2)建议更换为国外服务器，比如Vultr、digital等等，按时间付费都很便宜。

3)乖乖备案，合理使用，适合白帽子！

0x8 总结
------

 本文可以说是保姆级别的手把手教程了，涵盖了Github中较为流行、可用性强的项目具体搭建过程及其调试步骤，一条龙服务踩坑填坑，将三者的优缺点进行了分析。基于本文，读者不仅可以无需购买域名即可快速搭建出私人的Log，还可深入拓展研究，由于三个项目都是开源的，且代码量不大，有能力的读者完全可以去DIY自己的功能。