---

layout: post
title: 理解$hadow$ocks与Proxy原理
date: 2017-04-24
tags: 
- shadowsocks
- proxy

---

<!-- more -->

**以下所有提到\$hadow\$ocks的均以ss指代**

### 为什么要用ss呢？

在早期（如今绝大多数也是），对于互联网的访问流程是及其简单的：浏览器（或其他客户端）向互联网服务器提出一个请求（request），然后等待互联网服务器回应（response），最后在本地解析渲染。麻烦一点的，中间会有多个代理帮助我们进行数据的请求。
![requestresponse](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2017-04-26-ss/requestresponse.png)

后来，墙出现了，我们不能再愉快的访问外网了。因为墙相当于一个巨大的墙阻挡着我们和外界，当我们想要访问外网的时候，这个请求数据包必然会经过墙的检查：
![checkbyw](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2017-04-26-ss/checkbyw.png)
当检查到被墙禁止访问的内容请求的时候，自然会被“过滤”或被“忽视”掉。但，有人还是想要往墙外看看。怎么办呢？有人想出了一个方法，如同最上面图中右边的拓扑，但是中间代理服务器本身就在国外，当我们想要访问外部内容的时候，首先和这个在国外的代理服务器进行ssh通信，告诉它，我想要请求xx资源，再让这台代理服务器去访问目标服务器，得到回应之后，再把我们想要的数据通过ssh传递给我们：
![byssh](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2017-04-26-ss/byssh.png)

刚开始这样还不错。因为ssh本身基于RSA加密，所以首先，墙无法查看其请求内容（ip、内容等），所以也就无法分析你的请求关键词（用于筛选连接），避免了被重置链接的问题，但由于创建隧道和数据传输的过程中，ssh本身的特征是明显的，我们的墙研发人员根据相关的如连接频率、连接时间等进行分析，再通过一定的手段干预连接，于是后来墙愈来越强了。使用ssh tunnel的方式不太灵了！ 

既然ssh你还能看出连接时间、频率等特征值。那我干脆不用了，转换一下，我往外界发送合法的TCP数据包该没有问题吧？好，每次我访问某个资源的时候，首先将请求在本地转换为（拆分、加密等手段）一个个非常规端口的TCP数据包，里面的内容也是加密过的。当我发送给外部服务器的时候，墙只能知道你发送的是合法TCP数据包，它也不敢妄自丢弃和查看特征值，因为他没法保证这个TCP数据包究竟是要请求什么，外面看来本身就是个普通的数据包。等发到外部的代理服务器的时候，我运用相关的协议进行重组解密，得到原始请求。后面就和ssh tunnel有异曲同工之妙。于是，如下的拓扑出现了：
![bysock](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2017-04-26-ss/bysock.png)

理解了这个之后，搭建服务器就是一件很简单的事情了。网上资源很多，这里不再赘述。

### Privoxy作用

当我们搭建好了ss服务器之后，我们本地需要使用ss客户端监听1080（通常的默认端口），配置相关的命令，从而实现当我们本地机器访问外部网络的时候，请求数据首先经过本的1080端口进行加密拆分为原始的TCP数据包。然而需要注意的是，当我们监听了默认的1080端口之后，并不意味着，我们访问数据的时候，所有网络数据就走1080这个ss客户端监听的端口了。由于Windows和macOS客户端通常包含了相关的代理转发功能，所以我们在进行HTTP或者是HTTPS的时候，会默认的启动一个代理转发的模块，来替我们监听HTTP和HTTPS请求，并将这些请求数据送入1080端口应用（即ss客户端）。

然而，在Linux下的ss客户端，并没有包含这个功能，所以，但我们进行HTTP(s)请求的时候，需要设置代理，当我们使用Privoxy的时候，安装Privoxy：
```shell
$ sudo yum -y install epel-release
$ sudo yum -y install privoxy
```
第一步的安装epel这是因为像centos这类衍生出来的发行版，他们的源有时候内容更新的比较滞后，或者说有时候一些扩展的源根本就没有。EPEL(http://fedoraproject.org/wiki/EPEL) 是由 Fedora 社区打造，为 RHEL 及衍生发行版如 CentOS、Scientific Linux 等提供高质量软件包的项目。

默认的配置文件地址在 /etc/privoxy/config 目录下。
```shell
# 把本地HTTP流量转发到本地1080 SOCKS5代理
# 即当我们发出HTTP(S)请求的时候，会将请求转为Socks5
# 注意最后的 "."
forward-socks5 / 127.0.0.1:1080 .
# 可选，默认监听本地连接端口为8118
# listen-address 127.0.0.1:8118
```
接下里启动：
```shell
systemctl start privoxy
```
配置完成以后还没有结束！请考虑一下，目前我们只是设定了如果有对于所有HTTP(S)的访问（即foward-sock5后面的“/”）都将转为socks5送入被ss客户端监听的1080端口中去，但是并没有设定HTTP(S)代理服务器！，所以，最后你还需要在.bash_profile（或者是.bashrc）中设定代理服务器的环境变量：
```shell
export http_proxy=http://127.0.0.1:8118
export https_proxy=https://127.0.0.1:8118
# 最后再source一下
```
最后，我们再以一个整体拓扑来表达一下我们的访问流程：
![flow](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2017-04-26-ss/flow.png)
