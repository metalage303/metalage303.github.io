---
layout: post
title: 翻墙教程（4） - 部署Shadowsockas透明代理服务器（实战）
description: 
summary: Options to learn to code online.
tags: [GFW]
---

#### 1、Shadowsocks-libev

shadowsocks-libev是原shadowsocks项目的一个分支，由madeye维护，github地址：

https://github.com/shadowsocks/shadowsocks-libev

这是一个由C语言重新编写的shadowsocks，自身集成了三端，即服务器端+客户端+转发端都在里面了，无论是搭建SS服务器，还是作为其他SS服务器的客户端，或者利用其来进行端口转发，都是用它。

特别提示: 在本篇中，主要是利用其ss-redir (转发) 功能，实现透明代理功能。至于如何利用其搭建SS服务器，本文不涉及，请参考其他攻略。

----
- 安装Shadowsocks-libev

  - PS: 因为需要到github克隆源代码，因此这一部可能需要借助梯子（看GFW心情，github并不是它的重点打击对象）

  - 输入命令：

  ```bash
  git clone https://github.com/shadowsocks/shadowsocks-libev.git
  cd shadowsocks-libev
  git submodule update --init --recursive
  ./autogen.sh 
  ./configure 
  make && make install
  ```
  
- 安装完毕后，三端的启动程序分别是：

  - **ss-server** : 作为SS服务器端的时候使用它，本文不涉及。
  - **ss-local** : 作为SS客户端的时候使用它，本文略微要用它做一下测试。
  - **ss-redir** : 利用它进行端口转发，实现透明代理，本文的主角。
  - **ss-tunnel** : 类似VPN的隧道，利用它也可以实现透明代理，但本文不采用此方案。

- 对于非Centos 8系统，或许还需要手动安装以下几个依赖，以下步骤Centos 8 用户可以忽略：

  提示: 前提是笔者默认你已经在前文安装dnsmasq的时候执行了整体依赖安装的步骤，如果出现Shadowsocks-libev无法编译的情况，强烈建议你翻回上一篇博客去看看。
  
  ```bash
  # 安装 Libsodium [CENTOS8略过]
  mkdir libsodium
  cd libsodium
  wget https://download.libsodium.org/libsodium/releases/LATEST.tar.gz
  tar xvf LATEST.tar.gz
  cd libsodium-stable
  ./configure --prefix=/usr && make
  make install
  ldconfig
  
  # 安装 MbedTLS  [CENTOS8略过]
  git clone https://github.com/ARMmbed/mbedtls.git
  cd mbedtls
  make SHARED=1 CFLAGS=-fPIC
  make DESTDIR=/usr install
  ldconfig
  ```
----

#### 2、安装simple-obfs 混淆插件

**simple-obfs** 是一个简单的混淆插件, 可以伪装成 `http` 流量。

- 安装simple-obfs，输入命令：

  ````bash
  git clone https://github.com/shadowsocks/simple-obfs.git
  cd simple-obfs
  git submodule update --init --recursive
  ./autogen.sh
  ./configure && make
  make install
  ````

----

#### 3、检测你的Shadowsocks服务器

- 全部安装完毕，先创建一个ss配置文件：

  ```bash
  nano /root/ss.json
  ```

  将以下内容复制粘贴进去：

  ```json
  {
      "server":"123.123.123.123",
      "server_port":1234,
      "local_address":"0.0.0.0",
      "local_port":3333,
      "password":"password",
      "timeout":600,
      "method":"aes-128-gcm"
  }
  [存盘退出]
  ```

  - `server` 填你的SS服务器地址，也可以是域名
  - `server_port` 填你的SS服务器端口
  - `local_address` 只能填写0.0.0.0，不可填127.0.0.1, 否则只能转发本机端口。
  - `local_port` 将远程端口映射到本地的端口号，为了和接下来的实验保持一致，这里就还是填3333好了
  - `password` 填你的SS服务器密码
  - `timeout` 默认600
  - `method` 填你的SS服务器加密协议，这个可以从你的SS服务器设置或者机场给你的设置文件中找到

- 全部安装完毕，先利用**ss-local**测试本机是否已经可以翻墙：

  - 输入以下命令：

    ```bash
    nohup ss-local -c /root/ss.json --plugin obfs-local --plugin-opts "obfs=http;obfs-host=www.baidu.com" > /root/ss.log 2>&1 &
    ```

    特别提示: 以上命令除了 **obfs-host** 外，其他不需要修改。这个obfs-host起混淆视听的作用，obfs插件利用http协议将我们的翻墙数据伪装好像我们不停地在访问baidu.com一样，是忽悠GFW的一种手段。在前几个版本，这个网址是可以随便填写的。但最近，我发现如果这个host如果不按机场发给你的指定配置去填写，ss-local就跑不起来，所以强烈建议按照机场给你的设置原封不动地将这个网址填进去。

  - 如果输入以上命令后，生成了一个pid，那么大概率就是运行成功了，我们接着测试：
  
    ```bash
    ps -A|grep ss-local (先看看有没有ss-local进程？)
    cat /root/ss.log (看看启动日志是否正常？)
    netstat -lnp  (看看3333端口是否启动？)
    ```
  - 如果上述测试都正常，说明本地配置基本没问题，最后我们让它去外网转一圈，看看究竟能不能工作：
  
    ```bash
    curl --socks5 127.0.0.1:3333 https://httpbin.org/ip 
    ```

    PS:因为ss-local是一个标准的socks5代理，正常情况下还需要通过Privoxy之类工具做二次代理将socks5协议转换为普通的HTTP协议以供上网使用，但此处curl命令能够直接使用socks5访问这个IP查询网站）

  - 如果curl命令回显：
  
    ```
    {
      "origin": "你的梯子的IP"
}
    ```
  
  - 恭喜你，这说明你的SS服务器可用，配置文件也没有任何问题，如果这一步出问题了，主要排查两点：
  
    - /root/ss.json 文件是否填写有误，是不是严格按照机场给你的配置填写的？
    - ss-local启动命令行里的obfs-host，也需要按照机场发给你的指定配置填写。


----

#### 4、配置ss-redir

- ss-redir转发设置其实和ss-local基本一致，前面能跑起来这一步就大概率不会出问题。

  `重要提示: 如果之前运行了ss-local，请务必用 ps -A|grep ss-local命令找到ss-local的进程pid，将之kill掉，否则和接下来的试验产生冲突！`

- 输入命令：

  ```bash
  nohup ss-redir -c /root/ss.json --plugin obfs-local --plugin-opts "obfs=http;obfs-host=www.baidu.com" > /root/ss.log 2>&1 &
  ```

  [^提示]:上面这行命令只是将 **ss-local** 替换成 **ss-redir**而已，其他部分完全相同。

- 我们直接用curl命令测试一下：

  ```bash
  curl -4vsSkL 127.0.0.1:3333 https://httpbin.org/ip 
  ```

  注意:这条命令跟前面的ss-local略有不同，因为ss-redir并非是一个socks5代理，它是透明的,  你可以理解它只负责将前端数据无脑转发到指定端口，与协议无关，因此此处的curl命令不需要添加--socks5标记)

- 如果curl命令回显：

  ```
  {
    "origin": "你的梯子的IP"
  }
  ```

- 恭喜你，这说明你的ss-redir配置成功了！

本节到此结束，下一节我们使用 nftables 进行分流处理，关于分流这部分，网上绝大多数文章都是以iptables为例来讲解的。但是iptables是一个很古老的路由管理工具。在Centos 8和一些新的Linux系统里基本是要被淘汰的节奏，所以我们也与时俱进一下，尝试采用它的继任者nftables来进行这个工作。 

**祝大家生活愉快~**