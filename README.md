# homeline-instabox

Residential-exit (家宽) lines on an InstaBox cloud computer, adapted from
free-dam-man/homeline-residential-exit-skill.

InstaBox is not a normal VPS. Two adaptations are mandatory, and they are
the whole point of this repo:

1. **No Reality.** On InstaBox, a Reality inbound does not survive the
   edge. Use VLESS + WebSocket, plain WS inside the machine, TLS
   terminated by InstaBox's port-publish edge.
2. **No raw TCP/UDP inbound.** InstaBox port publish gives you an HTTPS
   public address per port, up to 16 ports per machine, only while the
   machine is running. It does not give you a public IP, a UDP port, or
   SSH from the internet. WireGuard (UDP) therefore does not work on
   InstaBox. VLESS + WS over HTTPS does.

## What is deployed, per country line

- One VPN Gate gateway container (`gateway/`): OpenVPN client to a
  volunteer node, microsocks SOCKS5 bound to `127.0.0.1` only, an
  in-container iptables OUTPUT DROP kill switch, and a minute watchdog.
  It never exposes SOCKS to the internet.
- One standalone Xray (`xray-config.template.json`): VLESS + WS inbound
  on a high port, outbound through the gateway's SOCKS. No panel, no
  subscription server.
- One systemd unit (`xray-home.service.template`): Xray only. The
  gateway is a Docker container with `--restart unless-stopped`.
- One port publish: the Xray port, published to
  `https://<PORT>-<MACHINE_ID>.instabox.tech`. The client connects to
  port 443 on that hostname, with TLS SNI = that hostname, WS path =
  the path in the line's `.env`.

Measured, on 2026-10-29, from an InstaBox dev (Debian 13, Docker,
systemd, TUN available, 4 vCPU / 8 GiB):

- US: SOCKS exit 174.87.100.177, California, Charter / Spectrum,
  hosting=false, proxy=false. Pool was 2 nodes; try 1 AUTH_FAILED,
  try 2 worked. US pool is thin and fluctuates; a failed scan is normal.
- KR: SOCKS exit 118.41.251.223, Daegu, Korea Telecom,
  hosting=false, proxy=false. Pool was 25 nodes; try 1 worked.

Exit IPs change when a node dies and is swapped. The client link does
not change on a node swap (see below).

## Prerequisites on the InstaBox machine

- Debian, Docker, systemd, `/dev/net/tun` openable.
- The iCloud workbench or any other production service on the same
  machine is untouched: use only new ports, new container names,
  new systemd units, new directories. Never publish port 5901 (VNC).
- Port publish is done with the InstaBox API, not from inside the
  machine. Example, with the machine id from `ibx.py get dev`:
  `POST /v1/machines/<MACHINE_ID>/ports {"port": <XRAY_PORT>}`

## Deploy one line

Replace every `<...>`; never commit a filled-in config, `.env`,
`client.txt`, `ovpn/` or QR image. Those live only on the machine,
mode 600.

```bash
BASE=/data/home/homeline
NAME=us            # or kr, lowercase
CC=US              # uppercase country, must match the line
SOCKS_PORT=12081   # loopback only, first free
XRAY_PORT=30001    # first free high port, to be port-published

# 1. Gateway image, once per machine
mkdir -p $BASE/gateway
cp gateway/Dockerfile gateway/entrypoint.sh gateway/scan.sh $BASE/gateway/
docker build -t homeline-gw:v2 $BASE/gateway/

# 2. Find a working node, one country at a time
mkdir -p $BASE/scan-$CC
printf 'vpn\nvpn\n' > $BASE/scan-$CC/auth.txt && chmod 600 $BASE/scan-$CC/auth.txt
python3 fetch_candidates.py $CC $BASE/scan-$CC 8
docker run --rm --dns 8.8.8.8 --memory 48m --cap-add NET_ADMIN \
  --device /dev/net/tun -e VPSIP=<DEV_PUBLIC_IP> -e LIMIT=8 \
  -v $BASE/scan-$CC:/scan --entrypoint sh homeline-gw:v2 /scan.sh $CC
# Output WORKING <CC> try=N writes scan-<CC>/working-<CC>.ovpn
# Output NONE = no usable node for that country right now; retry later,
# do not silently substitute another country.

# 3. Line dir, credentials, gateway
mkdir -p $BASE/$NAME/ovpn
printf 'GATE_USER=gw%s\nGATE_PASS=%s\n' "$(openssl rand -hex 3)" \
  "$(openssl rand -hex 16)" > $BASE/$NAME/.env && chmod 600 $BASE/$NAME/.env
cp $BASE/scan-$CC/working-$CC.ovpn $BASE/$NAME/ovpn/client.ovpn
cp $BASE/scan-$CC/auth.txt $BASE/$NAME/ovpn/auth.txt && chmod 600 $BASE/$NAME/ovpn/*
docker run -d --name vpngw-$NAME-gateway --restart unless-stopped \
  --dns 8.8.8.8 --memory 48m --cap-add NET_ADMIN --device /dev/net/tun \
  --env-file $BASE/$NAME/.env -p 127.0.0.1:$SOCKS_PORT:1080 \
  -v $BASE/$NAME/ovpn:/vpn homeline-gw:v2

# 4. Xray: install, config from template, test, systemd
mkdir -p $BASE/$NAME/xray
curl -fsSL -o /tmp/xray.zip \
  https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip
unzip -q -o /tmp/xray.zip xray -d $BASE/$NAME/xray && chmod +x $BASE/$NAME/xray/xray
# Fill xray-config.template.json -> $BASE/$NAME/xray-config.json
# (UUID from $BASE/$NAME/xray/xray uuid, WS_PATH from openssl rand -hex 8,
#  GATE_USER/GATE_PASS from $BASE/$NAME/.env, ports as numbers, not strings)
$BASE/$NAME/xray/xray run -test -c $BASE/$NAME/xray-config.json
# Must print Configuration OK and grep -c '<' xray-config.json must print 0.
sudo cp xray-home.service.template /etc/systemd/system/xray-home-$NAME.service
# Edit ExecStart paths in the unit to $BASE/$NAME, then:
sudo systemctl daemon-reload && sudo systemctl enable --now xray-home-$NAME

# 5. Port publish the Xray port, then test the public URL
# Public URL should answer 404 in ~1-2s (Xray WS with no WS upgrade),
# not 502 (machine suspended) and not a timeout.
curl -s -o /dev/null -w "%{http_code}\n" \
  https://$XRAY_PORT-<MACHINE_ID>.instabox.tech/

# 6. Client link, on the machine only, mode 600
# vless://<UUID>@<XRAY_PORT>-<MACHINE_ID>.instabox.tech:443?encryption=none&security=tls&sni=<HOST>&fp=chrome&type=ws&host=<HOST>&path=/<WS_PATH>#Home-<CC>
# where <HOST> = <XRAY_PORT>-<MACHINE_ID>.instabox.tech
# Generate a QR from that link on the machine, scan it with Shadowrocket
# / FlClash, and confirm the IP page shows the measured exit.
```

## Verify before calling a line done

- V1: SOCKS probe returns an IP.
- V2: SOCKS exit differs from the machine's own public IP (no leak).
- V3: Geolocation of the exit matches the requested country and is
  not a datacenter/hosting ISP. On any mismatch, ask the operator:
  accept the actual exit, swap to the next node, or change country.
  Never relabel another country's exit.
- V4: A real client on a real device/network imports the exact link
  and loads an IP page through it. A server-side SOCKS probe alone
  is not V4. A client test from a proxy-only cloud VM may fail on
  TLS dial for network reasons unrelated to the line; that failure
  alone does not prove the line is broken.

## When a node dies

Do not re-issue the link or QR. Swap only the gateway's node:

```bash
# Rescan for the same country, excluding the dead node, then:
cp scan-<CC>/working-<CC>.ovpn $BASE/$NAME/ovpn/client.ovpn
docker restart vpngw-$NAME-gateway
# Xray, UUID, port, WS path, port publish and client link are untouched.
# The exit IP changes; the country should stay the same.
```

A new link/QR is needed only when the Xray UUID, port, WS path,
port publish hostname, or machine id changes, or when the country
line itself changes.

## What is not in this repo, on purpose

- No `.env`, no `client.txt`, no `ovpn/`, no QR images, no UUIDs,
  no WS paths, no SOCKS passwords, no machine ids, no port publish
  URLs from a real deployment. Templates use placeholders only.
- No panel, no subscription server, no Reality config, no
  WireGuard config. Those do not fit InstaBox's HTTPS-only edge.

## Credits

Gateway, scan and candidate-fetch logic are adapted from
free-dam-man/homeline-residential-exit-skill (public repo), with the
InstaBox WS + port publish adaptation above. Xray is XTLS/Xray-core.
VPN Gate nodes are volunteers; a line's exit will change and a
country's pool can be empty for hours.
