esp32-c3-adblock
一个运行在 2 美元 ESP32-C3 上的 Pi-hole 风格 DNS 广告拦截器——无需 PSRAM。

📰 曾被 Tom's Hardware、XDA Developers 和 Korben 报道。

所有人都忽略的诀窍：你不需要把拦截名单放在 RAM 里。把域名存成排序好的 40 位哈希值放在 Flash 中，然后二分查找。14 万+ 域名只占约 0.7 MB Flash，匹配耗时约 10 ms，仅用约 50 KB RAM。

text
查询进入 ──▶ 提取域名 ──▶ FNV-1a 哈希（含父级后缀）
         ──▶ 在 Flash 哈希表中二分查找
              ├─ 命中  ──▶ 应答 0.0.0.0   （黑洞）
              └─ 未命中 ──▶ 转发给上游解析器，中继回复
为什么这个方案有意思
大多数 ESP32 DNS 黑洞方案会把拦截名单（域名字符串）加载进 RAM，因此需要 PSRAM。本项目改为在 Flash 中存储固定的 5 字节（40 位）哈希值：

字符串存 RAM 方案	本方案（哈希存 Flash）
硬件	ESP32 + PSRAM（约 8 美元）	ESP32-C3，无需 PSRAM（约 2 美元）
14.1 万域名	约 2.5 MB RAM	0.67 MB Flash
RAM 占用	几乎全部	约 50 KB
查找	字符串比较	约 18 次 Flash 读取（含 Wi-Fi RTT 约 10 ms）
碰撞	不适用	14.1 万时 0 次（53.7 万时 1 次）
为什么是 40 位？ 这是该 Flash 预算下的最佳平衡点。碰撞遵循生日悖论——14.1 万域名时约 0 次，53.7 万时约 1 次（即一个倒霉域名被过度拦截）。降到 32 位能省 20% Flash，但 25 万时会有约 7 次碰撞；升到 64 位则每个域名浪费 3 字节，去解决一个你根本不存在的问题。

同样的技巧在更大的芯片上也成立——它不是 C3 的变通方案。在 16 MB 的 ESP32-S3 上，这些哈希可容纳约 270 万域名，而 8 MB PSRAM 存字符串只能容纳约 46.6 万。哈希存 Flash 基本上在所有场景都优于字符串存 PSRAM；C3 只是让这一点无可辩驳。

硬件
任意 ESP32-C3 开发板（在 C3 SuperMini 上测试过），4 MB Flash，无需 PSRAM

经典 ESP32（DevKit / WROOM，4 MB）也可构建：pio run -e esp32dev -t upload（社区贡献，已编译测试；C3 是实测目标）

用稳定的 USB 电源供电（手机充电器或路由器的 USB 口）。劣质/松动的 USB-C→A 转接头会在 Wi-Fi 发射时导致射频欠压。

一个 USB-A → USB-C 转接头可以让它直接插进大多数路由器背面的空闲 USB 口——无需电源、无需额外盒子。

外壳
C3 SuperMini 的可打印外壳：hardware/esp32-c3-supermini-enclosure.stl

打印说明：

无需支撑；0.2 mm 层高、约 15% 填充即可。

保持天线端空旷。 C3 的 PCB 天线是 USB-C 口对侧短边上的锯齿状走线——不要把它埋进实心塑料里或在附近放金属，否则 RSSI 会变差。

保持通风口敞开：板子空闲时约 45–55 °C。

构建与烧录（PlatformIO）
一次 USB 烧录即可启动——之后固件和拦截名单都可通过 Wi-Fi 更新（见下文）。

⚠️ 使用较新的 PlatformIO——VSCode PlatformIO 插件自带的核，或在 venv 中 pip install -U platformio。发行版/apt 的 platformio 包（如 4.3.4）太旧，会报 AttributeError: ... 'resultcallback'（issue #4）。一键浏览器安装器正在开发中（托管待定）。

bash
# 1. 复制 secrets 模板（已被 gitignore，仅本地保留）并编辑：
#    - WIFI_SSID / WIFI_PASS 是可选的——保留占位符，改用设备上的
#      配网门户（见下文）。
#    - WEB_USER / WEB_PASS / OTA_PASS 不是可选的：它们保护仪表盘的状态变更
#      端点（/ban、/addblock、/upload、/update、/setupdate、/forgetwifi）
#      和网络 OTA。请设置真实值——这些端点以前对局域网内任何人都是敞开的。
cp src/secrets.example.h src/secrets.h
#    然后编辑 src/secrets.h

# 2. 构建拦截名单哈希表（默认 = StevenBlack base + Hagezi Light，
#    约 10 万条，对 WhatsApp/社交安全）
python3 tools/build_blocklist.py data/blocklist.bin

# 3. 烧录固件 + 拦截名单文件系统（唯一一次 USB 烧录）
pio run -t upload
pio run -t uploadfs

# 4. 观察启动，记下 IP / 打开仪表盘
pio device monitor          # -> http://c3adblock.local
使用自己的拦截名单
build_blocklist.py OUT.bin [SOURCE ...] 接受任意混合的 URL 和本地文件，支持以下格式：

hosts 文件——0.0.0.0 ads.example.com tracker.example.com（行内所有域名都会包含）

纯域名列表——每行一个域名

AdGuard / Adblock 基础规则——||ads.example.com^ 拦截，@@||ok.example.com^ 移除某个域名（例如镜像 AdGuard Home 的白名单）

被拦截的域名也会拦截其子域名。DNS 哈希列表无法表达的规则（正则、通配符、$ 修饰符、外观类 ## 规则）会被跳过并计数。@@ 规则只能取消拦截该精确条目——它无法从被拦截的父域名中单独放行子域名。如果某个来源无法下载，构建会停止而不是静默生成更小的列表（用 --allow-missing 覆盖）。

Wi-Fi 设置（无需重新烧录）
如果无法连接（或你从未设置 secrets.h），它会启动一个开放热点 C3-AdBlock-XXXX 并带强制门户——用手机加入，选择你的网络，输入密码，完成。以后要换网络：点击仪表盘上的 Forget WiFi，或上电时按住 BOOT 按钮，配网门户会重新出现。（/forgetwifi 现在需要认证，所以不再是可以直接访问的裸 URL——见下方安全说明。）

空中升级（不再需要 USB）
http://c3adblock.local 的仪表盘全部搞定：

拦截名单——把新构建的 blocklist.bin 拖进 Blocklist → Upload，或在 Remote auto-update 下设置 URL，设备会按计划拉取预构建的 blocklist.bin。默认名单由 GitHub Actions 每周一重新构建并发布到稳定 URL，粘贴一次即可让设备自行保持更新：
https://github.com/M-Abozaid/esp32-c3-adblock/releases/download/blocklist/blocklist.bin

固件——在 Firmware → OTA update 下上传 .pio/build/c3/firmware.bin；设备会校验并重启进入新镜像。或从命令行通过 Wi-Fi 推送：

bash
pio run -t upload --upload-port c3adblock.local --upload-protocol espota
4 MB Flash 的取舍： 固件 OTA 需要两个 app 分区，这给拦截名单留下约 1.3 MB（最多约 25 万域名）。激进的 53.7 万“终极”名单只能适配单 app 分区表（无固件 OTA）。在 partitions.csv 中选择你的取舍。

安全
仪表盘的只读视图（/、/stats.json）保持开放，但所有状态变更端点都需要 HTTP Basic Auth（来自 secrets.h 的 WEB_USER/WEB_PASS）：

/ban、/addblock、/unblock、/forgetwifi

/upload、/update（拦截名单和固件 OTA）

/setupdate、/fetchnow

网络 OTA（ArduinoOTA，如 pio run -t upload --upload-port c3adblock.local --upload-protocol espota）需要同一文件中的 OTA_PASS。

没有这些，局域网内任何能访问设备的人都能零凭据重刷任意固件或重写拦截名单——考虑到设备位于你网络每个 DNS 查询的路径上，这一点值得知道。自定义拦截域名在仪表盘渲染前也会做 HTML 转义，关闭了一条存储型 XSS 路径（否则通过 /addblock 添加的含标记的域名字符串会在查看浏览器中执行）。

这里的 Basic Auth 是局域网信任边界控制，不是加密。 所有内容都是 :80 上的明文 HTTP——这颗芯片没有现实预算跑 TLS 服务器。Basic Auth 凭据是 base64（非加密）且每个认证请求都会发送；任何能嗅探你局域网流量的人（开放/访客 Wi-Fi、ARP 欺骗）都能从线路上读到它们。这加固的是常见情况——你网络上的另一台设备零凭据访问 API，或浏览器标签页 CSRF——而不是路径上的网络攻击者。

缓存 Basic Auth 导致的 CSRF： 浏览器会自动把缓存的 Basic Auth 凭据附加到对已认证源的任何后续请求——包括同一浏览器稍后访问的完全无关页面触发的请求（例如 <img src="http://c3adblock.local/forgetwifi">，无需 JS）。这会让任何网页在你登录仪表盘一次后静默驱动此 API，无论谁在你的局域网上。上述每个变更端点现在还需要自定义 X-Requested-With: c3-adblock 头，普通 <img>/自动提交的 <form> CSRF 无法附加（只有同源 fetch() 可以，这正是仪表盘自己的 JS 所做的）——这就是为什么 /forgetwifi 不再是可以直接访问的裸 URL；请使用仪表盘按钮。

默认凭据： 如果 secrets.h 仍是 secrets.example.h 中的占位值 CHANGE_ME_WEB_PASSWORD / CHANGE_ME_OTA_PASSWORD，设备会以一个公开的“密码”启动（它就放在本仓库的示例文件里）。此时固件会通过串口打印警告并在仪表盘显示横幅——但它仍会启动运行，所以在不完全受控的网络上信任它之前，不要跳过在 secrets.h 中设置真实值。

目前不在范围内：Wi-Fi 配网门户的接入点（C3-AdBlock-XXXX）按设计仍是开放的（未加密）——它需要在不预先知道密码的情况下可加入。你在门户中输入的真实 Wi-Fi 密码，在短暂配网窗口内仅与那段本地无线电链路同等安全。

使用它
把某设备的 DNS 指向 C3 的 IP，或把它作为主 DNS 后面的辅助解析器。测试：

bash
dig @<c3-ip> doubleclick.net   # -> 0.0.0.0  （已拦截）
dig @<c3-ip> github.com        # -> 真实 IP  （已转发）
踩过的坑（血泪教训）
ModemManager（Fedora/Ubuntu 默认启用）会抢占 /dev/ttyACM0 并切换 DTR/RTS，导致 C3 复位并阻塞串口。修复：

bash
sudo systemctl stop ModemManager
echo 'ATTRS{idVendor}=="303a", ENV{ID_MM_DEVICE_IGNORE}="1"' | sudo tee /etc/udev/rules.d/99-esp-no-modemmanager.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
C3 的 USB-Serial-JTAG 控制台在主机连接前会吞掉早期启动输出（while(!Serial) 有帮助）。

DNS 客户端会添加 EDNS OPT 记录；被拦截的回复必须只包含问题 + 答案（ANCOUNT=1, NSCOUNT=ARCOUNT=0），否则格式错误。

已完成 / 未来可扩展
✅ Web 仪表盘——按客户端拦截/放行计数、封禁客户端、添加自定义域名

✅ mDNS（c3adblock.local）用于发现

✅ OTA——固件 + 拦截名单通过 Wi-Fi 更新，加上定时远程拉取拦截名单

✅ 强制门户 Wi-Fi 配网（无需硬编码凭据）+ 一键浏览器网页安装器

⬜ 分桶前缀索引——每次查找约 18 次 Flash 读取 → 约 1–2 次（issue #3），吞吐量提升

⬜ 充当 DHCP 服务器（把自己作为 DNS 分发出去）实现真正即插即用

致谢
灵感来自 s60sc/ESP32_AdBlocker——"对拦截名单域名应答 0.0.0.0" 的想法。本项目是独立从零实现，聚焦于无 PSRAM 芯片的哈希存 Flash 优化。