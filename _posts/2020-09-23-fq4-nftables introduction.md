---
layout: post
title: LET'S FQ4 - nftables 入门
description: 
summary: Options to learn to code online.
tags: [GFW]
---

#### 1、关于Linux防火墙

在Linux系统里，防火墙并非只负责网络安全，只要跟数据包相关的事务都与之有关，比如说局域网内共享上网，尽管听上去和网络安全无关，但也是通过防火墙命令来实现的。

我相信大家经常会见到一些诸如Netfilter、Iptables、Firewalld、nftables 这样的命令，他们都属于防火墙相关命令，那么这几个命令之间究竟是什么关系呢？

- Netfilter 、 Iptables/Nftables 、 Firewalld 四者关系:

  - Netfilter：运行在Linux的核心层，它是处理数据包的基础框架，可以理解为最底层；

  - Iptables：是一个可以操作Netfilter的工具，可以理解为中间层；

  - Nftables：与Iptables平级且作用相当，但是它比前者更新、更时髦；

  - Firewalld：位于最顶层的前端工具（支持图形和命令行），通过调用在后端运行Iptables（或Nftables）让防火墙配置及使用变得更容易。

  - 防火墙在Linux系统中的演变：
    - 最早：只有两层Iptables -) Netfilter
    - 后来：为了简化操作，添加了Firewalld，变成了3层：Firewalld -) Iptables -) Netfilter
    - 现在：新版本Linux系统中，基本结构不变，还是3层：Firewalld -) Nftable -) Netfilter（请注意Iptables被替换成更时髦的Nftables）
    
  - PS: 在三层结构中，位于顶层的Firewalld只是为了让防火墙操作更简单和人性化，它并非防火墙系统的必需品，如果操作者对中间层（Iptables或Nftwable）的配置很熟悉，可以关闭Firewalld，纯粹用iptables/nftables搭建防火墙，其效果一样。
  
  ![](https://raw.githubusercontent.com/metalage303/metalage303.github.io/master/_posts/images/nft-01.png)



- 由于iptables已经处于被淘汰的边缘，本节只讨论最新的nftables，并且本节也不讨论firewalld，如果对这二者感兴趣的同学，请自行参考其他攻略。

----

#### 2、Nftables 基础

- 一个完整的nftables防火墙规则由簇（Family）、表（Tables）、链（Chains）、规则（Rules）组成。

- 基本结构：簇 ) 表 ) 链 ) 规则

  - **簇（Family）** ， 由5个基本簇组成：
    - ip: 匹配ipv4数据包
    - ip6: 匹配ipv6数据包
    - inet: 同时包含了IPv4 和 IPv6
    - arp: 匹配地址解析协议数据包 (ipv4地址和mac地址的映射)
    - bridge ：匹配网桥设备的数据包
    - netdev : 这是以前没有的新的簇，位于数据包入站通道非常靠前的位置，主要用于防止DDOS攻击

  ----

  - **表（Tables）**：一个容器，用于存放规则链。

  ----

  - **链（Chains）**：指规则链，一条链上可以挂靠多条规则，链分两种类型：

    - 常规链： 常规链就是指自定义链，名字可以随便定义，起作用主要是从逻辑上对规则进行分类。

    - 基本链 : 总共有5个，简单地说就是5座位置不同的关卡，注意他们的名字是预设好的，不可随意更改：
      - `prerouting`：预路由链，这时候数据包刚刚达到，未进入本机系统，也未对其进行转发。

      - `input`：输入链，准备进入本机的数据包。

      - `forward`：转发链，如果数据包不属于本机，则安排转发

      - `output`：输出链，从本地主机传出的数据。

      - `postrouting`：后路由链，数据包离开时的最后一道关卡。

        ![](https://raw.githubusercontent.com/metalage303/metalage303.github.io/master/_posts/images/nft-02.png)

        - 数据包的流向概括：

          1. 首先进入 `PREROUTING `（预路由）

          2. 进行路由判断，这时候出现两个分支：

             分支1. 数据包如果属于本地主机的，走`INPUT`（输入） 通道：

             - 进入`INPUT`的数据包，经过本机处理后，流向 `OUTPUT` (输出) 通道；
             - 从`OUTPUT`出来的数据，流向`POSTROUTING`, 出站

             分支2. 数据包不是本地主机的，流向`FORWARD`(转发) 通道

             - 从`FORWARD`出来的数据，流向`POSTROUTING`, 出站
  
  ----

  - **规则（Rules）**: 规则由语句或表达式构成，包含在链中，规则最终决定了对数据包采取什么动作：

    - accept：接受数据包。
    - drop：丢弃数据包。
    - queue：将数据包排队到用户空间。
    - continue：使用下一条规则继续进行规则评估。
    - return：从当前链返回并继续执行最后一条链的下一条规则。
    - jump ：跳转到指定的规则链，当执行完成或者返回时，返回到调用的规则链。
    - goto ：类似于跳转，发送到指定规则链但不返回。

    以上这些动作，可用匹配的部分列表如下：
    
    - meta （元属性，如接口）
      - oif (output interface INDEX)
      - iif (input interface INDEX)
      - oifname (output interface NAME)
      - iifname (input interface NAME)
        （oif 和 iif 接受字符串参数并转换为接口索引）
        （oifname 和 iifname 更具动态性，但因字符串匹配速度更慢）
    - icmp （ICMP协议）
      - type (icmp type)
    - icmpv6 （ICMPv6协议）
      - type (icmpv6 type)
    - ip （IP协议）
      - protocol (protocol)
      - daddr (destination address)
      - saddr (source address)
    - ip6 （IPv6协议）
      - daddr (destination address)
      - saddr (source address)
    - tcp （TCP协议）
      - dport (destination port)
      - sport (source port)
    - udp （UDP协议）
      - dport (destination port)
      - sport (source port)
    - sctp （SCTP协议）
      - dport (destination port)
      - sport (source port)
    - ct （链接跟踪） state (new / established / related / invalid)

----

#### 3、Nftables 实例

如果你对上面的知识点有点迷迷糊糊的，不要紧，现在来看这个实例：

- 第1步，建表：
  `nft add table inet my_table `

  建一个基于inet簇的表，表名：my_table

- 第2步，建链：
  `nft add chain inet my_table my_chain`

  在 inet 簇的 my_table 表里建一个链，链名：my_chain

#### 3.1 增加规则

- 第3步，建几条规则：

  ```bash
  nft add rule inet my_table my_chain ip saddr 127.0.0.1 accept
  nft add rule inet my_table my_chain ip saddr 127.0.0.2 accept
  nft add rule inet my_table my_chain ip saddr 127.0.0.3 accept
  (这三条规则没有太大意义，仅供展示之用)
  ```

  第一条规则的含义是：在 inet 簇的 my_table 表 的 my_chain 链里，创建一条新的规则: 来自源ip (saddr) 127.0.0.1 的数据包，采取：放行（accept）策略。

- **Nftables的序号系统**，所有的nft表都是可以带序号查看规则的，在list后面加上 --handle 开关，可以将规则的序号显示出来：
  `nft --handle list chain inet  my_table my_chain`

  系统回显如下：

  ```bash
  table inet my_table {
          chain my_chain { # handle 1
                  ip saddr 127.0.0.1 accept # handle 2
                  ip saddr 127.0.0.2 accept # handle 3
                  ip saddr 127.0.0.3 accept # handle 4
          }
  }
  ```

  通过list命令，我们刚才建立的规则很直观的显示出来了，包括簇、表、链、规则。

  你会发现每一条规则后面都跟了一个handle (句柄，可以理解为序号），这个序号主要用于对规则进行增、删、改操作。

  【小窍门：带序号查看全部表用这个命令：
  
  ```bash
  nft --handle list ruleset 
  ```
  
  这是一个高频率命令，你可以把它做成一个alias方便使用】
  
- **insert 命令**，这个命令和**add** 命令差不多，都是起增加规则的作用，比如我们在2号规则前面加一条新规则：
  `nft insert rule inet my_table my_chain handle 2  ip saddr 127.0.0.99 accept`

  再用`nft --handle list chain inet  my_table my_chain` 命令看看，系统回显如下：

  ```bash
  table inet my_table {
          chain my_chain { # handle 1
                  ip saddr 127.0.0.99 accept # handle 5
                  ip saddr 127.0.0.1 accept # handle 2
                  ip saddr 127.0.0.2 accept # handle 3
                  ip saddr 127.0.0.3 accept # handle 4
          }
  }
  ```

  注意：在2号规则上面，新增了一条规则，序号为5，此处也可以采用nft add 命令去新增规则，与insert不同的是：**net add总是在链的末端添加规则，net insert 则可以在指定序号的前面添加规则**。

#### 3.2 删除规则

- **delete 命令**，下面演示一下如何删除规则，比如现在删除第3号规则：
  `nft delete rule  inet my_table my_chain handle 3`

- 然后用`nft --handle list chain inet  my_table my_chain` 命令看看，系统回显如下：

  ```bash
  table inet my_table {
          chain my_chain { # handle 1
                  ip saddr 127.0.0.99 accept # handle 5
                  ip saddr 127.0.0.1 accept # handle 2
                  ip saddr 127.0.0.3 accept # handle 4
          }
  }
  ```

  注意：第3号规则消失了，已被删除。

  - 删除可以用nft delete或者nft flush来完成，从大到小按照： **表 > 链 > 规则** ，有着不同的效力：
    - 微（删规则）：nft delete rule  inet my_table my_chain handle 3 （删除my_table\my_chain链中的3号规则）
    - 小（清空规则）：nft flush chain inet my_table my_chain （清空my_table\my_chain链的所有规则）
    - 中（删链）：nft delete chain inet my_table my_chain （删除my_table\my_chain链）
    - 大（清空链）：nft flush table inet  my_table （清空my_table表中的所有链）
    - 巨（删除表）：nft delete table inet my_table  (删除my_table表)
    - 完全清空：nft flush ruleset (清空所有表)

#### 3.3 修改规则

- **replace 命令**，下面我们展示一下如何修改一条已存在的规则，输入命令：

  ` nft replace rule inet my_table my_chain handle 2 ip saddr 127.0.0.9 accept`

  利用nft replace rule命令，将第2号规则改为：ip saddr 127.0.0.9 accept

- 然后用`nft --handle list chain inet  my_table my_chain` 命令看看，系统回显如下：

  ```bash
  table inet my_table {
          chain my_chain { # handle 1
                  ip saddr 127.0.0.99 accept # handle 5
                  ip saddr 127.0.0.9 accept # handle 2
                  ip saddr 127.0.0.3 accept # handle 4
          }
  }
  ```

  注意：2号规则已经变成127.0.0.9 （之前是127.0.0.1）

----

#### 4. 课外阅读：NAT实例

以上例子展示了nftables的基础用法，其实掌握到这一步，已经能对nftables有一些大致的轮廓了，至少对于翻墙这个目标而言基本足够。下面再举一个更复杂的例字，可以加深大家对nftables和基本NAT原理的一些认识。（PS：此节为延申阅读，可以略过）

- NAT简介
  NAT英文全称是“Network Address Translation”，中文意思是“网络地址转换”，分为3大类别：

1. 静态NAT
2. 动态NAT
3. 网络地址端口转换NAPT

其中第三种适用于只有一个IP地址有需要有多台机器需要上网的场合，本文只讲第三种。
本例的初始环境如下，现有网关一台：

- 公网IP：202.96.112.113
- 内网IP：192.168.1.1

**NAPT的NAT分为SNAT和DNAT两种**：
为了讲清楚这部分，我们先看看一个数据包究竟是怎么在互联网上的流动的：

- 网关直接访问外网，**无需NAT**

  很简单，比如现在我们直接在网关机器上访问qq.com：
  网关自带公网IP：202.96.112.113:1234 （1234是一个随机端口），向qq.com (58.247.214.47:80) 发起一个WEB请求，这个发送包的状态是：
  
  - （(font color=#0000FF)目标：58.247.214.47:80 源：202.96.112.113:1234(/font)）
  
  QQ服务器处理完毕后，回传数据，这个回传包的状态是：
  
  - （(font color=#0000FF)目标：202.96.112.113:1234 源：58.247.214.47:80(/font)）
  
  数据包传输结束。

但是，如果整个局域网只有一个公网IP，局域网有多台机器需要同时上网的时候就要稍微复杂一点了。

----

#### SNAT

- 局域网分机访问外网，需要进行 **“源地址转换”（SNAT）**

  现有局域网机器192.168.1.2 ，192.168.1.3

  当他们同时访问qq服务器58.247.214.47的时候，数据需要通过SNAT进行转换：

**数据包从两台不同的局域网分机 发送 到QQ.COM的流程如下(PS: SRC=源 DST=目标)：**

![](https://raw.githubusercontent.com/metalage303/metalage303.github.io/master/_posts/images/nft-03.png)

- 192.168.1.2:2345 发出一个数据包到 58.247.214.47:80（QQ服务器,80端口）

- 192.168.1.3:2345 发出一个数据包到 58.247.214.47:80（QQ服务器,80端口）

- 网关将其源IP修改为网关的公网IP，又因为两台分机的端口都是2345，所以网关必须为其分配两个不同的端口，修改完毕后，首先将这个对应关系记录到**Connection Track Table**，然后将数据包发送到QQ.COM

  ----

**当QQ.COM收到这两个数据包后，数据回传到局域网分机的流程如下：**

![](https://raw.githubusercontent.com/metalage303/metalage303.github.io/master/_posts/images/nft-04.png)

- 数据包**接收**过程其实就是其**发送**的逆过程，网关根据之前在**Connection Track Table** 中记录的信息，将数据包还原，然后发送回局域网分机，具体见图，这里就不赘述了。

----

#### 用nftables实现SNAT:

- 别看上述流程一大堆，其实对nftables来说也就一条规则：

  ```bash
  nft 'add rule ip nat postrouting ip saddr 192.168.1.0/24 snat 202.96.112.113'
  ```

- 我们把这行规则拆开来解释：

  - **add rule  ip nat postrouting**：是指在ip簇、nat表、postrouting 链上添加一个规则，`postrouting` 是一个内置链，表示"后路由"，所有数据包准备出站前的最后一道关口；
  - **ip saddr 192.168.1.0/24** ：是指匹配整个IP网段192.168.1.X的所有数据包；
  - **snat 202.96.112.113**： 将匹配到的数据包的”源地址“（snat），全部修改为网关地址 202.96.112.113
  
  其中，包括端口转换、 Connection Track Table 的记录与查询等等这些操作nftables都已经自动帮我们完成了，可以说已经是非常简化了。

----

### DNAT

-  **“目标地址转换”（DNAT）**, 用于外网访问局域网内部分机的场景，比如局域网有一台WEB服务器，但这台服务器又不是置于网关位置，而是安装在局域网某台分机的时候，外网主机是不能直接访问它的，这时候就需要用到DNAT了。

- 因为DNAT 跟我们的主题 “翻墙” 其实关系不大，所以本文不打算对其进行详细描述，只大略讲一下其实现。

- 用nftablaes来实现DNAT，建立规则：

  ```bash
  nft 'add rule nat prerouting iif enp1s0 tcp dport { 80, 443 } dnat 192.168.1.2'
  ```

- 解释：

  - **add rule ip nat prerouting** ：是指在ip簇、nat表、prerouting 链上添加一个规则，`prerouting` 是一个内置链，表示"预路由"，是所有数据包准备入站前的第一道关口。
  - **iif enp1s0** ： 匹配外网网卡enp1s0的进来的流量 （iif是input interface的缩写，即输入流量界面）
  - **tcp dport { 80, 443 }**：匹配tcp协议的80，443端口 （dport是destination port的缩写，即目标端口）
  - **dnat 192.168.1.2**  ： 将匹配到的数据包的”目标地址“（dnat），全部转发到局域网分机 192.168.1.2 上去


----

#### nftables 脚本

- **将脚本导入成规则**：

- nftables除了使用nft命令逐条创建规则之外，还支持脚本，例如我们现在创建一个规则脚本：

  `nano /root/test.nft`

  输入以下内容：

  ````bash
  table inet my_table {
          chain my_chain {
                  ip saddr 127.0.0.1 accept 
                  ip saddr 127.0.0.2 accept 
                  ip saddr 127.0.0.3 accept
          }
  }
  [存盘退出]
  ````

- 先清空所有规则：

  `nft flush ruleset`

- 再将我们刚才创建的脚本导入为nftables规则：

  `nft -f /root/test.nft`

- 查看一下：

  `nft list ruleset`

  系统将回显：

  ```bash
  table inet my_table {
          chain my_chain {
                  ip saddr 127.0.0.1 accept
                  ip saddr 127.0.0.2 accept
                  ip saddr 127.0.0.3 accept
          }
  }
  ```

  刚才创建的规则脚本现在已经成为规则并生效了。

  利用脚本创建nftables规则的好处是便于持久化，因为用nft命令创建的规则当系统重新启动后将会被清空，使用脚本则没有这个问题，只需要利用 **nft -f** 指令即可将规则还原。

- **将规则导出成脚本：**

  我们接着上一个例子继续，再为其添加一个规则：

  `nft add rule inet my_table my_chain ip saddr 127.0.0.4 accept`

- 将当前规则导出成脚本：

  ```bash
  echo -n "" > /root/test.nft (首先清空test.nft)
  nft -s list ruleset >> /root/test.nft (将当前规则导出为脚本文件)
  ```

  查看脚本文件：

  `cat /root/test.nft `

  系统将回显：

  ```
  table inet my_table {
          chain my_chain {
                  ip saddr 127.0.0.1 accept
                  ip saddr 127.0.0.2 accept
                  ip saddr 127.0.0.3 accept
                  ip saddr 127.0.0.4 accept
          }
  }
  ```

  我们刚才创建的新规则 ip saddr 127.0.0.4 accept ，已经被保存在了这个脚本文件里。

- **nftables默认的启动脚本：**

  - 要开机自动运行nftables脚本，首先需要开启nftables服务：

    ```bash
    systemctl enable nftables
    systemctl start nftables
    ```

  - nftables的默认启动脚本是这个文件：

    ```bash
    nano /etc/sysconfig/nftables.conf
    ```

    自定义规则可以添加在这个文件的尾部，这样每次开机后nftables就会自动生成这些规则了。

----

参考资料：

- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/getting-started-with-nftables_configuring-and-managing-networking
- https://hezhiqiang8909.gitbook.io/nftables/
- https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes

