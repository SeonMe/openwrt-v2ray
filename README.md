前提：此示范仅在原版 V2Ray 上做测试，所有配置操作均基于当前最新 x86 架构的 OpenWRT 19.07.1 版本进行

## 1、V2Ray 服务端

楼主的服务端采用的是 VMess + WebSocket + Nginx + TLS （没有CDN）配置，阁下可根据自己的实际情况进行修改配置客户端配置文件，如何安装这里不进行复述，自行解决。：）

## 2、OpenWRT

首先肯定是解决 DNS 污染问题啦，无庸置疑的。具体根据楼主先前写的 Trojan 教程既可。(点击这里)[https://github.com/SeonMe/openwrt-trojan]

备注：

从第 2 章节的 2.1 至 2.3，2.4之后的先不用去理会。

## 2.1、Netflix 部分

Trojan 文章没有提到 Netflix ，所以这里进行一个补充。

原理就是由 DNS 解析获得 Netflix 的 IP，并添加到 ipset 集，由 iptables 进行转发。很简单，只需要在 `/etc/dnsmasq.d/` 文件夹里放入一个文件，后缀 `netflix.conf`

```
server=/netflix.com/127.0.0.1#5353
ipset=/netflix.com/netflix_rules
server=/netflix.net/127.0.0.1#5353
ipset=/netflix.net/netflix_rules
server=/nflxvideo.net/127.0.0.1#5353
ipset=/nflxvideo.net/netflix_rules
server=/nflximg.net/127.0.0.1#5353
ipset=/nflximg.net/netflix_rules
server=/nflxext.com/127.0.0.1#5353
ipset=/nflxext.com/netflix_rules
server=/nflxso.net/127.0.0.1#5353
ipset=/nflxso.net/netflix_rules
```

说明：

127.0.0.1#5353 是 dnscrypt-proxy 的，也就是 Netflix 交由它去解析并添加到 `netflix_rules` 这个 ipset 集。

## 3、V2Ray 客户端

然后这里到了客户端的环节了，开头前提说过，这是基于 x86 架构来进行配置的，所以只需要很简单几个环节就可以了。

首先是下载 V2Ray 本体，请移步 GitHub V2Ray仓库下载当前最新版本的 `v2ray-linux-64.zip` 解压并上传到 OpenWRT，然后这里需要说明的一点就是，仅仅需要上传 v2ray 和 v2ctl 两个文件就可以了。

但是先要创建用于放置 V2Ray 配置文件的文件夹，如果阁下直接放置在 `/etc/` 文件夹下亦可，但需要修改启动脚本文件位置。

3.1、V2Ray 启动脚本

直接简单粗暴：

```
#!/bin/sh /etc/rc.common

START=90

USE_PROCD=1

start_service() {
        procd_open_instance
        procd_set_param command /usr/sbin/v2ray -config /etc/v2ray/config.pb -format pb
        procd_set_param user root
        procd_set_param respawn 300 0 5
        procd_set_param stdout 1
        procd_set_param stderr 1
        procd_set_param pidfile /var/run/v2ray.pid
        procd_close_instance
}
```

配置文件放在 `/etc/init.d/` 文件夹里既可，名称随阁下，别忘了赋予文件可执行权限 `chmod +x`。但是先别启动，100% 阁下这个时候是无法启动的，后面还有没讲。

说明：

V2Ray 本体位置在：/usr/sbin/v2ray
配置文件位置在：/etc/v2ray/ 文件夹

3.2、V2Ray 配置文件

阁下使用该示例文件时请先根据自己的实际情况进行修改，并删除掉所有中文注释。

示例配置文件：

```
{
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field", // 本地
        "inboundTag": ["socks_proxy","http_proxy"],
        "balancerTag": "local_proxy"
      },
      {
        "type": "field", // 普通节点的路由配置
        "inboundTag": ["dokodemo_proxy_1"],
        "balancerTag": "transparent_proxy_1"
      },
      {
        "type": "field", // Netflix 等节点的路由配置
        "inboundTag": ["dokodemo_proxy_2"],
        "balancerTag": "transparent_proxy_2"
      }
    ],
    "balancers": [ // 多少条节点添加多少个负载
      {
        "tag": "local_proxy", // 本地 HTTP、SOCKS 服务
        "selector": ["ws_proxy_1"]
      },
      {
        "tag": "transparent_proxy_1", // 普通节点
        "selector": ["ws_proxy_1"]
      },
      {
        "tag": "transparent_proxy_2", // Netflix 等节点
        "selector": ["ws_proxy_2"]
      }
    ]
  },
  "inbounds": [
    {
      "port": "10001", // 普通节点
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "timeout": 30,
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "tag": "dokodemo_proxy_1"
    },
    {
      "port": "10002", // Netflix 等节点
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "timeout": 30,
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "tag": "dokodemo_proxy_2"
    },
    {
      "port": "1080",
      "listen": "10.0.0.1", // 修改为自己的网关
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth"
      },
      "tag": "socks_proxy"
    },
    {
      "port": "1081",
      "listen": "10.0.0.1", // 修改为自己的网关
      "protocol": "http",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {},
      "tag": "http_proxy"
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "URL",
            "port": 443,
            "users": [
              {
                "id": "UUID",
                "level": 1,
                "alterId": 100,
                "security": "chacha20-poly1305"
              }
            ]
          }
        ]
      },
      "tag": "ws_proxy_1", // 普通节点
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {},
        "wsSettings": {
          "path": "/"
        }
      }
    },
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "URL",
            "port": 443,
            "users": [
              {
                "id": "UUID",
                "level": 1,
                "alterId": 100,
                "security": "chacha20-poly1305"
              }
            ]
          }
        ]
      },
      "tag": "ws_proxy_2", // Netflix 等节点
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {},
        "wsSettings": {
          "path": "/"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ]
}

```

说明：

配置文件中没有 LOG、DNS 等功能，因为 V2Ray 的 DNS 默认是本地，所以不需要，我们需要使用其他服务来做无污染 DNS 解析方案，以及完整的路由配置。你需要做的就是根据几条节点来添加对应的 outbounds ，然后再开头的 rules 和 balancers 添加对应的条目就可以了。示例文件是两个节点，一个普通，一个 Netflix 节点，依葫芦画瓢既可。

### 3.2.1、还是配置文件

前面已经编辑好了配置文件，我们还需要进行一个步骤，那就是让 V2Ray 本地不依赖 V2Ctl 运行。至于为什么 V2Ray 会依赖 V2Ctl 可自行百科。

还记得刚刚上传的 `v2ray` 和 `v2ctl` 两个文件吗？如果阁下两个文件都放置在了 `/usr/sbin/` 文件夹，那么现在可以直接在 OpenWRT 终端使用命令行进行操作了，就很简单一个，把常规的 json 文件转换为 Protobuf 文件，为什么要转换刚刚已经提到过了。

命令行：

```
v2ctl config < config.json > config.pb
```

说明：

< config.json > 里面的内容就是你配置文件的绝对地址，config.pb 就是输出的文件，最后别忘记把 config.pb 放入到前面启动脚本里写好的配置文件夹目录，以及正确的名字。转换完格式以后 v2ctl 是用不到了，阁下可自行删除掉。

## 3.3、iptables 部分

又来到了阁下喜欢的 iptables 部分，其实阁下如果了解的话，在前面 Trojan 的文章里 iptables 规则同样也适用 V2Ray，因为本质上没什么区别，都是透明代理。

```
iptables -t nat -N V2Ray
iptables -t nat -A V2Ray -d 您的节点 IP -j RETURN
iptables -t nat -A V2Ray -d 0.0.0.0/8 -j RETURN
iptables -t nat -A V2Ray -d 10.0.0.0/8 -j RETURN
iptables -t nat -A V2Ray -d 127.0.0.0/8 -j RETURN
iptables -t nat -A V2Ray -d 169.254.0.0/16 -j RETURN
iptables -t nat -A V2Ray -d 172.16.0.0/12 -j RETURN
iptables -t nat -A V2Ray -d 192.168.0.0/16 -j RETURN
iptables -t nat -A V2Ray -d 224.0.0.0/4 -j RETURN
iptables -t nat -A V2Ray -d 240.0.0.0/4 -j RETURN
# Delete China Domain List
iptables -t nat -D V2Ray -m set --match-set chinalist dst -j RETURN
# Delete China IP
iptables -t nat -D V2Ray -m set --match-set chnroute dst -j RETURN
# Delete Netflix IP
iptables -t nat -D V2Ray -m set --match-set netflix_rules dst -j RETURN
ipset destroy
ipset create chinalist hash:net
ipset create chnroute hash:net
ipset create netflix_rules hash:net
for i in `cat /opt/chnroute/chnroute.txt`;
do
 ipset add chnroute $i
done
# Add China Domain List
iptables -t nat -A V2Ray -m set --match-set chinalist dst -j RETURN
# Add China IP
iptables -t nat -A V2Ray -m set --match-set chnroute dst -j RETURN
# Add Netflix IP
iptables -t nat -A V2Ray -p tcp -m set --match-set netflix_rules dst -j REDIRECT --to-ports 10002 # Netflix 代理
iptables -t nat -A V2Ray -p tcp -j REDIRECT --to-ports 10001 # 普通代理
iptables -t nat -A PREROUTING -p tcp -j V2Ray
```

说明：

Trojan 文章里有解释过，这里仅仅需要添加 Netflix 的 ipset 集既可，匹配到之后即刻转发到 10002 端口，所以要放在 10001 前面，iptables 规则顺序是由上而下的。

## 屁话部分

进阶部分 Trojan 那里讲的相对清楚了，如果阁下需要可根据那篇文章进行实践既可。
