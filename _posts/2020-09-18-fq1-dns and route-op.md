---
layout: post
title: 翻墙教程（2）- DNS污染的原理以及应对策略（实战）
description: 
summary: Options to learn to code online.
tags: [GFW]
---

#### 1、基本架构：

- 环境说明：本文运行环境是Centos 8.2,  官网：https://www.centos.org/ 
  
- 可以在官网下载ISO映像，将映像刻录到U盘启动，在实体机或虚拟机安装皆可。
  
- 本节需要配置两个DNS服务：
  
  - **dnsmasq** 为主DNS服务器，负责国内域名解析，通过阿里DNS获得IP，一旦发现国外域名，转交给dnscrypt-proxy处理
  
  - **dnscrypt-proxy** 为辅DNS服务器，负责加密传输，获取真实的境外IP
  
- 首先安装相关依赖包、工具：
  
  ````bash
  dnf install epel-release -y
  dnf install git -y
  dnf install -y gcc gettext autoconf libtool automake make pcre-devel asciidoc xmlto udns-devel c-ares-devel libev-devel libsodium-devel mbedtls-devel net-tools wget bind-utils nano
  ````

#### 2、配置 dnscrypt-proxy：

- 使用dnf安装dnscrypt-proxy：

  ```bash
  dnf install -y dnscrypt-proxy
  ```

- 修改dnscrypt-proxy主配置文件：

  ```bash
  nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml
  ```
  注意这行, 确保中括号里面不要有任何内容即可:
  ```
  listen_addresses = [] 
  [存盘退出]
  ```

  修改dnscrypt-proxy的socket配置文件:

  ```bash
  nano /usr/lib/systemd/system/dnscrypt-proxy.socket
  ```

  将Socket这一节整体替换为下面这样：
  ```
  [Socket]
  ListenStream=
  istenDatagram=
  ListenStream=127.0.0.1:30053
  ListenStream=[::1]:30053
  ListenDatagram=127.0.0.1:30053
  ListenDatagram=[::1]:30053
  [存盘退出]
  ```
- 文件都修改完毕后，执行以下命令让他们开机自启：

  ```bash
  systemctl start dnscrypt-proxy.socket  
  systemctl enable dnscrypt-proxy.socket
  systemctl start dnscrypt-proxy.service
  systemctl enable dnscrypt-proxy.service
  ```

#### 3、配置 dnsmasq：

- 使用dnf安装dnsmasq

  ```bash
  dnf install epel-release -y
  dnf install -y dnsmasq
  systemctl start dnsmasq
  systemctl enable dnsmasq
  ```

- 修改dnsmasq配置文件:

  ```bash
  nano /etc/dnsmasq.conf
  ```
  
  ```
  listen-address=127.0.0.1,192.168.1.1 (监控本机，以及内网IP地址)
  expand-hosts (删掉这行前面的#号，即: 取消注释让其生效）
  domain=rockage.lan （设置局域网域名）
  server=223.5.5.5 （因为dnsmasq对应的是国内域名，因此我们采用阿里DNS）
  server=223.6.6.6
  address=/rockage.lan/192.168.1.1 (将域名与IP绑定)
  dhcp-range=192.168.1.50,192.168.1.100,12h （为局域网的机器自动分配从.50到.100的IP地址）
  [存盘退出]
  ```
- dnsmasq 配置文件语法检查, 如果显示Error表示上述修改内容有误, 请仔细检查:

  ```bash
  dnsmasq --test
  ```
  
- 设置dnsmasq和本机dns的关联

  > /etc/resolv.conf 文件是本机DNS的设置文件, 由本地守护程序（NetworkManager）维护，因此用户对这个文件所做的任何更改, 在系统重启之后都将被覆盖还原。解决这个问题的方法是给它加一把锁, 防止系统进程对它进行自动修改

- 首先解锁：

  ```bash
  chattr -i /etc/resolv.conf
  ```

- 解锁之后就可以修改了:

  ```bash
  nano /etc/resolv.conf
  ```

  删除这个文件的所有nameserver部分, 或者在前面加#号注释, 只留一行：

  ```
  nameserver 127.0.0.1 (将nameserver指向本机)
  [存盘退出]
  ```

- 编辑完成后需要加锁, 以防被系统进程自动修改:

  ```bash
  chattr +i /etc/resolv.conf
  ```

- 查看一下加锁状态: (PS: 下次如需修改此文件，需先解锁再编辑 )

  ```bash
  lsattr /etc/resolv.conf
  ```

- 接下来, 打通dnsmasq和本机预定义host的关联:
  本机的 /etc/hosts 文件预先存储了一些IP和域名的对应关系，需要将他们指向dnsmasq

  ```bash
  nano /etc/hosts
  ```

  ```
  在文件尾部加一行：
  127.0.0.1       dnsmasq
[存盘退出]
  ```
- 尽管不是必须, 但鉴于修改内容较多，建议重启一次机器:

  ```bash
  reboot
  ```

#### 测试

- 在测试之前首先安装测试工具Dig:

  ```bash
  dnf install bind-utils
  ```

- 输入以下命令测试:

  ```bash
  dig -x rockage.lan
  ```

  如果回显：**[SERVER: 127.0.0.1#53(127.0.0.1)]** 表示OK

- 在局域网上随便找台机器，

  ```
  IP设为192.168.1.X (X可以取值2-255), 
  网关设为: 192.168.1.1
  DNS 设为: 192.168.1.1
  然后输入命令: ping rockage.lan 
  如果回显:	**来自 192.168.1.1 的回复: 字节=32 时间<1ms TTL=64**，表示OK
  ```
  
  现在针对国内的DNS服务就已经设置完毕了，但是如果ping google.com之类的域名，那么结果肯定还是假的.

  本节参考文章: https://www.tecmint.com/setup-a-dns-dhcp-server-using-dnsmasq-on-centos-rhel/

#### 4、配置 gfwlist 

- gfwlist 是一个公益项目, 它将被GFW所屏蔽的网站做了一个汇总, 并以文件方式提供给大家下载。

  项目地址：https://github.com/gfwlist/gfwlist ，有兴趣的可以去围观点赞。

- dnsmasq-gfwlist.py 是一个Python 程序，它将原始的gfwlist文件转换为dnsmasq能读懂的格式。

- 首先下载 dnsmasq-gfwlist.py：

  ```bash
  wget https://gist.githubusercontent.com/lanceliao/85cd3fcf1303dba2498c/raw/7391429f8fdc5e3f4c82ba98b98767922b8bb473/dnsmasq-gfwlist.py
  ```

- 然后对文件进行一点修改：

  ```bash
  nano dnsmasq-gfwlist.py
  ```

  需要修改的内容不多，只需要将：

  ```
  mydnsport = '1053' 
  改为 
  mydnsport = '30053' 
  [存盘退出]
  ```
  
  PS: 端口号30053是我们之前设置好的dnscrypt-proxy的端口。另外，因为这个文件是Python 2.7写的，注意不能用python 3.8来运行，如果此时你的系统还没有安装Python 2.7,  可以用dnf命令先安装：
  
  ````bash
  dnf install -y python27
  ````

- 接着，开始编译执行dnsmasq-gfwlist.py：

  ```bash
  python2.7 dnsmasq-gfwlist.py
  ```

  注意：这一步需要在梯子已经建好的情况下，因为程序会自动去下载gfwlist文件，下载这个文件需要翻墙。

- 程序运行完毕，转换后的文件将自动保存在/etc/dnsmasq.d 里，先查看一下文件内容：

  ```bash
  cat /etc/dnsmasq.d/gfwlist.conf
  ```

  如果看到一大堆类似这样的内容：

  ```
  server=/instagram.com/127.0.0.1#30053      
  ipset=/instagram.com/gfwlist
  ```

  表示文件没有问题，创建成功了。

- 接着，还需要将它导入dnsmasq，编辑dnsmasq设置文件：

  ```bash
  # nano /etc/dnsmasq.conf 
  ```

  ```
  查找 "conf-dir=" 
  默认的应该是这样：
  conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
  或者这样：
  conf-dir=/etc/dnsmasq.d 
  只要保证conf-dir没有被注释，另外保证/etc/dnsmasq.d这个路径包含在设置清单内即可。
  [存盘退出]
  ```
  
- 退出后，再做一次语法检查确保配置文件无误：

  ```bash
  dnsmasq --test
  ```

- 最后重启dnsmasq并检查启动是否成功：

  ```bash
  systemctl restart dnsmasq 
  systemctl status dnsmasq
  ```

- 如果出现： dnsmasq: failed to create IPset control socket: Permission denied
  这个错误，这个多半是由于selinux没有关闭造成的，现在关闭selinux:

  ```bash
  nano /etc/selinux/config
  ```

  ```
  将： SELINUX=enforcing
  改为：SELINUX=disabled
  [存盘退出]
  ```

  修改selinux需要重启机器：

  ```bash
  reboot
  ```

  重启后，输入以下命令确认selinux已关闭：

  ```bash
  sestatus
  ```

- 如果出现： failed to create listening socket for port 53: Address already in use

  首先使用这个命令，查看一下到底是哪个程序占用了53端口：

  ```bash
  systemctl list-sockets
  ```

  如无意外，我猜大概率是跟 **dnscrypt-proxy.socket** 打架了，解决方法：

  - 请严格按照我上面的dnscrypt-proxy安装方法去做，诚然，主运行端口留空这一点可能会引起大家的疑惑，但dnscrypt-proxy官方文档写得很清楚，socket方式就是这样设置的。


  - 主要检查两个地方：


    - /etc/dnscrypt-proxy/dnscrypt-proxy.toml 文件中：**listen_addresses = []** （这个中括号一定要为空）
    
    - /usr/lib/systemd/system/dnscrypt-proxy.socket 文件中：**Socket** 一节原封不动复制粘贴我的配置
    
-  如果显示：Unable to retrieve source [public-resolvers]，这个问题倒不大，因为服务器在境外有时候会出现读取不了的情况，只需要重新restart一下dnscrypt-proxy服务，再用status查看一下，一般来说多刷几次就正常了。

- 检查完毕后，用 **reboot** 命令重启，重启后输入：

  ```bash
  systemctl status dnscrypt-proxy
  systemctl status dnsmasq 
  ```

    如果两者都显示绿字而没有任何红字，表示设置成功了！
    如果实在问题多多，就用这个命令仔细看看启动log，再具体分析:

  ```bash
  journalctl -r -u dnscrypt-proxy.service 
  ```

#### 5、设置 ipset
- 首先是建一个名为 "gfwlist" 的表:

  ```bash
  ipset create gfwlist hash:net
  ```

  注意这个gfwlist和上面我们说的gfwlist文件不是一回事, gfwlist 源文件通过转换后能够被dnsmasq识别，那么，通过dnsmasq 筛选出来的国外域名会转交给 dnscrypt-proxy 做加密解析，最终，这些解析后的IP存放在哪呢？就存放在现在咱们用ipset做的这个名叫gfwlist的表里了。

-  表建好之后，现在查看一下：

  ```bash
  ipset list gfwlist
  ```

  意料之中，目前表是空的，Members后面什么都没有。

- 现在输入：

  ```bash
  nslookup qq.com
  ```

  再输入：

  ```bash
  ipset list gfwlist
  ```

  啥变化也没有嘛，为何？因为这是一个国内网站，并没有被gfwlist收录

  接着输入：

  ```bash
  nslookup facebook.com
  ```

  再输入：

  ```bash
  ipset list gfwlist
  ```

  神奇的事情发生了！内容有变化，Members后面跟了一个IP，这就对了，因为这是一个被gfwlist收录的域名，所以被ipset过滤了进来！你还可以多试几个网站，比如google.com、twitter.com、youtube.com 等等你懂的系列网站，你会发现Members列表会越来越长。

- 最终需要做到：

  1. 用nslookup命令查询一个国内域名，gfwlist表中的Members不增加；
  2. 用nslookup命令查询一个被GFW墙了的域名，gfwlist表中的Members增加其对应的IP地址。
  

恭喜，本文的所有目标你都实现了！
