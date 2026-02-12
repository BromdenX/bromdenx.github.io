---
layout:     post
title:      BuildingYourBridge
subtitle:   BuildingYourBridge
date:       2026-02-12
author:     BromdenX
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Blig
---


> To adong: Building your bridge to the free web. — **BromdenX**

## 1.核心思想
- 采用最新的系统、软件和技术
- 尽可能的伪装自己的访问行为
- 有备份的策略，避免单一方案被封
- 从主机到软件的全流程安全配置

## 2.整体的思路
- 结合Google到的相关信息，当前最新的开源节点搭建中，相对社区比较活跃并且技术比较牛的就是两个，一个是xray的Reality，另一个hysteria2，所以锁定这两个作为代理协议；
- 伪装是当前“现代“FQ理念（相见[关于GFW的历史](关于GFW的历史.md)）的核心，首先加密我们要用业界最通用最牛的https标准加密方式（包括443端口）；其次单纯的加密容易被发现，只有伪装流量行为，并且防止被探测才是王道，所以我们的核心理念是每一个工具注意利用它的”回落“特性；
- 备份策略要结合自己的伪装技术，也就是在一个VPS搭建多个节点，每个节点是一个层次的伪装，结合我的经验，建议分三层：
	- 第一层：日常行为的伪装，在节点上搭建一个网盘的服务，日常使用节点时，流量就模仿这个网盘的访问行为，探测是也是这个网盘；而且这个网盘的服务要结合nginx的特性以及xray的特性，确保它的https是最安全最顶级的配置；
	- 第二层：SNI的伪装，主要是为了防范不同网络中可能基于SNI进行域名的限制访问，或者为了针对SNI白名单的限制；因为第一层的伪装，我们访问自己的网盘可能是自己的小成本域名（XX.top或XX.xyz），未来SNI白名单时，被封的概率很大；所以要启用SNI伪装，利用其他符合条件的SNI进行伪装，例如www.cisco.com，从SNI的角度来看，你的流量就是在访问思科的官网；
	- 第三层：IP的伪装，目的是为了防止IP地址被墙或者被限制的情况，这里只能使用CF的特性来进行访问了；
	- 主机做好防护
- VPS的主机作为一个对互联网开放的服务器设备，基础的设备安全防护要有的，否则挂在互联网网上的开放机器，每天的扫描不计其数；

## 3.操作步骤
 ==Checklist清单==
 - 必备条件的准备
	 - VPS主机--------------------------
	 - 自有的域名-----------------------
	 - CF账号---------------------------
- 基础环境的部署
	- 安装1panel------------------------
	- 1panel中开启ssl-------------------
	- 修改1panel的密码-----------------
	- 修改ssh的端口--------------------
	- 安装Fail2ban----------------------
	- 安装ufw---------------------------
	- 禁密码登陆，改证书登陆-----------
- 安装部署软件
	- 安装nginx--------------------------
	- 安装xray---------------------------
	- 安装hysteria-----------------------
- 梳理代理流量规划
	- 梳理流量规划----------------------
	- 配置文件调整----------------------
	- 效果的检验和验收------------------
### 3.1 必备条件的说明
**关于VPS主机的选择**
- 建议提前测试一下到你这里的延迟情况，这里关乎到自己的体验；
- 优先选择具有ipv6地址的主机，一般IPv6会给三个地址或者一个64位的地址，多一个地址就相当于多一个节点；
- VPS选择系统的时候优先选择ubuntu最新的系统，主观感觉例如ubuntu 24.04

**关于自有的域名申请**
- 怎么便宜怎么来，但是必须是到国外的网站注册，国内要备案；
- 如果便宜的可以直接续费9年，一劳永逸，必定这个资源越来越稀少；
- 拿到域名后直接托管到cf平台，操作方便，界面友好；（很多域名提供商的平台不敢恭维）；

提前配置域名解析(假设你申请的域名为x.top)，以下为示例
- 1panel.x.top，作用是给1panel访问使用，可以自己定义；
- pan.x.top，伪装网盘使用，后续访问就是你的网盘；
- blog.x.top，代理使用，后续访问会通过cf代理访问你的一个网页（也可以是网盘）；
### 3.2 基础环境的部署
#### 3.2.1 安装1panel服务器管理面板
这个是可选的，优先是开源安全，并低的资源占用率；安装面板的目的和作用有几个
- 随时可以查看服务器资源的使用情况；
- 方便查看ssh访问的日志情况；
- 申请证书和搭建网盘节点图形界面一键操作；

https://1panel.cn/
根据网页提醒，一键安装脚本

#### 3.2.2 在1panel中开启ssl的管理
> 安装了1panel过后，这个动作很重要，没有开启ssl之前，你所有的操作都是明文的，理论上可以被抓包获取账号密码等权限；
> 当然了，这个也是为了保险，到此刻你还在安装部署一台正常的服务器呢，应该没人会监听；

这里的步骤分四步：
- cf中找到自己域名的管理页面，页面右下角“获取您的API 令牌“，创建令牌，编辑区域DNS模版，特定区域选择自己的域名，继续显示摘要，获得令牌（记住这个令牌）；
- 1panel页面中，网站-证书，DNS账号，创建一个账户 邮箱随便填，重点的是“API Token”填写上面的令牌；
- 1panel页面中，网站-证书，申请证书，填写a.x.top域名；
- 1panel设置页面，开启ssl访问，可以正常通过域名访问，就证明成功了；

#### 3.2.3 修改1panel的密码
> 这里是建议操作，目的是假设开启ssl之前自己的密码已经失窃了，这里是在https的保护之下修改的密码，理论上是安全的；

#### 3.2.4 修改ssh的端口
> VPS默认的密码是22端口，从你开启机器的那一刻，你就会被发现已经被开始各种扫描了；安装了1panel之后可以通过日志管理页面查看；
> 修改为非常用端口的目的也是为了躲避一大部分的扫描，但是经验告诉我们，一点时间后还是会有各种扫描；

方法一：通过命令
编辑ssh配置文件： `sudo vi /etc/ssh/sshd_config`， 加入`Port 2222`保存`:wq`退出；
（注意如果不想同时关闭22端口的话 ，可以不注释掉，后面调整完防火墙策略等后再关闭；避免把自己拒之门外了）

方法二：通过图形界面
1panel中有修改的地方，直接图形操作即可；

#### 3.2.5 安装fail2ban
> 虽然我们修改了SSH的端口号，但是根据经验一段时间后仍然有大量的扫描；Fail2ban是一个开源的 Linux 入侵防御工具，能够自动识别并封禁有暴力破解嫌疑的 IP 地址，直接将它列入黑名单；

1panel页面中，工具箱，Fail2ban直接安装，安装后注意在配置页面修改对应的SSH端口号；关于禁用时间和周期等字段可以自行设置。

#### 3.2.6 安装ufw
> UFW是ubuntu系统上的防火墙设备的前端管理工具，方便撰写ACL策略的；

**1.安装ufw软件**
`apt install ufw`

**2.添加端口放行**
`ufw allow 443`

**3.使能生效**
`ufw enable`

#### 3.2.7 禁用密码登陆开启证书登陆
> 传统的密码登陆方式，肯跟存在被暴力破解的情况；通过密钥整理，由于采用公私钥架构，理论上无法被暴力破解，更安全；

**1.本地生成证书**
在自己的电脑上操作，一般都已经安装了`ssh-keygen`
```
ssh-keygen -t ed25519
```

**2.拷贝公钥证书到服务器**
根据前面的生成，会看到公钥和私钥的证书内容，通过命令将公钥证书直接传给服务器
```
ssh-copy-id -p 2222 root@1panel.x.top
```

**3.修改证书登陆和禁用密码登陆**
在服务器上进行操作，或者在1panel上有对应的操作页面（SSH管理）
```
vi /etc/ssh/sshd_config

禁用密码登陆
PasswordAuthentication no

开启公钥登陆
PubkeyAuthentication yes
```

**4.验证可以通过证书登陆了**
通过其他ssh工具或者自带的命令行都可以验证；
建议使用第三方的ssh工具，这样可以管理ssh证书

**5.拷贝备份自己的私钥**
> 根据上面ssh-keygen生成的回显提示，找到自己的私钥路径，进行拷贝或者备份，方便后面使用或者备份；因为默认路径，后续你生成其他密钥会被覆盖掉。

==至此服务器的管理和基本防护功能已经OK==
### 3.3 安装部署软件
#### 3.3.1 安装nginx软件
> nginx代理的作用主要是有三个：
> 1.将它顶到网络的最前沿，避免代理软件的特征暴露；443端口的服务，证书的提供以及其他的防护响应都有nginx的提供，最大限度的降低被探测风险；
> 2.后面整体流量的中转代理也都由nginx来提供，这是最标准和安全的，也进一步降低风险；
> 3.提供blog的网页服务，给探测者或者正常访问做正常的回应；

直接使用ubuntu的`apt install nginx`可以安装，但是不是最新的，一方面违背了我们采用最新软件的思想，另一方面也有一些高级特性例如stream和http3可能不支持。

安装最新版本的nginx版本的操作步骤如下，可以直接问大模型或者直接使用下面的方法：
**1. 安装必要工具：**
```
sudo apt update
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring -y
```

**2. 导入 GPG 密钥：**
```
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
| sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

**3. 添加 Mainline 软件源：**
```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
| sudo tee /etc/apt/sources.list.d/nginx.list
```

**4. 设置优先级（确保系统优先选择官方源）：**
```
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
| sudo tee /etc/apt/preferences.d/99nginx
```

**5.升级更新并安装软件**
```
sudo apt update
sudo apt install nginx -y
```

最后通过命令可以查询到安装的nginx版本`nginx -v`，最新版本应该为
```
# nginx -v
nginx version: nginx/1.29.5
```

#### 3.3.2 安装Xray代理工具
> 采用xray的代理工具的原因是它的设计最活跃，也是更新软件速度最快的；同时它的Reality协议也是目前比较安全的软件之一吧，选择他没错的；

可以到它的github平台找到一键安装脚本，安装最新的版本
```
https://github.com/XTLS/Xray-install
```

安装完成之后配置文件的路径如下：
```
/usr/local/etc/xray/config.json
```

后续只需要编辑这一个文件即可；
#### 3.3.3 安装Hysteria2代理工具
> 采用Hysteria2的代理工具原因，一方面它采用UDP暴力传输，可以拯救网路拉垮的VPS，另一方面它也号称是Reality的终结者，采用http3协议伪装，可以作为备用。

可以到它的官网或者github平台找到一键安装脚本，安装最新的版本
```
https://v2.hysteria.network/zh/docs/getting-started/Installation/
```

安装完成之后配置文件的路径如下：
```
cat /etc/hysteria/config.yaml
```

#### 3.3.4 安装网盘应用
> 1panel的应用商店中就有，直接安装就行，开源的openlist应该也比较安全；

1panel的管理页面，应用商店，云存储，openlist直接安装；
不需要任何的配置，安装完成后看到应用是监听的5244端口，密码在日志中可以查看到；

### 3.4 梳理代理流量规划
#### 3.4.1 统一出口
为了模仿传统的流量和尽可能的伪装，我们对外开放的只有一个443端口，这个也符合正常力量的特性；
防火墙中可以配置了，注意是开启TCP/UDP的443端口

#### 3.4.2 节点的设计
**节点1：伪装访问网盘的流量（TCP 443或者说是http2）**
`Pan(pan.a.top)  -->443-->xray的1111回落-->nginx的2220-->5244（网盘）`

**访问形式：**
- IP地址：VPS的IP地址
- SNI：pan.a.top
> 配置DNS的解析，正常访问域名就是这个组合；
> 这个正常的访问需要使用证书，按照申请1panel证书的路径同样给域名pan.a.top提前申请好证书，并设置推送路径，记住这个路径，后面会用到；

**代理协议Reality**

**流量走向说明：**
- 代理流量：流量到服务器的nginx的443端口，nginx的stream模块根据规则判断，会指向到Xray的1111端口，xray根据安全配置校验，校验通过判断是Reality的流量，那么直接处理，代理生效；
- 正常流量（探测流量）：流量到服务器的nginx的443端口，nginx的stream模块根据规则判断，会指向到Xray的1111端口，xray根据安全配置校验，校验不通过，会回落到nginx的2220端口，nginx的1110端口配置代理规则，指向自己的5244端口，5244网盘响应，回复网盘页面；

**节点2：伪装访问Cisco的流量（TCP 443或者说是http2）**
`Cisco（www.cisco.com) -->443-->xray的1112回落-->nginx的2221-->www.cisco.com`

**访问形式：**
- IP地址：VPS的IP地址
- SNI：www.cisco.com
> 代理工具中做配置，可以配置服务器是pan.a.top，SNI是www.cisco.com；这个逻辑就是带着SNI为www.cisco.com的数据包去访问pan.a.top；

**代理协议Reality**

**流量走向说明：**
- 代理流量：流量到服务器的nginx的443端口，nginx的stream模块根据规则判断，会指向到Xray的1112端口，xray根据安全配置校验，校验通过判断是Reality的流量，那么直接处理，代理生效；
- 探测流量：流量到服务器的nginx的443端口，nginx的stream模块根据规则判断，会指向到11112端口，xray根据安全配置校验，校验不通过，会回落到nginx的2221端口，nginx的2221端口配置代理规则，指向到www.cisco.com，思科官网响应，回复思科官网页面；

> 针对这种探测的流量，正常访问情况下应该很少出现，只能理解为针对思科的网站做了代理访问；这种代理访问效果可以通过curl工具校验得出结果，文后会提到。

**节点3：伪装访问网盘的流量（UDP 443或者说是http3）**
`Pan(pan.a.top)  -->UDP443-->Hysteria2的1114回落-->nginx的2220-->5244（网盘）`

**访问形式：**
- IP地址：VPS的IP地址
- SNI：pan.a.top
> 配置DNS的解析，正常访问域名就是这个组合；

**代理协议Hysteria2**

**流量走向说明：**
- 代理流量：流量到服务器的nginx的UDP 443端口，nginx的stream模块根据规则判断，会指向到Hysteria2的1114端口，Hysteria2根据安全配置校验，校验通过判断是Hysteria2的流量，那么直接处理，代理生效；
- 正常流量（探测流量）：流量到服务器的nginx的UDP 443端口，nginx的stream模块根据规则判断，会指向到Hysteria2的1114端口，Hysteria2根据安全配置校验，校验不通过，会Proxy到nginx的1110端口，nginx的1110端口配置代理规则，指向自己的5244端口，5244网盘响应，回复网盘页面；

**节点4：伪装访问网盘的流量（TCP 443或者说是http2）****
`Blog(blog.a.top)    -->CF443--->443-->xray的1113回落-->9443/Blog（blog网页）

**访问形式：**
- IP地址：cf的随机主机IP地址
- SNI：blog.a.top
> 配置DNS的解析，并开启了小黄云代理，那么你的域名就会被解析道CF的服务器IP地址上；
-这里有一个技巧，我们在访问的时候正常采用blog.a.top域名，cf会帮我解析到随机的IP地址，网上有一些IP优选或者域名优选的方法，说白了就是制定到特定的服务器，域名优选比较简单，直接找到 一些开启了cf优化的域名就可以，例如www.wto.org，使用这个节点的时候，服务器直接填写这个域名就可以了。
这个正常的访问需要使用证书，按照申请1panel证书的路径同样给域名blog.a.top提前申请好证书，并设置推送路径，记住这个路径，后面会用到；

**代理协议XHTTP**

**流量走向说明：**
- 代理流量：流量到CF服务器的443端口，由于开启了小黄云，根据域名blog.a.top进行匹配，CF服务器转发到你的VPS的nginx 443端口，nginx的stream模块根据规则判断，会指向到nginx的2220端口并匹配域名blog.a.top，nginx根据配置的路径规则判断，代理指向xray的1115端口，xray的1113端口根据安全配置校验，校验通过判断是XHTTP的流量，那么直接处理，代理生效；
- 正常流量（探测流量）：流量到CF服务器的443端口，由于开启了小黄云，根据域名blog.a.top进行匹配，CF服务器转发到你的VPS的nginx 443端口，nginx的stream模块根据规则判断，会指向到nginx的2220端口并匹配域名blog.a.top，nginx根据配置的路径规则判断，是网页路径，直接返回blog网页；

#### 3.4.3 网络端口的设计梳理
**ngix的监听端口**
- 443端口，对外，TCP和UDP都对外开放，主要是响应各种节点的请求；
- 2220端口，对内，响应其他协议的回落；可以根据域名或路径指向不同的其他端口或者网站；
- 2221端口，对内，响应来自xray的www.cisco.com的访问回落，直接指向思科网站；

**xray的监听端口**
- 1111端口，Reality协议，处理来自pan.a.top的SNI的节点数据；
- 1112端口，Reality协议，处理来自www.cisco.com的SNI的节点数据；
- 1113端口，XHTTP协议，处理来自blog.a.top的SNI的节点数据；

**hysteria2的监听端口**
- 1114端口，hysteria2协议，处理来自pan.a.top的SNI的节点数据；

#### 3.4.4 最终的流量走向图

```
正常访问的流量：
Pan_http2(pan.a.top) -->443-->xray的1111回落-->nginx的2220-->5244（网盘）
Cisco(www.cisco.com) -->443-->xray的1112回落-->nginx的2221-->www.cisco.com 
Pan_http3(pan.a.top)  -->UDP443-->Hysteria2的1114回落-->nginx的2220-->5244（网盘）
Blog_CF (blog.a.top)  -->CF443--->nginx的443-->xray的1113回落-->nginx的2220/Blog（blog网页）

节点访问流量：
Reality_Pan    -->443-->Xray的1111端口回应
Reality_Cisco  -->443-->Xray的1112端口回应 
Hysteria2_Pan  -->UDP443-->Hysteria2的1114端口回应 
XHTTP_CF_WTO   -->CF443--->443-->Xray的1113端口回应
```

### 3.5 配置关键点说明
#### 3.5.1 nginx的根据域名转发流量
//内部的为主要的配置说明

```
stream {
    log_format stream_log '$remote_addr [$time_local] '
                       '$protocol $status $bytes_sent $bytes_received '
                       '$session_time $upstream_addr $upstream_connect_time';

    access_log /var/log/nginx/stream_access.log stream_log;
    log_format simplified '$time_local $remote_addr $upstream_addr $ssl_server_name $status';
    access_log /var/log/nginx/stream_access_simplified.log simplified;
    error_log /var/log/nginx/stream_error.log warn;
//日志的格式配置，可以根据自己的要求，通过大模型自行生成

    map $ssl_preread_server_name $backend_name {
        pan.a.top reality_pan_backend;
        cisco.com reality_cisco_backend;
        www.cisco.com reality_cisco_backend;
        default web_backend;
    }
//规则设置，根据SNI转发到指定的规则
  
    upstream reality_pan_backend {
        server 127.0.0.1:1111;
    }

    upstream reality_cisco_backend {
        server 127.0.0.1:1112;
    }

    upstream web_backend {
        server 127.0.0.1:2220;
    }

    upstream hysteria_backend {
        server 127.0.0.1:1114;
    }    
//规则设置，对应到指定的内部端口
  

    server {
        listen 443 reuseport;
        listen [::]:443 reuseport;

        ssl_preread    on;
        proxy_pass     $backend_name;
        proxy_protocol on;
    }

    server {
        listen 443 udp reuseport;
        listen [::]:443 udp reuseport;
        proxy_pass    hysteria_backend;
        proxy_timeout 20s;
    } 
//替代传统的server内的监听端口，对外监听443端口规则
}

```




#### 3.5.2 Reality的关键配置
//内部的为主要的配置说明

**1. 配置节点1中监听的1111端口**
```
        {
            "listen": null,
            "port": 1111,  //监听的端口，配置为1111端口；
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "email": "PC",
                        "flow": "xtls-rprx-vision",
                        "id": "b1c24f20-216c-44ad-bd29-689abcc61338"  //UUID，建议通过xray uudi自行生成；
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": "127.0.0.1:2220",  //回落的目标设置，根据前面的规划，回落到本地的nginx的2220端口
                        "xver": 0
                    }
                ]
            },

            "streamSettings": {
                "network": "tcp",
                "realitySettings": {
                    "dest": "2220",  //这个是Reality的目标，获取对应的证书特征，同样还是到nginx的2220端口
                    "privateKey": "oKdYhdHO5X_ZCVFjH_f8jw64OdNh5rgexkgUbKx3oG0",   //通过xray x25519生成，privateKey填写到这里，public填写到自己的客户端；
                    "serverNames": [
                        "pan.a.top"   //跟节点访问的SNI相匹配；
                    ],
                    "shortIds": [
                        "112233445566"      //根据规范设置，字符为0-f，偶数。
                    ],
                    "xver": 1
                },
                "security": "reality",
                "tcpSettings": {
                    "acceptProxyProtocol": true,
                    "header": {
                        "type": "none"
                    }
                }
            },
            "tag": "inbound-1111",
            "sniffing": {
                "enabled": true,
                "destOverride": [
                    "http",
                    "tls"
                ]
            }
        },
```

**2. 配置节点2中监听的1112端口**
```
        {
            "listen": null,
            "port": 1112,  //监听的端口，配置为1111端口；
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "b1c24f20-216c-44ad-bd29-689abcc61338",  //UUID，建议通过xray uudi自行生成，可以跟上面一样；
                        "flow": "xtls-rprx-vision"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": "127.0.0.1:2221",  //回落的目标设置，根据前面的规划，回落到本地的nginx的2220端口
                        "xver": 0
                    }
                ]
            },
            
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "dest": "www.cisco.com:443",  //这个是Reality的目标，获取对应的证书特征，同样还是到nginx的2220端口
                    "serverNames": [
                        "www.cisco.com"   //跟节点访问的SNI相匹配；
                    ],
                    "privateKey": "oKdYhdHO5X_ZCVFjH_f8jw64OdNh5rgexkgUbKx3oG0",   //通过xray x25519生成，privateKey填写到这里，public填写到自己的客户端；
                    "shortIds": [
                        "11223344"      //根据规范设置，字符为0-f，偶数。
                    ]
                },
                "tcpSettings": {
                    "acceptProxyProtocol": true,
                    "header": {
                        "type": "none"
                    }
                }
            },
            "tag": "inbound-1112",
            "sniffing": {
                "enabled": true,
                "destOverride": [
                    "http",
                    "tls"
                ]
            }
        },
```



#### 3.5.3 Hysteria2的关键配置
//内部的为主要的配置说明

```
listen: 127.0.0.1:1114

tls:
cert: /opt/cert_pan/fullchain.pem
key: /opt/cert_pan/privkey.pem

auth:
type: password
password: fsaiohfag89354

masquerade:
type: proxy
proxy:
url: https://pan.a.top/
rewriteHost: true
```

#### 3.5.4 XHTTP的关键配置
//内部的为主要的配置说明

```
        {
            "listen": null,
            "port": 1113,  //监听的端口，配置为1111端口；
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "email": "tn1vc1nu",
                        "flow": "",
                        "id": "b1c24f20-216c-44ad-bd29-689abcc61338"  //UUID，建议通过xray uudi自行生成，可以跟上面一样；
                    }
                ],

                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": "127.0.0.1:2220",  //回落的目标设置，根据前面的规划，回落到本地的nginx的2220端口
                        "xver": 0
                    }
                ]
            },

            "streamSettings": {
                "network": "xhttp",
                "security": "none",
                "xhttpSettings": {
                    "headers": {},
                    "host": "",
                    "mode": "auto",
                    "noSSEHeader": false,
                    "path": "/faoga9geanc",      //设置路径，跟客户端和nginx的配置一致就可以了。
                    "scMaxBufferedPosts": 30,
                    "scMaxEachPostBytes": "1000000",
                    "scStreamUpServerSecs": "20-80",
                    "xPaddingBytes": "100-1000"
                }
            },

            "tag": "inbound-1113",
            "sniffing": {}

        },
```

#### 3.5.5 nginx网页和代理的关键配置
//内部的为主要的配置说明

**1.nginx的服务响应，转发到网盘**
```
    server {
        listen 127.0.0.1:1111 quic reuseport;
        listen 127.0.0.1:1 ssl proxy_protocol reuseport;

        server_name pan.a.top;

        http2 on;

        set_real_ip_from 0.0.0.0/0;
        real_ip_header   proxy_protocol;


        ssl_certificate     /opt/cert_pan/fullchain.pem;
        ssl_certificate_key /opt/cert_pan/privkey.pem;
        //上面提到的申请到的pan.a.top证书的路径，保持一致。


        ssl_protocols             TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers               ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305;
        ssl_ecdh_curve            secp521r1:secp384r1:secp256r1:x25519;

        location / {
            add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
            add_header Alt-Svc 'h3=":443"; ma=86400';

            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   Host $host;
            proxy_pass         http://127.0.0.1:5244;        //重定向到网盘。
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection "upgrade";
        }
    } 

```

**2.nginx的CF服务响应，转发到blog**
```

    server {
        listen 127.0.0.1:111 quic ;
        listen 127.0.0.1:1111 ssl proxy_protocol ;
        
        server_name blog.a.top;
        
        http2 on;
  
        set_real_ip_from 0.0.0.0/0;
        real_ip_header   proxy_protocol;

        ssl_certificate     /opt/cert_blog/fullchain.pem;
        ssl_certificate_key /opt/cert_blog/privkey.pem;
        //上面提到的申请到的blog.a.top证书的路径，保持一致。

        ssl_protocols             TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers               ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305;
        ssl_ecdh_curve            secp521r1:secp384r1:secp256r1:x25519;

        location / {
            add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
            add_header Alt-Svc 'h3=":443"; ma=86400';
            root /var/www/html;
            index index.html;
        }
        //设置本地网址的路径，对应的index.html即为自己的博客内容；推荐采用纯静态，效率高还安全。
    }
```

**3.nginx的服务响应，转发到Cisco**
```
    server {
        listen 127.0.0.1:1112 ;
        server_name www.cisco.com;
        location / {
            proxy_pass https://www.cisco.com;
            proxy_ssl_server_name on;
            proxy_ssl_name www.cisco.com;
            proxy_set_header Host www.cisco.com;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    } 
```


### 3.6 部分效果校验、技巧和FAQ

#### 关于Reality目标网站的选择
方法：打开你想要配置成为目标的网站，使用浏览器的web开发工具调试，在控制台打开协议，刷新页面，看到H2或者HTTP/2就是支持h2的网站，然后点击证书看是不是tls 1.3，还有就是ip地址是否和你vps地址相近，越近越好。

具体操作：通过chrome浏览器去筛选（更多工具-开发者工具）
1. 「安全」选项卡查看以下内容，显示 `已使用 TLS 1.3 、X25599` 为符合要求的网站，其他不符合。
2. 「网络」选项卡需要先刷新页面，在表头空白处点击右键，勾选协议，如协议列中出现 `H2` 为符合要求的网站，没有出现则不符合。

之前检验过可用的域名
```
apache.org
www.phoronix.com
www.wireshark.org
www.yoctoproject.org
```

#### 关于一些Linux日志的优化
配置完成后，系统会产生大量的日志，默认的nginx会使用logrotate 每天压缩日志，但是信息依然不好找；xray等工具的日志更是会将一个文件保存的无限大；

**日志轮转（自动命名为 .1, .2, .3 等）**  
Nginx 本身不直接支持自动日志轮转或生成 .1, .2, .3 等文件。这种行为通常由外部工具 **logrotate** 实现，logrotate 是 Linux 系统中常用的日志管理工具。你的描述（日志文件自动命名为 .1, .2, .3）表明系统中可能已配置了 logrotate。

**Logrotate 的工作原理**
- **logrotate** 会定期检查日志文件大小或时间（如每天、每周或达到特定大小）。
- 当触发轮转条件时，logrotate 会：
    1. 重命名当前日志文件（如 access.log 变为 access.log.1）。
    2. 如果 access.log.1 已存在，则依次重命名（access.log.1 变为 access.log.2，依此类推）。
    3. 创建一个新的空 access.log 文件。
    4. 通知 Nginx 重新打开日志文件（通常通过发送 HUP 信号）。
- 轮转后的文件通常以 .1, .2, .3 等命名，具体取决于 logrotate 配置中的 rotate 参数。

```
    检查 Logrotate 配置
    你的 Nginx 日志文件（如 /var/log/nginx/*.log）很可能由系统默认的 logrotate 配置管理。检查配置文件：  
    cat /etc/logrotate.d/nginx
```

典型 logrotate 配置示例（/etc/logrotate.d/nginx）：
```
/var/log/nginx/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 nginx nginx
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

**配置说明**：
- daily：每天轮转一次。
- rotate 7：保留 7 个历史日志文件（生成 access.log.1 到 access.log.7）。
- compress：对轮转后的日志进行压缩（如 access.log.2.gz）。
- create 0640 nginx nginx：轮转后创建新日志文件，权限为 0640，属主为 nginx 用户。
- postrotate：轮转后发送 USR1 信号给 Nginx，通知其重新打开日志文件。

**轮转后文件命名**：
- 第一次轮转：access.log → access.log.1，新 access.log 创建。
- 第二次轮转：access.log.1 → access.log.2，access.log → access.log.1，新 access.log 创建。
- 如果 rotate 设置为 7，最多保留 access.log.1 到 access.log.7，旧文件会被删除。

最终调整日志状态，按照日期进行  
Xray的为例
```
/var/log/xray/*.log {
daily
rotate 90
compress
dateext
dateformat -%Y%m%d-%H%M%S
missingok
notifempty
create 0640 nobody nogroup
sharedscripts
postrotate
	systemctrl restart xray
endscript
}
```

#### 关于一些工具的使用
**Curl探测工具**
可以使用一个工具做web探测
```
curl -v https://www.example.com
```

可以在web探测的时候制定sni信息
```
curl -v --resolve www.example.com:443:1.2.3.4 https://www.example.com

```
这里的 `--resolve` 告诉 curl：当访问 `www.example.com` 时，请解析到 `1.2.3.4`，同时 TLS 握手阶段的 SNI 字段会填入 `www.example.com`。



#### 关于部分FAQ的问题
**nginx提示用户不存在**

```
# 创建一个名为 nginx 的系统用户，且不允许登录 
useradd -s /sbin/nologin -M nginx
```

**CF访问不成功**
通过443端口的CF无法访问blog，可能是SSL证书严格的问题，修改为“完全（严格）”解决了问题；

