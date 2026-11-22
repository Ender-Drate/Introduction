# TechXPasskey 交接文档

> 最后更新：2026-11-22  
> 固件版本：`0.1.0`  
> 源码仓库：<https://github.com/Ender-Drate/esp32-passkey>  
> 本文档只写架构、操作和注意事项，**不包含 PIN、KEK、DEK、主密码、私钥**。这些由持有人线下交接。

---

## 1. 项目是什么

TechXPasskey 是一套“两块 ESP32 的 FIDO2 物理 Passkey”：

- **主板**：ESP32-S3-N16R8，跑完整 CTAP2.1 / U2F 认证器，USB HID + CDC + BLE。
- **副板**：第二块 ESP32-S3，保存 link key 和 **KEK**；主板只有副板在场并完成挑战后，才能解开 DEK、解密凭据库、执行签名。
- **管理器**：本地 Python + Web UI，管理 Passkey、PIN、密码库、TOTP、OTA、副板。
- **Android**：BLE 伴生 App，管理 PIN 和凭据。
- **外壳**：Blender MCP 生成的 MAIN/KEY 两套 3D 打印外壳，面包板尺寸。

核心设计：**主板上没有 Passkey 私钥明文**。`/creds.bin` 是 PKC2 密文，DEK 被副板 KEK 包裹；没有副板，flash dump 也解不开。

---

## 2. 代码与文档索引

| 内容 | 位置 |
|---|---|
| 主固件 | `esp32-passkey/firmware/` |
| 副板固件 | `esp32-passkey/firmware-key/` |
| 管理器 | `esp32-passkey/manager/` |
| Android | `esp32-passkey/android/` |
| 外壳 | `esp32-passkey/enclosure/` |
| 测试 | `esp32-passkey/tests/` |
| 设备管理 CLI | `esp32-passkey/scripts/pkdev.py` |
| 完整说明 | `esp32-passkey/docs/TechXPassKey.md` |
| 外壳说明 | `esp32-passkey/enclosure/README.md` |
| 副板说明 | `esp32-passkey/firmware-key/README.md` |
| Android 说明 | `esp32-passkey/android/README.md` |

主固件关键文件：

```
firmware/src/main.cpp         启动、USB、BLE、LOOP
firmware/src/pk_ctap2.cpp     CTAP2.1 命令、PIN、credMgmt、largeBlobs
firmware/src/pk_u2f.cpp       CTAP1/U2F
firmware/src/pk_ctaphid.cpp   USB HID CTAPHID
firmware/src/pk_mgmt.cpp      CDC 管理协议 + OTA
firmware/src/pk_usb.cpp       TinyUSB 复合设备
firmware/src/pk_ble.cpp       BLE GATT 0xFFFD
firmware/src/pk_store.cpp     加密凭据库、DEK/KEK、largeBlob
firmware/src/pk_interlock.cpp 副板 UART 挑战、配对、KEK、换板/清除
firmware/src/pk_led.cpp       WS2812 状态灯
```

---

## 3. 硬件与接线

### 3.1 板子

| 板子 | 型号/说明 |
|---|---|
| 主板 | ESP32-S3-N16R8（16MB flash / 8MB OPI PSRAM），原生 USB + CH343 UART |
| 副板 | ESP32-S3（第二块 ESP32），GPIO41/42 做 UART1，GPIO48 WS2812，GPIO0 BOOT |
| 连接 | 主板 GPIO16/17 ↔ 副板 GPIO41/42 + GND |

### 3.2 两线 UART

```
主板 GPIO16 (TX) ──→ 副板 GPIO42 (RX)     推荐交叉
主板 GPIO17 (RX) ←── 副板 GPIO41 (TX)
GND ─────────────── GND
```

- 固件也支持**直连**（`16→41`、`17→42`）：主板每 6 秒自动交换 RX/TX，直到收到 proof。
- 两块板可以各自 USB 供电；GND 必须共地。
- 副板不接 USB 也能工作（只靠主板的 3V3/GND 时，注意共地）。

### 3.3 USB 口

- 主板原生 USB：`VID 0x303A / PID 0x82F1`，复合设备：
  - HID：FIDO/CTAPHID，供浏览器/Windows 使用
  - CDC：管理器/OTA 管理口
- 主板 CH343：UART0 调试/烧录，独立串口。
- 副板 CH343：UART0 日志/烧录。
- 副板原生 USB：USB-Serial-JTAG（烧录/日志用）；副板固件不初始化 USB CDC。

> **端口号会变**，不要写死。用 MAC/序列号识别：
> - 主板 MAC：`98:c3:77:ad:15:f8`（原生 USB 序列 `PKF815AD77C398`）
> - 副板 MAC：`ec:da:3b:42:de:40`
> - 主板 CH343：`5CF9109571`
> - 副板 CH343：`5C93064480`

### 3.4 外壳（可选）

`enclosure/` 里有 MAIN/KEY 两套 STL：

- 外形 54.1 × 81.6 mm，总高约 26.3 mm
- USB-C 12×6 mm，DC 头 Ø8 + 5.6 mm 钥匙槽
- 4 个卡扣 + 前缘撬槽，免螺丝
- 板轨间距 25.6 mm

注意：卡扣是建模验证，**没有实际打印试装**；第一次先打一个盖试松紧。

---

## 4. 功能与协议

### 4.1 主固件

- CTAP 2.1：`getInfo`、`makeCredential`、`getAssertion`、`getNextAssertion`、`clientPIN`（协议 1&2）、`reset`、`credentialManagement`、`selection`、`largeBlobs`、`authenticatorConfig`
- CTAP1/U2F：register / authenticate / version
- 常驻凭据（resident keys），最多 100 条
- USB HID（CTAPHID）+ USB CDC（管理/OTA）
- BLE FIDO GATT `0xFFFD`，`CMD/HLEN/LLEN` + `SEQ` 分帧，支持 CBOR `0x90` / U2F MSG `0x83`
- OTA：通过 CDC 写 firmware.bin
- 双 3MB OTA 槽 + ~9.9MB LittleFS

### 4.2 副板

- 保存 32 字节 **link key**
- 保存 32 字节 **KEK**
- 命令行协议：`K`（配对）、`C`（nonce 挑战）、`P`（HMAC 证明）、`E`（加密 KEK）、`R`（重置配对）、`W`（彻底清除）、`S`/`PING`
- LED：红灯 = 无 key/等待配对；绿灯 = 有 key

### 4.3 管理器

- 概览、Passkeys（列表/复制/删除）、密码库、安全、WebAuthn 演示、关于
- 密码库：scrypt(N=2^15) + AES-256-GCM，主密码不落盘
- TOTP/HOTP、密码生成器
- 导入：Chrome/Edge/Bitwarden/1Password/KeePass/LastPass CSV；导出 CSV/JSON
- 安全页：PIN、触摸确认、OTA、副板更换/重置/清除、恢复出厂
- 只监听 `127.0.0.1`

### 4.4 Android

- BLE 扫描/连接 `TechXPasskey`
- 设备信息、PIN 设置/修改、凭据列表/删除、日志
- CTAP2 over BLE；不替代系统 WebAuthn 选择器

---

## 5. 构建与烧录

### 5.1 主固件

```bash
cd esp32-passkey
pio run -d firmware
pio run -d firmware -t upload --upload-port COM8
```

- `COM8` 是示例，必须换成**主板 CH343/UART 口**。主板原生 USB 是 Passkey 口，不是烧录口。
- 只接原生 USB 时：按住 **BOOT** → 点 **RST** → 松开，再用 `scripts/flash_passkey.ps1` 或 esptool 烧录。
- `pio run -t upload` 写 app0；如果设备 otadata 仍指向 app1，可能启动旧固件。处理：
  - 用管理器 OTA（会正确切换分区），或
  - 擦除 otadata（`0xE000`，0x2000），或
  - 擦除旧 app1。

### 5.2 副板

```bash
pio run -d firmware-key
pio run -d firmware-key -t upload --upload-port COM12
```

- 副板 CH343 或原生 USB-Serial-JTAG 都可以烧录。
- 刷副板**不会清 NVS**：link key/KEK 保留。
- 烧完建议 hard reset 让 app 启动（`esptool --port COMx chip_id`）。

### 5.3 管理器

```bash
cd esp32-passkey/manager
python server.py
# http://localhost:8787
```

依赖：Python 3.11+、pyserial、python-fido2；仓库不包含 `.venv`。

### 5.4 Android

```bash
cd esp32-passkey/android
./gradlew assembleDebug
# app/build/outputs/apk/debug/app-debug.apk
```

需要 JDK 17+、Android SDK；`local.properties` 指本机 SDK。

### 5.5 外壳

直接打印 `enclosure/*.stl`；或改 `techxpasskey_enclosures.blend` 后重新导出。
生成脚本 `enclosure/build_enclosures.py` 通过 Blender MCP（`mcp-for-blender` + Blender 5.2 GUI）执行。

---

## 6. 首次配对与日常使用

### 6.1 首次配对

1. 主副板接线、上电。
2. 主板没有副板 key 时，每 3 秒发 `K <link key>`。
3. 副板是空白时保存 link key，回 `OK K`。
4. 之后每 700ms 一次 `C <nonce>` → `P <HMAC>`。
5. 主板日志：`second board stored the link key` → `second board OK`，LED 转蓝。
6. 主板请求 `E <nonce>`，副板回 `E <AES-GCM(link key, KEK)>`；主板解出 KEK，解 DEK，解密凭据到 RAM。

**配对是一次性的**：成功 proof 后写 `paired=true`，之后不再静默发 `K`。换副板必须走“更换副板”流程。

### 6.2 日常

- 浏览器选择 **TechXPasskey**，输入电脑上的 PIN，按 BOOT（当前 `upMode=0` 按键模式），副板必须在场。
- 管理器 Passkeys 页可看/删除；删除需要 PIN。
- 副板断开 2.5 秒：红灯、`il=false`、RAM 中 DEK/凭据清零；管理器列表 403。
- 副板恢复：重新 proof + KEK + 解密；实测 3 条凭据恢复。

---

## 7. 副板管理（管理页 → 安全 → 第二块板）

三个按钮，**都需要设备 PIN**：

| 按钮 | 做什么 | Passkey |
|---|---|---|
| **更换副板** | 旧副板先验证 → 主板生成 pending link key → 新副板保存并生成自己的 KEK → 主板用新 KEK **重新包裹 DEK** → 原子切换 | **保留** |
| **重置配对** | 清两块板的 link key，**保留副板 KEK** 和 DEK | **保留**，自动重新配对 |
| **彻底清除** | 清副板 link key + KEK，同时恢复出厂主板（删 DEK/凭据/PIN） | **删除**，不可恢复 |

### 7.1 物理清除

副板上电时按住 **BOOT** 3 秒：

- 清除 link key **和 KEK**
- 主板上的旧 DEK 将无法再解开（除非之前走过“更换副板”）
- 所以优先用管理器的“重置配对”或“更换副板”，最后才用 BOOT 物理清除

### 7.2 重要警告

- **只清副板 KEK、不清主板 = 数据永久不可解**。这正是“彻底清除”要同时恢复出厂的原因。
- 副板 KEK 丢失后，主板上的 PKC2 凭证库没有恢复手段（除非 RAM 里还有 DEK 且能赶在断电前重新包装，实际不可依赖）。

---

## 8. 安全架构

### 8.1 存储

| 数据 | 位置 | 保护 |
|---|---|---|
| Passkey 私钥/记录 | LittleFS `/creds.bin` | **PKC2**：AES-256-GCM，密钥 DEK |
| largeBlob | LittleFS `/blob.bin` | **PKB1**：同一个 DEK |
| DEK | 主板 NVS `dek` | `AES-GCM(KEK, DEK)`，KEK 不在主板 |
| KEK | **副板 NVS** | 仅副板；上电 proof 后加密发给主板 |
| PIN hash | 主板 NVS `cfg` | `AES-GCM(masterKey, SHA256(pin)[:16])` |
| masterKey | 主板 NVS `mk` | 只用于 PIN hash/兼容旧 key handle |
| link key | 主板 NVS `pk-il` + 副板 NVS | 双向挑战，用于 UART 认证和 KEK 加密 |

### 8.2 启动链

```text
主板 → 副板：C <nonce>
副板 → 主板：P <HMAC(link_key, nonce)>
主板 → 副板：E <nonce>
副板 → 主板：E <AES-GCM(link_key, KEK)>
主板：解 KEK → 解 NVS 里的 DEK → 解 /creds.bin 到 RAM
```

- 主板上电时**不加载**凭据；没有副板就是 `creds=0`。
- proof 超时 2.5 秒：清 RAM 中 DEK/KEK/凭据；重新接上再解密。

### 8.3 互锁门禁

以下操作要求 `interlock::ready()`（副板在场）：

- `makeCredential` / `getAssertion` / U2F
- `credentialManagement`（枚举/删除）
- 管理命令 0x40/0x41/0x42（列表/删除/全删）
- `largeBlobs`、`authenticatorConfig`
- OTA、BOOTLOADER
- 设置/修改 PIN
- 切换自动确认

不要求互锁的：`getInfo`、`clientPIN` 的读重试/getToken（签名本身仍被 gate）、`reset`（恢复出厂需要触摸，允许无副板）。

### 8.4 攻击面与边界

**只拿主板 + flash dump：**
- 得到 PKC2 密文、`AES-GCM(KEK, DEK)`、link key、master、PIN hash
- 没有 KEK，解不开 DEK/私钥
- 可以离线爆破 PIN hash（master 在 flash），但拿不到 Passkey

**只拿副板：**
- 得到 KEK + link key
- 没有主板 DEK/密文，不能单独使用

**两块都拿到：**
- 可以完整解密；这是双因子物理持有的边界，无法只靠软件防

**仍然存在的风险：**
- **Secure Boot / Flash Encryption 未启用**：能刷主板的人可以刷恶意固件，等你插上副板时请求 KEK。必须靠 Secure Boot 堵住。
- JTAG/ROM 下载未关闭；解锁期间 RAM 有明文。
- 关闭副板后 RAM 会清，但清之前有 2.5 秒窗口。
- non-resident（U2F/rk=false）旧 key handle 保留 masterKey 兼容解密路径；新 handle 用 DEK。
- largeBlob 已加密但还没有真实数据测试。
- 管理器密码库导出是明文，需自行保管。
- `il`/`ilDbg`/`ilLast` 调试字段在 INFO 里可见（不含密钥）。

---

## 9. 当前设备状态（交接时）

- 主板：MAC `98:c3:77:ad:15:f8`，原生 USB 序列 `PKF815AD77C398`
- 副板：MAC `ec:da:3b:42:de:40`
- 主固件 `0.1.0`，凭据库 `enc=true`
- 3 条常驻 Passkey，PIN 已设置（**PIN 线下交接，不在仓库**）
- `upMode=0`（按键确认）
- `minPin=6`
- `il=true`、`ilPaired=true`、`ilReplace=false`、`stErr=ok`
- 副板 link key/KEK 与主板匹配
- 副板固件为 UART0 日志版本（`ARDUINO_USB_MODE=0`）

> 端口号会变；以 MAC/序列号为准。当前曾观察：主板原生 COM10、主板 CH343 COM8、副板 CH343 COM11、副板原生 USB-JTAG COM12。

---

## 10. 测试与验证

### 10.1 快速检查

```bash
cd esp32-passkey
python scripts/pkdev.py info
```

预期关键字段：

```json
{
  "il": true,
  "ilState": "ok",
  "ilPaired": true,
  "ilReplace": false,
  "enc": true,
  "creds": 3,
  "minPin": 6,
  "upMode": 0,
  "stErr": "ok"
}
```

**注意**：管理器运行时会占用主板 CDC（COM10）。先关闭管理器，再跑 `pkdev.py`。

### 10.2 已有测试

```bash
python tests/ctap_test.py        # CTAP2 + PIN + 注册/断言
python tests/protocols_test.py   # U2F + credMgmt + largeBlob
python tests/pin_persist.py      # 重启后 PIN/凭据保留
python tests/ble_ctap_test.py    # BLE（需 -DPK_BLE_ALIAS=1 构建）
```

部分测试会改 PIN/凭据或恢复出厂，先看脚本再跑。

### 10.3 已验证的实测

- Chrome/Windows WebAuthn 注册 + 登录
- CTAP2 makeCredential/getAssertion 签名校验
- U2F 注册/认证/计数器
- clientPIN 协议 1&2
- credMgmt 枚举/删除
- largeBlobs 写入/读回
- OTA + 重启 + 数据保留
- BLE CTAP（getInfo/PIN/注册/断言）
- 副板 KEK 流程：`C → E → KEK → DEK`；3 条凭据解密
- 副板断开 2.5s：`creds=0`、`stErr=locked`、列表 403；恢复后 3 条
- 更换副板：旧板验证、pending key、DEK rewrap（代码路径；完整新板未实物演练）
- 重置配对：代码/UI 已完成；**成功路径需要真实 PIN，交接时未执行**（错误 PIN 返回 401）
- 彻底清除：代码/UI 已完成；**未执行**（会删 Passkey）

---

## 11. 已知限制与后续 TODO

### 必须做（安全）

1. **Secure Boot v2 + Flash Encryption + NVS 加密 + 关闭 JTAG/下载模式**
   - 当前 Arduino/PlatformIO bootloader 不是 Secure Boot 版本
   - 需要 ESP-IDF 重新构建签名 bootloader
   - eFuse 不可逆；需要备份、签名密钥离线保存、签名 OTA
   - 完成后 OTA 必须发签名镜像
2. 评估 **anti-rollback**（会阻止 OTA 降级，可能影响功能）
3. 副板也做 Secure Boot/Flash Encryption（保护 KEK）

### 功能/工程

- Android：改为使用 `minPinLength` 动态提示（已改源码），重新出 APK
- 外壳：实际打印试装，调卡扣 0.1–0.2 mm
- 管理器：可选显示 `ilDbg`/`enc` 诊断
- CI：固件构建、STL 校验、Android APK
- 文档：把 `docs/TechXPassKey.md` 与本文档保持同步

### 代码注意

- `peerReset` 在管理命令里阻塞最多 ~2.5 秒；此时 CTAP 不服务，可接受但需知道。
- `store::poll_security` 在主循环执行；不在中断里清 RAM。
- `encrypted_at_rest()` 在文件不存在时返回 true（空库也显示“已加密”）。
- 旧非驻留 key handle 走 masterKey 兼容解密；要彻底移除需迁移/重新注册。

---

## 12. 交接清单

- [ ] 主板 ×1、副板 ×1、USB 线（数据）×2、杜邦线 3 根（16/17/GND）
- [ ] 副板 CH343 或原生 USB 烧录线
- [ ] 设备 PIN（**线下告知，不写入仓库**）
- [ ] 管理器密码库主密码（**线下告知**）
- [ ] GitHub 仓库权限
- [ ] 确认没有 Secure Boot 签名密钥需要交接（当前未生成）
- [ ] 3D 打印外壳 STL / Blender 文件
- [ ] 备份当前 `.blend`、固件源码、管理器 `data/vault.enc`（如已创建，勿提交）

---

## 13. 常见故障

| 现象 | 原因/处理 |
|---|---|
| 管理器列表 403 | 副板不在场 / proof 超时 / `ilReplace` 模式；接好副板等 1–3 秒 |
| `creds=0`、`stErr=locked` | 副板断开后 RAM 已清；接回副板会自动 KEK/DEK 解密 |
| `il=false` 但副板绿灯 | UART 接反/线松；等 6 秒自动换 RX/TX；检查 GND |
| 副板无日志 | 副板固件 `ARDUINO_USB_MODE=0` 时日志在 UART0 CH343，不在原生 USB |
| `pkdev.py` 打不开 COM10 | 管理器占用了 CDC；先关管理器 |
| 烧录成功但启动旧固件 | app0/app1/otadata；用管理器 OTA 或擦 otadata |
| OTA begin rejected | 新固件 OTA 被副板互锁 gate；等 `il=true` 再 OTA |
| Windows BLE 看不到 0xFFFD | Windows FIDO 栈独占；浏览器正常；自动化测试用 `PK_BLE_ALIAS=1` 映射 0xFFF0 |
| 忘记 PIN | CTAP reset（需要触摸）或重新烧录；会清凭据；副板 KEK 也会失配 |
| 换副板后旧板还能用 | 旧板 link key 已被替换；旧板数据建议用“重置配对/彻底清除”或 BOOT 清除 |

---

## 14. 关键提交

| 提交 | 内容 |
|---|---|
| `c0106cf` | 副板 KEK + DEK 包裹、PKC2 凭据库 |
| `7fee623` | 断开副板清 RAM、PIN ≥6 |
| `029a414` | largeBlob 加密、OTA/PIN/UP-mode 互锁 |
| `d8e2c3d` | 管理器副板重置/彻底清除（PIN 验证） |
| `9756f7b` / `c415c68` | 外壳 STL / 卡扣盖 |

完整历史：<https://github.com/Ender-Drate/esp32-passkey/commits/main>

---

## 15. 一句话交接

> 主板上只有密文和被 KEK 包裹的 DEK；KEK 在副板。开机先 UART 挑战，副板才交出 KEK；没有副板，Passkey 既不能签名也解不开。换板用“更换副板”，重配用“重置配对”，报废才“彻底清除”。Secure Boot/Flash Encryption 尚未启用，这是下一步。
