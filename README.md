# 🌠 Comet Gateway

实验性 Naiveproxy 透明网关. [WIP]

> 配置未经严格测试，欢迎[反馈](https://github.com/kwaa/comet/discussions)或[提交 bug](https://github.com/kwaa/comet/issues)。

## 用法

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

接下来修改 Naiveproxy 和 Sing Box 的配置（替换 `comet.local, user, pass` 为你的域名、账号和密码）：

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
      "geosite": "cn",
      "geoip": [
        "private",
        "cn"
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

TODO

### 可选 - UDP over TCP

为了使用 [UDP over TCP](https://sing-box.sagernet.org/configuration/shared/udp-over-tcp/)，你需要使用支持的服务端（例如 Sing Box）。

只需在客户端配置文件加上 `"udp_over_tcp": true`：

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
