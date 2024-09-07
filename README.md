# MultiClients
> 本项目启发自部分校园网对学生设备的数量限制
> 
> 为了更好的利用网络（多几台设备），特编写此项目，希望提供一些思路

> Written by [@ELT](https://github.com/ELT17604)
>
> 感谢Github用户TianxinZhao的项目[HuiHuBuTong](https://github.com/TianxinZhao/HuiHuBuTong)及友好的讨论

***

## §0 如何正确办理校园网
具体请参考[此处](/Guides/banli.md)  
***仅供参考！具体以实际为准***

# §1 校园网自动登录的解决思路
部分校园网系统使用Portal登录，联网后若未登录便会跳出登录认证界面，若未跳出也可手动进入特定内网IP登陆。
非常烦人的是长时间无网路行为会自动下线造成下次使用不便。经研究，以下办法或许可行。
1. 本人正在使用的办法  
- Step1:  
使用抓包软件检测在登陆时发送的内容，于是发现所谓登录其实是向一内网IP进行了一个curl的发送，
包含了UA，账户(就是手机号)和密码，保存该条，测试发现直接发送可以登录。  

- Step2:  
在OpenWRT系统设置中设置当检测到断网时发送curl指令，可通过脚本实现。
或者，简单粗暴的方式就是在系统计划里每分钟执行一次curl指令，亲测一年下来正常使用，具体的语法应是
` * * * * * curl xxxxxxx `,其中*表示每天每小时每分钟执行一次。

2. 更简单粗暴的方式  
- 因为检测的是长时间无网络行为，于是可以后台一直ping某网址。
- 例如，在子网任何设备里24小时ping baidu.com，即可实现一次登录长久使用。

# §2 部分限制的原理
1. 设备数量限制  
- 最简单的检测方式是设备MAC号，此办法通过个人路由器开启DHCP即可绕过。

- 更高端一点办法就是检测TTL。
`所谓TTL就是用于防止数据包在互联网上无限传播采取的方式，在经过节点时TTL数值减一，数据为0时此包无效`
于是，使用自己路由器时，由设备发出的TTL在经过路由器时出现了减一的操作，于是被运营商检测出来，从而影响网络使用。
    >关于TTL的详细信息可以参考[HuiHuBuTong](https://github.com/TianxinZhao/HuiHuBuTong/tree/main?tab=readme-ov-file#2%E8%A7%A3%E9%94%81%E8%B7%AF%E7%94%B1%E5%99%A8%E5%AD%90%E7%BD%91%E9%99%90%E5%88%B6)项目里有关解释

    > 同时，我们怀疑运营商对用户的UA进行了检测。` UA，即User Agent，是互联网用于表示设备的标识 `。
如果不进行统一化，在运营商看来，同一个设备发出的信息就会出现异常。
非常形象的可以看到，前一秒还是macOS(Macintosh)，下一秒就变成了Windows，可能瞬间又变成Linux。
可能可以通过该方法确定用户仍使用多个设备。于是对网络进行限制。

- 流量特征检测，部分管理工具（如锐捷）甚至可能搭配了深度包检测的功能。

- 封锁了部分steam下载节点，可能切换一个节点即可解决。
  
- 同时注意到，根据VPN/网络代理原理，TTL的限制使得翻墙有了更高的难度。
    > Remember, BIG BROTHER is WATCHING you.

# §2 限制的一些解决思路
` 可能需要一台OpenWRT设备或Linux终端 `

### OpenWRT的前置工作。

1. 准备OpenWRT系统镜像文件。
   可在OpenWRT官网选择想要的系统版本，或者使用[LEDE](https://github.com/coolsnowwolf/lede)项目自行编译合适的镜像。

2. 将编译出的系统镜像刷至路由器。具体步骤根据硬件不同而不同，因此不展开解释。

3. 待系统准备完成后，在连接至路由器的电脑上使用SSH连接OpenWRT。
    >在初次连接时，可能报错无法连接，此时，应当使用` ssh-keygen -f ".\.ssh\known_hosts" -R "192.168.1.1" `，随后便可正常连接。

### OpenWRT上的一些操作
**！！此步骤极为重要，不可跳过**  

此时，若一切正常，你将已经在SSH中连接至OpenWRT，应该已经看到OpenWRT的欢迎界面，于是可以进一步操作

1. 首先，为保证连接正常，先进行一个源的换。  
` sed -i 's_https\?://downloads.openwrt.org_https://mirrors.tuna.tsinghua.edu.cn/openwrt_' /etc/opkg/distfeeds.conf `  
    本指令将自动把opkg配置变更为来自清华tuna协会的镜像，极大提升了成功率和速度。  

2. 随后，更新一下软件包，此指令后你将看到很多包含tsinghua.tuna.moe的网址，表示上一步成功  
` opkg update `

3. 安装一下马上要用到的妙妙工具。  
` opkg install privoxy luci-app-privoxy luci-i18n-privoxy-zh-cn iptables iptables-mod-ipopt nano `

4. 进行privoxy的配置，此步用来修改UA，保险作用为主，避免薛定谔的网络连接。  
   ` nano /etc/privoxy/user.action `  
    找一行新增  
   ` {+client-header-filter{user-agent-filter}}
   / `  
    随后  
   ` nano /etc/privoxy/user.filter `  
    找一行新增一句  
   ` CLIENT-HEADER-FILTER: user-agent-filter
   s@User-Agent:.*@User-Agent: YourPersonalChoice@ `  
    > 自选一个喜欢的UA即可（

5. 随后配置TTL相关部分  
根绝上文所述有关TTL部分，解决方案逻辑非常清晰简单，即通过OpenWRT等个人网关修改来自LAN的所有网络活动TTL部分为64或128.
步骤如下:

   - 终端中输入:  
    ` iptables -t mangle -l POSTROUTING -J TTL --ttl-set 64 `  
    至此，理论上已解决设备数量限制,可以测试一下是否正常，若正常没，进行下一步。
   
   - 因为iptables重启就寄，于是写入配置  
   ` nano /etc/rc.local `  
   新增配置语句：  
   ` iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64 `  
   于是重启也不会掉配置。至此，已经基本解决网路限制。



# §3
* **本着技术开源的精神特将探索过程及解决原理公布于此。任何人未经授权不可将本项目用于商业盈利用途。**  
* **如有兄弟有新的思路，非常期待与您的讨论，您可直接在本项目提issue讨论。**


***

Written by [@ELT](https://github.com/ELT17604), uploaded by CYIST in 20240908
