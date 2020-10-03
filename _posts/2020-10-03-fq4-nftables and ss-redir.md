---
layout: post
title: LET'S FQ4 - nftables 入门（基础）
description: 
summary: Options to learn to code online.
tags: [GFW]
---

#### 1、适配ss-redir转发的nftables脚本：

- 为了方便描述，本脚本仅针对Shadowsocks进行了端口转发操作，没有添加其他任何防火墙规则，因此如果需要添加防火墙安全规则的话，请自行深入研究nftables的一些例子，参考资料：

  https://wiki.archlinux.org/index.php/Nftables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

- `# cat /root/my-ruleset`显示如下：

```bash
flush ruleset
table ip nat {
#创建一个集合：direct-list，这是一个IP地址集合，这些保留地址不需要走SS通道
#只需要注意123.123.123.123处，此处需要填写你的SS远程IP，其他按例默认即可
set direct-list {
	type ipv4_addr
	flags interval
	elements = { 123.123.123.123, 0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16, 172.16.0.0/12, 192.168.0.0/16, 224.0.0.0/4, 240.0.0.0/4 }
}

#自定义链:SHADOWSOCKS
chain SHADOWSOCKS {
	#目标ip如果在direct-list集合里，return（不做处理）：
	ip daddr @direct-list return
	#否则全部转发到ss-redir的3333端口做翻墙处理：
	meta l4proto tcp redirect to :3333
}

#本机输入链：默认放行
chain INPUT {
	type filter hook input priority filter; policy accept;
}

#转发链：默认放行
chain FORWARD {
	type filter hook forward priority filter; policy accept;
}

#本机输出链：默认放行
chain OUTPUT {
	type filter hook output priority filter; policy accept;
}

#预路由链：默认放行
chain PREROUTING {
	type nat hook prerouting priority dstnat; policy accept;
}

#后路由链：默认做NAT处理
chain POSTROUTING {
	type nat hook postrouting priority srcnat; policy accept;
	#masquerade(伪装)，是SNAT的一种特殊情况，其中源地址自动设置为输出接口的地址。
	#简化地说就是SNAT的智能版。
	masquerade
	}
}
```

- 将以上内容粘贴复制成一个文件，比如`/root/myruleset` ，唯一需要修改的地方就是将123.123.123.123改为你自己的SS远程服务器地址。

- 文件创建完毕后，通过 ``nft -f /root/my-ruleset` 命令加载为nftables规则。

- 通过`nft list ruleset` 查看，刚才导入的规则已经生效了（nftables会自动将注释部分取消），显示如下：

  ```bash
  table ip nat {
          set direct-list {
                  type ipv4_addr
                  flags interval
                  elements = { 0.0.0.0/8, 10.0.0.0/8,
                               123.123.123.123, 127.0.0.0/8,
                               169.254.0.0/16, 172.16.0.0/12,
                               192.168.0.0/16, 224.0.0.0/4,
                               240.0.0.0-255.255.255.255 }
          }
  
          chain SHADOWSOCKS {
                  ip daddr @direct-list return
                  meta l4proto tcp redirect to :3333
          }
  
          chain INPUT {
                  type filter hook input priority filter; policy accept;
          }
  
          chain FORWARD {
                  type filter hook forward priority filter; policy accept;
          }
  
          chain OUTPUT {
                  type filter hook output priority filter; policy accept;
          }
  
          chain PREROUTING {
                  type nat hook prerouting priority dstnat; policy accept;
          }
  
          chain POSTROUTING {
                  type nat hook postrouting priority srcnat; policy accept;
                  masquerade
          }
  }
  ```

- 此时，局域网分机已经可以正常共享上网了（比如访问qq.com），但是仍然不能翻墙（例如无法访问google.com），这是为何？

- 细心的你一定会发现，上面那个脚本，`Shadowsocks链`是一个孤立的存在，整个脚本中并不存在类似`jump shadowsocks`这样的跳转语句，那么它目的何在？

  这说起来就比较沮丧了，因为nftables是一个比较新的防火墙前端框架，因此很多旧软件都无法与它联动，这里面就包括我们前面的**dnsmasq** 和 **ipset**，这两个软件是我们智能翻墙的核心部分，但很可惜他们都只支持旧版的**iptables**，不支持**nftables**。

- 也许是考虑大量软件还存在兼容性问题（例如ipset只能跟iptables搭配使用），因此新系统（比如Centos 8）并没有彻底遗弃iptables，换句话说，iptables和nftables目前是可以混用的。

- 回到我们的案例，此时我们必须访问ipset集合，但nftables又没办法直接访问它，那么我们还得把iptables这个老家伙再祭出来用用：

  ```bash
  iptables -t nat -I PREROUTING -p tcp -m set --match-set gfwlist dst -j SHADOWSOCKS
  ```

  这个命令的拆解如下：

  - `-t nat -I PREROUTING` : 表nat，链PREROUTING
  - `-p tcp` : 协议tcp
  - `-m set --match-set gfwlist`: 匹配ipset集合gfwlist中的所有ip地址
  - `dst` : destination, 目标地址
  - `-j SHADOWSOCKS` : 跳转到SHADOWSOCKS

- 通过这条命令，我们会发现其实老版本的iptables可以与nftables共享信息，比如表nat是nftables创建的，SHADOWSOCKS是nftables的一个自定义链，这些项目iptables都可以直接访问。

- 敲入这条命令后，nftables这边也为其自动添加了一条规则，现在我们来看看nftables发生了什么变化，输入：

  ```bash
  nft list ruleset
  ```

  其他内容都没有变，请注意，PREROUTING链新增了一条规则：

  ```bash
          chain PREROUTING {
                  type nat hook prerouting priority dstnat; policy accept;
                  meta l4proto tcp # match-set gfwlist dst jump SHADOWSOCKS
          }
  ```

  多了一行： **meta l4proto tcp # match-set gfwlist dst jump SHADOWSOCKS**

  这一行规则是当我们敲入上面那条iptables命令的同时，nftables为我们自动生成的，经我的实际观察，此规则貌似只能通过nftables自动生成，没办法通过脚本方式反推回去（或许是我没找到方法）。但无论如何，能解决问题就行了，通过这种迂回的方法相当于在nftables和ipset之间建立了一条通道，让原本不兼容的二者至少可以做到单向访问。尽管处理方式不太优雅，但从结果而言确实是可行的，我也没有感觉在效率上有什么损失。

- 好，现在咱们回到局域网分机上，此时已经可以正常访问google.com了，整个智能翻墙工作完成！

----

#### 2、收尾工作

- 设置开机自启动

  因为整个流程有许多孤立的命令需要输入，因此有必要将其设为开机自启动以方便使用：

  - 创建启动脚本：

    `nano /root/autoexec.sh`

    内容如下：

    ```bash
    #!/bin/bash
    ipset create gfwlist hash:net
    nohup ss-redir -c /root/ss.json --plugin obfs-local --plugin-opts "obfs=http;obfs-host=baidu.com" > /root/ss.log 2>&1 &
    nft -f /root/my-ruleset
    iptables -t nat -I PREROUTING -p tcp -m set --match-set gfwlist dst -j SHADOWSOCKS
    
    [存盘退出]
    ```

    这个脚本很简单，解释如下：

    - 第一步：创建ipset的gfwlist集合
    - 第二步：启动ss-redir
    - 第三步：导入nftables规则脚本
    - 第四步：输入一条iptables命令，让nftables能访问ipset

  - 修改启动脚本权限：

    `chmod +777 autoexec.sh`

  - 创建系统服务：

    `nano /usr/lib/systemd/system/autoexec.service`

    内容如下：

    ```bash
    [Unit]
    Description=Autoexec
    After=network.target
    
    [Service]
    Type=forking
    ExecStart=/root/autoexec.sh
    PrivateTmp=true
    
    [Install]
    WantedBy=multi-user.target
    
    [存盘退出]
    ```

    输入以下命令：

    ```bash
    systemctl daemon-reload
    systemctl start autoexec
    systemctl enable autoexec
    ```

  - 大功告成，以后每次开机系统都将自动运行/root/autoexec.sh这个脚本，省得每次都要敲一堆命令。

- 最后，再用一张图大致说明一下本章节的总体流程：

  ![](D:\Sync\Code\metalage303.github.io\_posts\images\nft-05.png)

