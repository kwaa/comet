# 🌠 Comet Gateway

实验性 Naiveproxy 透明网关. [WIP]

> 配置未经严格测试，欢迎[反馈](https://github.com/kwaa/comet/discussions)或[提交 bug](https://github.com/kwaa/comet/issues)。

## 介绍

### 这是什么

这是一套基于 Docker Compose 的透明网关方案，简单修改配置便可启动。

你需要一个（或多个）可用的 Naiveproxy 服务端。

### 为什么是

#### Naiveproxy

我从 2019 年开始用 Naiveproxy，2022 年 10 月的大规模封锁（[net4people/bbs#129](https://github.com/net4people/bbs/issues/129)）证明了我的选择是对的。

#### sing-box

不熟悉 Clash / Clash Premium / Clash.Meta，就是这样。

#### HAProxy

因为它很流行。

## 用法

> 推荐 512M 以上 RAM，RK3328 以上 CPU

以 Debian 系发行版为例，以下所有命令都以 root 账号执行。

```bash
# 安装 docker, docker-compose, git, curl
apt install docker.io docker-compose git curl
# 跳转到你希望的目录，此处将复制到 /opt/comet
cd /opt
# 复制存储库
git clone https://github.com/kwaa/comet.git
# 移动到 /opt/comet
cd comet
```

接下来修改 Naiveproxy 和 sing-box 的配置（替换 `comet.local, user, pass` 为你的域名、账号和密码）：

###### ./naive/config.json

```diff
{
  "listen": "socks://0.0.0.0:1080",
- "proxy": "https://user:pass@comet.local",
+ "proxy": "https://admin:1234@example.com",
  "log": ""
}
```

###### ./sing-box/config.json

```diff
"route": {
  ...
  "rules": [
    ...
    {
      "domain_keyword": [
-       "comet.local",
+       "example.com"
      ],
      "outbound": "direct"
    },
  ],
  ...
},
```

将 `./sysctl.d/39-comet.conf` 复制到 `/etc/sysctl.d/`，开启 IP 转发：

```bash
cp ./sysctl.d/39-comet.conf /etc/sysctl.d/
sysctl --system
```

设置 iptables 规则：

```bash
iptables -t filter -A FORWARD -j ACCEPT
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
```

现在可以启动了。

```bash
docker compose up -d
```

在你希望代理的设备（或主路由）上将默认网关和 DNS 改为 Comet 所在机器的局域网 IP。就是这样！

### 可选 - 负载均衡

如果有多个可用服务器，可以启用基于 HAProxy 的负载均衡以提高整体可用性。

取消 HAProxy 段落的注释：

###### ./docker-compose.yml

```diff
- # haproxy:
- #   container_name: haproxy
- #   image: haproxy:alpine
- #   restart: always
- #   volumes:
- #     - ./haproxy:/usr/local/etc/haproxy
- #   ports:
- #     - 1080:1080/tcp
- #     - 1080:1080/udp
+ haproxy:
+   container_name: haproxy
+   image: haproxy:alpine
+   restart: always
+   volumes:
+     - ./haproxy:/usr/local/etc/haproxy
+   ports:
+     - 1080:1080/tcp
+     - 1080:1080/udp
```

增加 naive 客户端：从 `ports` 改为 `expose`，并按照喜好重命名；

还需要在 `./naive/` 文件夹内创建新的配置文件。

###### ./docker-compose.yml

```diff
- naive:
+ naive-a:
-   container_name: naive
+   container_name: naive-a
    image: kwaabot/naive
    restart: always
    volumes:
-     - ./naive/config.json:/etc/naive/config.json
+     - ./naive/config-a.json:/etc/naive/config.json
-   # expose:
-   #   - '1080'
+   expose:
+     - '1080'
-   ports:
-     - 1080:1080/tcp
-     - 1080:1080/udp

+ naive-b:
+   container_name: naive-b
+   image: kwaabot/naive
+   restart: always
+   volumes:
+     - ./naive/config-b.json:/etc/naive/config.json
+   expose:
+     - '1080'
```

接下来在 `haproxy.cfg` 中导入服务器：

###### ./haproxy/haproxy.cfg

```diff
backend naive-out
  balance roundrobin
- server naive naive:1080
+ server naive-a naive-a:1080
+ server naive-b naive-b:1080
```

### 可选 - UDP over TCP

为了使用 [UDP over TCP](https://sing-box.sagernet.org/configuration/shared/udp-over-tcp/)，你需要使用支持此特性的服务端（例如 sing-box）。

只需在客户端 `outbounds.proxy` 配置文件加上 `"udp_over_tcp": true`：

###### ./sing-box/config.json

```diff
{
  "type": "socks",
  "tag": "proxy",
  "server": "127.0.0.1",
- "server_port": 1080
+ "server_port": 1080,
+ "udp_over_tcp": true
},
```

### 可选 - PaoPaoDNS

由于绕过 TUN 很麻烦，此存储库不再内置 [PaoPaoDNS](https://github.com/kkkgo/PaoPaoDNS)。

如果你有能力（在同一个 Docker Compose 文件中）搞定这些，欢迎 PR！

以下是搭建完成后 sing-box 配置对应的修改：

###### ./sing-box/config.json

```diff
{
  "dns": {
    "servers": [
-     {
-       "tag": "cloudflare",
-       "address": "https://1.1.1.1/dns-query"
-     },
      {
        "tag": "local",
-       "address": "223.5.5.5",
+       "address": "udp://{{PaoPaoDNS 服务的 IP}}",
        "detour": "direct"
      },
    ],
    ...
  }
  ...
}
```

### 可选 - 丢弃 GFW 伪造的 DNS 抢答包

来自 [如何构建一个防污染DNS - 影子屋](https://blog.bgme.me/posts/how-to-create-an-anti-pollution-dns/)。

```bash
iptables -t raw -A PREROUTING -m bpf --bytecode '38,48 0 0 0,84 0 0 240,21 34 0 96,48 0 0 0,84 0 0 240,21 0 31 64,48 0 0 9,21 0 29 17,40 0 0 6,69 27 0 8191,177 0 0 0,72 0 0 0,21 0 24 53,40 0 0 2,37 22 0 128,72 0 0 12,21 0 20 1,72 0 0 14,21 0 18 1,72 0 0 16,21 0 16 0,72 0 0 18,21 0 14 1,72 0 0 4,20 0 0 8,12 0 0 0,7 0 0 0,64 0 0 0,21 0 8 268435456,177 0 0 0,72 0 0 4,20 0 0 4,12 0 0 0,7 0 0 0,64 0 0 0,21 0 1 0,6 0 0 65535,6 0 0 0' -j DROP
iptables -t raw -A PREROUTING -m bpf --bytecode '27,48 0 0 0,84 0 0 240,21 23 0 96,48 0 0 0,84 0 0 240,21 0 20 64,48 0 0 9,21 0 18 17,40 0 0 6,69 16 0 8191,177 0 0 0,72 0 0 0,21 0 13 53,40 0 0 4,21 0 11 0,40 0 0 6,21 0 9 0,48 0 0 8,37 7 0 40,72 0 0 12,21 0 5 1,72 0 0 14,21 0 3 1,72 0 0 16,21 0 1 0,6 0 0 65535,6 0 0 0' -j DROP
iptables -t raw -A PREROUTING -p udp -m bpf --bytecode '39,40 0 0 20,21 0 36 53,32 0 0 36,21 0 34 0,32 0 0 32,21 3 0 65537,21 0 31 65536,40 0 0 30,21 15 29 33152,40 0 0 30,84 0 0 65487,21 17 0 34176,40 0 0 24,7 0 0 0,64 0 0 4,21 5 0 3222011905,21 0 21 536936448,64 0 0 8,21 0 19 0,64 0 0 12,21 3 17 0,64 0 0 10,37 15 0 255,53 0 14 64,32 0 0 4,21 11 0 0,21 11 0 16384,84 0 0 65535,21 8 9 16384,40 0 0 6,21 0 7 0,40 0 0 24,7 0 0 0,64 0 0 6,21 0 3 65537,64 0 0 10,21 0 1 60,6 0 0 1,6 0 0 0' -j DROP
ip6tables -t raw -A PREROUTING -m bpf --bytecode '29,48 0 0 0,84 0 0 240,21 0 25 96,48 0 0 6,21 0 23 17,40 0 0 40,21 0 21 53,40 0 0 4,37 19 0 128,40 0 0 52,21 0 17 1,40 0 0 54,21 0 15 1,40 0 0 56,21 0 13 0,40 0 0 58,21 0 11 1,40 0 0 4,20 0 0 8,7 0 0 1,64 0 0 40,21 0 6 268435456,40 0 0 4,20 0 0 4,7 0 0 6,64 0 0 40,21 0 1 0,6 0 0 65535,6 0 0 0' -j DROP
ip6tables -t raw -A PREROUTING -m bpf --bytecode '19,48 0 0 0,84 0 0 240,21 0 15 96,48 0 0 6,21 0 13 17,40 0 0 40,21 0 11 53,32 0 0 0,21 0 9 1610612736,40 0 0 4,37 7 0 128,40 0 0 52,21 0 5 1,40 0 0 54,21 0 3 1,40 0 0 56,21 0 1 0,6 0 0 65535,6 0 0 0' -j DROP
ip6tables -t raw -A PREROUTING -p udp -m bpf --bytecode '23,40 0 0 40,21 0 20 53,32 0 0 52,21 0 18 65537,32 0 0 56,21 0 16 0,40 0 0 0,84 0 0 65520,21 0 13 24576,40 0 0 44,7 0 0 0,64 0 0 24,21 5 0 3222011905,21 0 8 536936448,64 0 0 28,21 0 6 0,64 0 0 32,21 3 4 0,64 0 0 30,37 2 0 255,53 0 1 64,6 0 0 1,6 0 0 0' -j DROP
```

## 许可证

本项目采用 WTFPL（你他妈的想干嘛就干嘛公共许可证）。有关更多详情，请参阅 [COPYING](COPYING) 文件。

### 特别感谢

- 从 [chika0801/sing-box-examples](https://github.com/chika0801/sing-box-examples) 参考了一些配置项
