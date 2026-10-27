# esp32-c3-adblock

一个运行在 2 美元 ESP32-C3 上的 Pi-hole 风格 DNS 广告拦截器——无需 PSRAM。

> 📰 Featured on [Tom's Hardware](https://www.tomshardware.com/networking/clever-hacker-fits-537-000-domains-in-a-tiny-usd5-esp32-ad-blocking-dongle-firmware-uses-only-around-50kb-of-ram-and-can-answer-blocked-lookups-in-10-milliseconds), [XDA Developers](https://www.xda-developers.com/this-tiny-esp32-powered-gadget-blocks-537000-domains-only-uses-50kb-of-ram/), and [Korben](https://korben.info/en/half-million-ad-blocking-domains-50kb-ram-esp32.html).

所有人都忽略的诀窍：你不需要把拦截名单放在 RAM 里。把域名存成排序好的 40 位哈希值放在 Flash 中，然后二分查找。14 万+ 域名只占约 0.7 MB Flash，匹配耗时约 10 ms，仅用 约 50 KB RAM。
```
查询进入 ──▶ 提取域名 ──▶ FNV-1a 哈希（含父级后缀）
         ──▶ 在 Flash 哈希表中二分查找
              ├─ 命中  ──▶ 应答 0.0.0.0   （黑洞）
              └─ 未命中 ──▶ 转发给上游解析器，中继回复
```

## Why this is interesting

大多数 ESP32 DNS 黑洞方案会把拦截名单（域名字符串）加载进 RAM，因此需要 PSRAM。本项目改为在 Flash 中存储固定的 5 字节（40 位）哈希值：

	字符串存 RAM 方案	本方案（哈希存 Flash）
硬件	ESP32 + PSRAM（约 8 美元）	ESP32-C3，无需 PSRAM（约 2 美元）
14.1 万域名	约 2.5 MB RAM	0.67 MB Flash
RAM 占用	几乎全部	约 50 KB
查找	字符串比较	约 18 次 Flash 读取（含 Wi-Fi RTT 约 10 ms）
碰撞	不适用	14.1 万时 0 次（53.7 万时 1 次）

为什么是 40 位？ 这是该 Flash 预算下的最佳平衡点。碰撞遵循生日悖论——14.1 万域名时约 0 次，53.7 万时约 1 次（即一个倒霉域名被过度拦截）。降到 32 位能省 20% Flash，但 25 万时会有约 7 次碰撞；升到 64 位则每个域名浪费 3 字节，去解决一个你根本不存在的问题。

同样的技巧在更大的芯片上也成立——它不是 C3 的变通方案。在 16 MB 的 ESP32-S3 上，这些哈希可容纳 约 270 万域名，而 8 MB PSRAM 存字符串只能容纳约 46.6 万。哈希存 Flash 基本上在所有场景都优于字符串存 PSRAM；C3 只是让这一点无可辩驳。

## Hardware

任意 ESP32-C3 开发板（在 C3 SuperMini 上测试过），4 MB Flash，无需 PSRAM

经典 ESP32（DevKit / WROOM，4 MB）也可构建：pio run -e esp32dev -t upload（社区贡献，已编译测试；C3 是实测目标）

用稳定的 USB 电源供电（手机充电器或路由器的 USB 口）。劣质/松动的 USB-C→A 转接头会在 Wi-Fi 发射时导致射频欠压。

一个 USB-A → USB-C 转接头可以让它直接插进大多数路由器背面的空闲 USB 口——无需电源、无需额外盒子。

### Enclosure

A printable case for the C3 SuperMini: [`hardware/esp32-c3-supermini-enclosure.stl`](hardware/esp32-c3-supermini-enclosure.stl)

打印说明：

无需支撑；0.2 mm 层高、约 15% 填充即可。

保持天线端空旷。 C3 的 PCB 天线是 USB-C 口对侧短边上的锯齿状走线——不要把它埋进实心塑料里或在附近放金属，否则 RSSI 会变差。

保持通风口敞开：板子空闲时约 45–55 °C。

## Build & flash (PlatformIO)

One USB flash to get going — after that, **firmware and blocklist both update over WiFi** (see below).

> ⚠️ Use a **current PlatformIO** — the VSCode PlatformIO extension's bundled core, or
> `pip install -U platformio` in a venv. The distro/apt `platformio` package (e.g. 4.3.4) is
> too old and fails with `AttributeError: ... 'resultcallback'` (issue #4). A one-click browser installer is on the way (hosting TBD).


#复制 secrets 模板（已被 gitignore，仅本地保留）并编辑：
#WIFI_SSID / WIFI_PASS 是可选的——保留占位符，改用设备上的
#配网门户（见下文）。
#WEB_USER / WEB_PASS / OTA_PASS 不是可选的：它们保护仪表盘的状态变更
#端点（/ban、/addblock、/upload、/update、/setupdate、/forgetwifi）
#和网络 OTA。请设置真实值——这些端点以前对局域网内任何人都是敞开的。
```bash
# 1. copy the secrets template (gitignored, stays local) and edit it:
#    - WIFI_SSID / WIFI_PASS are optional — leave the placeholders and use the
#      on-device setup portal instead (below).
#    - WEB_USER / WEB_PASS / OTA_PASS are NOT optional: they gate the dashboard's
#      state-changing endpoints (/ban, /addblock, /upload, /update, /setupdate,
#      /forgetwifi) and network OTA. Pick real values — these used to be wide
#      open to anyone on the LAN.
cp src/secrets.example.h src/secrets.h
#    then edit src/secrets.h

# 2. 构建拦截名单哈希表（默认 = StevenBlack base + Hagezi Light，
#    约 10 万条，对 WhatsApp/社交安全）
# 2. build the blocklist hash table (default = StevenBlack base + Hagezi Light,
#    ~100k entries, WhatsApp/social safe)
python3 tools/build_blocklist.py data/blocklist.bin

# 3. 烧录固件 + 拦截名单文件系统（唯一一次 USB 烧录）
# 3. flash firmware + the blocklist filesystem (the one and only USB flash)
pio run -t upload
pio run -t uploadfs

# 4. 观察启动，记下 IP / 打开仪表盘
# 4. watch it boot, note the IP / open the dashboard
pio device monitor          # -> http://c3adblock.local
```

使用自己的拦截名单
build_blocklist.py OUT.bin [SOURCE ...] 接受任意混合的 URL 和本地文件，支持以下格式：

hosts 文件——0.0.0.0 ads.example.com tracker.example.com（行内所有域名都会包含）

纯域名列表——每行一个域名

AdGuard / Adblock 基础规则——||ads.example.com^ 拦截，@@||ok.example.com^ 移除某个域名（例如镜像 AdGuard Home 的白名单）

被拦截的域名也会拦截其子域名。DNS 哈希列表无法表达的规则（正则、通配符、$ 修饰符、外观类 ## 规则）会被跳过并计数。@@ 规则只能取消拦截该精确条目——它无法从被拦截的父域名中单独放行子域名。如果某个来源无法下载，构建会停止而不是静默生成更小的列表（用 --allow-missing 覆盖）。

Wi-Fi 设置（无需重新烧录）
如果无法连接（或你从未设置 secrets.h），它会启动一个开放热点 C3-AdBlock-XXXX 并带强制门户——用手机加入，选择你的网络，输入密码，完成。以后要换网络：点击仪表盘上的 Forget WiFi，或上电时按住 BOOT 按钮，配网门户会重新出现。（/forgetwifi 现在需要认证，所以不再是可以直接访问的裸 URL——见下方安全说明。）

The dashboard at **http://c3adblock.local** does it all:

拦截名单——把新构建的 blocklist.bin 拖进 Blocklist → Upload，或在 Remote auto-update 下设置 URL，设备会按计划拉取预构建的 blocklist.bin。默认名单由 GitHub Actions 每周一重新构建并发布到稳定 URL，粘贴一次即可让设备自行保持更新：
https://github.com/M-Abozaid/esp32-c3-adblock/releases/download/blocklist/blocklist.bin

固件——在 Firmware → OTA update 下上传 .pio/build/c3/firmware.bin；设备会校验并重启进入新镜像。或从命令行通过 Wi-Fi 推送：
  ```bash
  pio run -t upload --upload-port c3adblock.local --upload-protocol espota
  ```

4 MB Flash 的取舍： 固件 OTA 需要两个 app 分区，这给拦截名单留下约 1.3 MB（最多约 25 万域名）。激进的 53.7 万“终极”名单只能适配单 app 分区表（无固件 OTA）。在 partitions.csv 中选择你的取舍。

## Security

The dashboard's read-only view (`/`, `/stats.json`) stays open, but every
state-changing endpoint requires **HTTP Basic Auth** (`WEB_USER`/`WEB_PASS` from
`secrets.h`):

- `/ban`, `/addblock`, `/unblock`, `/forgetwifi`
- `/upload`, `/update` (blocklist and firmware OTA)
- `/setupdate`, `/fetchnow`

Network OTA (`ArduinoOTA`, e.g. `pio run -t upload --upload-port c3adblock.local
--upload-protocol espota`) requires `OTA_PASS` from the same file.

Without this, anyone who could reach the device on the LAN could reflash it
with arbitrary firmware or rewrite the blocklist with zero credentials — worth
knowing given the device sits in the path of every DNS query on your network.
Custom blocked-domain names are also HTML-escaped before being rendered on the
dashboard, closing a stored-XSS path where a domain string containing markup
(added via `/addblock`) would otherwise execute in the viewing browser.

**Basic Auth here is a LAN-trust-boundary control, not encryption.** Everything
is plain HTTP on :80 — this chip has no realistic budget to run a TLS server.
Basic Auth credentials are base64 (not encrypted) and sent on every authenticated
request; anyone who can already sniff your LAN traffic (open/guest WiFi, ARP
spoofing) can read them off the wire. This hardens against the common case —
another device on your network hitting the API with no credentials at all, or a
browser tab CSRF'ing it — not against an on-path network attacker.

**CSRF via cached Basic Auth:** browsers auto-attach cached Basic Auth
credentials to *any* subsequent request to an already-authenticated origin —
including one triggered by a totally unrelated page the same browser visits
later (e.g. `<img src="http://c3adblock.local/forgetwifi">`, no JS required).
That would let any webpage silently drive this API once you've logged into the
dashboard once, regardless of who's on your LAN. Every mutating endpoint above
now also requires a custom `X-Requested-With: c3-adblock` header, which a plain
`<img>`/auto-submitted `<form>` CSRF can't attach (only same-origin `fetch()`
can, which is what the dashboard's own JS does) — this is why `/forgetwifi` is
no longer a bare URL you can visit directly; use the dashboard button instead.

**Default credentials:** if `secrets.h` still has the placeholder
`CHANGE_ME_WEB_PASSWORD` / `CHANGE_ME_OTA_PASSWORD` values from
`secrets.example.h`, the device boots with a "password" that's public (it's
sitting in this repo's example file). The firmware logs a warning over serial
and shows a banner on the dashboard when this is the case — but it will still
boot and run, so don't skip setting real values in `secrets.h` before trusting
this on a network you don't fully control.

Out of scope for now: the WiFi setup portal's access point (`C3-AdBlock-XXXX`)
is still open (unencrypted) by design — it needs to be joinable without knowing
a password first. The real WiFi password you type into the portal is only as
safe as that local radio link during the brief setup window.

## Use it

Point a device's DNS at the C3's IP, or add it as a **secondary resolver** behind
your main DNS. Test:

```bash
dig @<c3-ip> doubleclick.net   # -> 0.0.0.0  (blocked)
dig @<c3-ip> github.com        # -> real IP  (forwarded)
```

## Gotchas (learned the hard way)

- **ModemManager** (default on Fedora/Ubuntu) grabs `/dev/ttyACM0` and toggles
  DTR/RTS, which **resets the C3** and blocks serial. Fix:
  ```bash
  sudo systemctl stop ModemManager
  echo 'ATTRS{idVendor}=="303a", ENV{ID_MM_DEVICE_IGNORE}="1"' | sudo tee /etc/udev/rules.d/99-esp-no-modemmanager.rules
  sudo udevadm control --reload-rules && sudo udevadm trigger
  ```
- The C3's USB-Serial-JTAG console can swallow early boot output until the host
  connects (`while(!Serial)` helps).
- DNS clients add an **EDNS OPT** record; a blocked reply must contain only the
  question + answer (ANCOUNT=1, NSCOUNT=ARCOUNT=0) or it's malformed.

## Done / how it could grow

- ✅ Web dashboard — per-client block/allow counts, ban a client, add custom domains
- ✅ mDNS (`c3adblock.local`) for discovery
- ✅ OTA — firmware + blocklist update over WiFi, plus scheduled remote blocklist pulls
- ✅ Captive-portal WiFi setup (no hardcoded creds) + one-click browser web-installer
- ⬜ Bucketed prefix index — ~18 flash reads/lookup → ~1–2 (issue #3), the throughput win
- ⬜ Act as the DHCP server (hand itself out as DNS) for true plug-and-play

## Credits

Inspired by [s60sc/ESP32_AdBlocker](https://github.com/s60sc/ESP32_AdBlocker) — the
"answer 0.0.0.0 for blocklisted domains" idea. This is an independent from-scratch
implementation focused on the hash-in-flash optimization for PSRAM-less chips.

## License

MIT — see [LICENSE](LICENSE).
#   e s p 3 2 - c 3 - a d b l o c k  
 