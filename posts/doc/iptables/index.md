# Iptables屏蔽所有国外IP!网络安全的利器！


# iptables简介

netfilter/iptables（简称为iptables）组成Linux平台下的包过滤防火墙，与大多数的Linux软件一样，这个包过滤防火墙是免费的，它可以代替昂贵的商业防火墙解决方案，完成封包过滤、封包重定向和网络地址转换（NAT）等功能。

#  **iptables基础**


​    规则（rules）其实就是网络管理员预定义的条件，规则一般的定义为“如果数据包头符合这样的条件，就这样处理这个数据包”。规则存储在内核空间的信息 包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如TCP、UDP、ICMP）和服务类型（如HTTP、FTP和SMTP）等。当数据包与规 则匹配时，iptables就根据规则所定义的方法来处理这些数据包，如放行（accept）、拒绝（reject）和丢弃（drop）等。配置防火墙的 主要工作就是添加、修改和删除这些规则。

# **iptables命令的管理控制选项**

```
-A 在指定链的末尾添加（append）一条新的规则
-D  删除（delete）指定链中的某一条规则，可以按规则序号和内容删除
-I  在指定链中插入（insert）一条新的规则，默认在第一行添加
-R  修改、替换（replace）指定链中的某一条规则，可以按规则序号和内容替换
-L  列出（list）指定链中所有的规则进行查看
-E  重命名用户定义的链，不改变链本身
-F  清空（flush）
-N  新建（new-chain）一条用户自己定义的规则链
-X  删除指定表中用户自定义的规则链（delete-chain）
-P  设置指定链的默认策略（policy）
-Z 将所有表的所有链的字节和数据包计数器清零
-n  使用数字形式（numeric）显示输出结果
-v  查看规则表详细信息（verbose）的信息
-V  查看版本(version)
-h  获取帮助（help）
```

# **防火墙处理数据包的四种方式**

ACCEPT 允许数据包通过
DROP 直接丢弃数据包，不给任何回应信息
REJECT 拒绝数据包通过，必要时会给数据发送端一个响应的信息。
LOG在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则

# iptables 官方文档

[https://netfilter.org/](https://netfilter.org/)



# 屏蔽外网IP

下面介绍一种方法只有国内的IP才能连接服务器，对防火墙建设有参考价值

## 背景

在我刚接触云产品时，使用的是腾讯云赠送的一个月云服务器。这一个月时间里我似乎也没部署过什么项目，就照着书上和网上的资料稍微捣鼓了下Ubuntu，但是没有使用网络服务的。可就在短短一个月内，有次上机检查，就发现cpu占用率99%，接着腾讯云给我发了条消息，说是中了挖矿病毒。查看日志后，来自卢森堡的一个IP成功登录。

后来我在阿里云租了个云服务器，在上面部署了C/S网络通信类的程序。刚开始网络编程功底不熟，没有对各种异常处理，如果没接收到预期格式的数据包就会报错。我使用的是一个很隐蔽的端口，但是经常会收到全球各地的连接，只要对方一发送数据包，服务器执行的程序就会报错退出。

再后来，我写了个网站，日志中经常能看到会有很多来自国外的爬虫进来逛。那么有没有什么方法屏蔽所有国外的连接呢？

## 建立一条规则链

创建一条规则链mylink，加到入站的规则中

```bash
iptables -N mylink
iptables -A INPUT -j mylink
```



## 获取国内所有的IP

获取所有国内的ip网段，保存到china_ssr.txt文件中

```bash
wget -q --timeout=60 -O- &#39;http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest&#39; | awk -F\| &#39;/CN\|ipv4/ { printf(&#34;%s/%d\n&#34;, $4, 32-log($5)/log(2)) }&#39; &gt; /root/china_ssr.txt
```

## shell脚本

观察ip网段，使用的是网络前缀格式，正好满足iptables的命令格式。用vim t跳转到末尾，发现竟然有8600多行国内网段。那么首先排除一个个手动输入命令的可能，于是我写了个脚本。

```bash
while read line
do
   iptables -A mylink -s $line -j ACCEPT
done &lt; china_ssr.txt
```

上面的脚本逐行读取文件的内容，然后执行命令。经过测试，发现iptables的优先级是自顶向下的，即当前规则必须是上面规则的子集，不然就无效。现在已经把所有国内IP允许访问，接下来就禁止全网IP。

```bash
iptables -A mylink -j DROP
```

上面这条命令极其危险。一定要放在链的末尾。~~本人就是输入了这条命令，导致无法连接到云服务器，无奈之下到阿里云回滚两个月前的快照。~~



---

> 作者: ZergFlood  
> URL: https://careltian.github.io/posts/doc/iptables/  

