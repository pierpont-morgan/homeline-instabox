# 家宽线部署：在 InstaBox 上跑 VPN Gate + Xray

一句话：家宽线不是买节点，是在 InstaBox 机器上先起一个 VPN Gate
网关容器，再让 Xray 走这个网关的 SOCKS 出去，出口就是志愿者
家里的宽带 IP，不是机房 IP。

## 为什么必须这样搭

- InstaBox 没有公网 IP、没有 UDP 入站，WireGuard 进不来，
  Reality 入站也活不下来，只能走 VLESS + WebSocket + port publish。
- 普通 Xray 直连出去是机器自己的机房 IP，干净度不够。
  前面加 VPN Gate 网关，出口才变成家宽 ISP。

## 需要什么

- 一台 InstaBox 机器，Debian，Docker、systemd、/dev/net/tun 能用。
- 记住 InstaBox 的规矩：免费版已实测至少可建 2 台、空闲 30 分钟暂停、连续 8 小时
  强制停、每台最多 16 个 port publish。家宽线不是 24 小时主线，
  是干净出口的备用线。
- 绝不 publish 5901（VNC），SOCKS 只绑 127.0.0.1，绝不公网暴露。

## 部署步骤（每国一条线）

1. 网关镜像，一台机器只建一次

```bash
docker build -t homeline-gw:v2 ./gateway/
```

网关里是：OpenVPN 客户端 + microsocks SOCKS5（只绑回环）+
OUTPUT DROP kill switch + 每分钟自检。

2. 找一个能用的节点，一个国家一次扫，别并行

```bash
python3 fetch_candidates.py US ./scan-US 8
docker run --rm --dns 8.8.8.8 --memory 48m --cap-add NET_ADMIN \
  --device /dev/net/tun -e VPSIP=<机器公网IP> -e LIMIT=8 \
  -v ./scan-US:/scan --entrypoint sh homeline-gw:v2 /scan.sh US
```

扫到 `WORKING US try=N` 才算有节点，`NONE` 就是这个国家
池子空了，等会儿再扫，绝不拿别国的节点冒充。

3. 起网关容器

```bash
docker run -d --name vpngw-us-gateway --restart unless-stopped \
  --dns 8.8.8.8 --memory 48m --cap-add NET_ADMIN --device /dev/net/tun \
  --env-file ./us/.env -p 127.0.0.1:12081:1080 \
  -v ./us/ovpn:/vpn homeline-gw:v2
```

`.env` 里只有网关自己的 SOCKS 用户名密码，自己随机生成，
`ovpn/client.ovpn` 就是扫到的那个节点文件。

4. 起 Xray，入站 WS，出站走网关 SOCKS

```json
{
  "inbounds": [{
    "port": 30001, "listen": "0.0.0.0", "protocol": "vless",
    "settings": { "clients": [{ "id": "<UUID>" }], "decryption": "none" },
    "streamSettings": { "network": "ws", "wsSettings": { "path": "/<WS_PATH>" } }
  }],
  "outbounds": [{
    "protocol": "socks",
    "settings": { "servers": [{
      "address": "127.0.0.1", "port": 12081,
      "users": [{ "user": "<GATE_USER>", "pass": "<GATE_PASS>" }]
    }] }
  }]
}
```

UUID 用 `xray uuid` 生成，WS 路径自己随机，别用默认路径。
配好先 `xray run -test -c xray-config.json`，必须 Configuration OK，
再用 systemd 跑起来并 `enable`。

5. port publish 这个 Xray 端口，拿到公网 HTTPS 地址

```
https://<端口>-<机器ID>.instabox.tech
```

公网测这个地址应该是 404（约 1 秒），不是 502（机器停了），
也不是超时。客户端链接格式：

```
vless://<UUID>@<端口>-<机器ID>.instabox.tech:443?encryption=none&security=tls&sni=<上面那个域名>&fp=chrome&type=ws&host=<上面那个域名>&path=/<WS_PATH>#Home-US
```

## 验线四步，缺一步都不算通

- V1：SOCKS 探针能返回 IP。
- V2：出口 IP 和机器自己的 IP 不一样，没漏。
- V3：出口国家和你要的国家一致，且是 ISP 不是机房。
  不一致就换节点或换国家，绝不改名冒充。
- V4：真机真网络扫码导入，打开 IP 页能通。只在服务器上
  测 SOCKS 不算 V4。

## 节点死了怎么办

不用重扫码。只换网关里的节点文件，重启网关容器就行：

```bash
cp scan-US/working-US.ovpn ./us/ovpn/client.ovpn
docker restart vpngw-us-gateway
```

Xray、UUID、端口、WS 路径、port publish、客户端链接全不动，
只是出口 IP 变成新节点。只有换国家、改 Xray、删建机器时
才需要新链接新二维码。

完整模板和脚本（无任何秘密，只有占位符）：
https://github.com/pierpont-morgan/homeline-instabox
