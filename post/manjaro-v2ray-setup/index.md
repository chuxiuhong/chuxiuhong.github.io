
题外话：最近一直想写一个实验，实验平台又必须用到Linux或者MacOS来搞，然而本机Win10本身带着Docker，虚拟机有冲突，WSL又嫌太麻烦了，双系统折腾好几次，始终不是很满意。突发奇想，把之前安了Manjaro的老笔记本掏出来，然后接一条显示线在主显示器上，再在某宝上买一个USB切换器，实现键鼠一键切换两台电脑，随时随地实现Win10和Manjaro的切换，目前看比较满意。然后就是任何新机的第一步：解决科学上网问题。

首先是我的机器版本：

![image](/static/manjaro_screenfetch_20210505.png)

切换pacman源之类的事网上有很多教程，就不赘述了，这里假定你pacman,yay都是做好准备的。

下载v2ray

```bash
sudo pacman -S v2ray
```

安装之后在`/etc/v2ray`文件夹下有一个`config.json`配置文件，用来配置你的v2ray客户端。下面是一个文件配置示例

```json
{
  "policy": null,
  "log": {
    "access": "/etc/v2ray/access.log",
    "error": "/etc/v2ray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "proxy",
      "port": 10809,
      "listen": "0.0.0.0",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "settings": {
        "auth": "noauth",
        "udp": true,
        "ip": null,
        "address": null,
        "clients": null
      },
      "streamSettings": null
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "123.123.123.123", //这里写你自己服务器IP
            "port": 443,//这里写你自己服务器端口号
            "users": [
              {
                "id": "166e8f09-eadb-5ad4-ab18-beb5f9e7ecb8",//这里写生成的用户ID
                "alterId": 74,//额外ID
                "email": "t@t.tt",//无所谓
                "security": "auto"//按照服务器配置写
              }
            ]
          }
        ],
        "servers": null,
        "response": null
      },
      "streamSettings": {
        "network": "ws", //传输协议类型，和服务器一致
        "security": "tls",//底层传输安全，是否启用tls和服务器配置一致
        "tlsSettings": {
          "allowInsecure": false,
          "serverName": "www.fuckgfw.top"//你服务器的域名，顺便说，fuckgfw这种域名国内还注册不了
        },
        "tcpSettings": null,
        "kcpSettings": null,
        "wsSettings": {
          "connectionReuse": true,
          "path": "/fuckfangbinxing",//存证书密钥的路径，和服务器设置一致
          "headers": {
            "Host": "www.fuckgfw.top"
          }
        },
        "httpSettings": null,
        "quicSettings": null
      },
      "mux": {
        "enabled": true,
        "concurrency": 8
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {
        "vnext": null,
        "servers": null,
        "response": null
      },
      "streamSettings": null,
      "mux": null
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {
        "vnext": null,
        "servers": null,
        "response": {
          "type": "http"
        }
      },
      "streamSettings": null,
      "mux": null
    }
  ],
  "stats": null,
  "api": null,
  "dns": null,
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "port": null,
        "inboundTag": [
          "api"
        ],
        "outboundTag": "api",
        "ip": null,
        "domain": null
      }
    ]
  }
}
```

这里还要注意，写日志的地方如果像我这样把日志文件放到`/etc/v2ray`文件夹下，那v2ray可能会遇到权限问题，最简单的解决方法加权限:

```bash
sudo chmod +777 -R /etc/v2ray
```

写好你的文件之后，用`v2ray -test`可以测试正确性。

配置好之后，我发现Manjaro貌似脑子不太行的样子，每次在系统设置里设置好代理，保存，然后退出来就变回去了，于是选择在浏览器里设代理。这里注意浏览器里仅设置socks代理就足够了，如果每个都设的话，浏览器连接任何网页都会无法建立安全连接，查日志会报

```
tcp:127.0.0.1:55250 rejected proxy/socks unkown Socks version:67
```

这一类的报错，这里就是有HTTP或者HTTPS的代理走了v2ray提供的socks代理端口，结果v2ray傻了，表示你没交代我还要干HTTP代理的活。

设置成下面这样

![image](/static/firefox_proxy_20210505.png)

这样就可以成功科学上网了

![image](/static/firefox_google_20210505.png)