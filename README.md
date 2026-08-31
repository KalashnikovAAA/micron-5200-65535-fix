# Micron 5200 系列 65535 小时固件 Bug:故障实录、诊断方法与修复工具包

**[中文](#中文) | [English](#english)**

---

# 中文

> Micron 5200 (5100/5300/5400 同族) SATA SSD 在累计通电 **65,535 小时(约 7.5 年)**后触发固件缺陷,
> 盘内调试日志风暴打满主控 CPU,读写吞吐坍塌到几 MB/s——SMART 全绿、无任何错误计数,极易被误诊为
> 阵列卡 / 背板 / 文件系统问题。
>
> **修复:刷入 D1MU040(1.92TB ECO/PRO/MAX)/ D1MU440 / D1MU540 / D1MU840。溢出已发生后再刷同样能恢复。**

本仓库包含:完整故障案例复盘、可复用的诊断方法(尤其是"如何不被 RAID 卡缓存欺骗")、
固件包(含 MD5/SHA256 校验)、Windows/Linux 刷写步骤、HPE Smart Array 环境的特殊注意事项。

**如果你手里有 2018-2019 年上电、7×24 运行的 Micron 5xxx 系列 SSD——现在就去查
`smartctl` 的 Power_On_Hours,预计都已越过或即将越过 65,535 小时。**

## 1. 根因(官方确认)

- 盘固件用 **16 位计数器**记录 workload log 条目,在 65,535 小时处溢出;
- 溢出后固件开始**持续产生调试日志**,耗尽盘内主控 CPU 资源,I/O 处理能力骤降;
- 表现:吞吐跌至 **1 MB/s 量级**、盘响应极慢;但**盘仍可访问、无数据损坏、SMART 完全正常**;
- Dell KB 000464492 与 Micron 支持工单均确认:**受影响固件 D1MU004 / D1MU020 / D1MU030,修复版 D1MU040**。

> Micron 支持工单原文(节选):
> "There is a known firmware issue in firmware versions D1MU004, D1MU020, and D1MU030 that is corrected
> in a NON-PUBLIC released version of firmware D1MU040 ... At the point the counter exceeds 65535 hours,
> the drive becomes near unresponsive with a data throughput of 1 MB/S or less."

修复版为 Micron 原厂签名固件(2026-01-28 编译,Clapton 分支),经 NDA 渠道流出,官网 EOL 页不提供下载。

## 2. 真实案例时间线(脱敏)

环境:8 × Micron 5200 1.92TB SATA SSD,HPE Smart Array P420i(1GB FBWC,10/90 读写配比),
RAID50 单卷 ~10.5TB,ESXi 6.7,7×24 运行。

| 时间 | 事件 |
|---|---|
| 2019-03 | 8 盘装机上线(其中 1 盘 2021 年更换,换盘通电时长比原装少 ~22,000h) |
| 2026-08-25 凌晨 | 原装 7 盘相继越过 65,535h(同批盘越线时刻只差几小时) |
| 08-26 11:25 | `vobd.log` 出现**首个** `vmfs.heartbeat.timedout`(驱动级 IO 延迟 24.8s)——比肉眼可见故障早 28 小时 |
| 08-26 23:56 | `vmkernel.log` 开始出现 H:0x8 命令超时,逐级加重 |
| 08-27 15:27 | 风扇狂转、全部虚拟机 I/O 爬行、ESXi 将卷 quiesce;**冷断电重启后"恢复正常"(实为缓存假象,见 §3)** |
| 08-28 | 阵列卡固件 3.54→8.32(修了已知活锁,但与根因无关);2GB 写测试暴露真实写入仅 7~12MB/s |
| 08-29 | 拔盘对照实验一度"确认单盘故障"——后被证明同样是缓存假象 |
| 08-30~31 | USB 盒外挂读 SMART:换装盘 43,560h(健康),原装盘 **65,681 / 65,687h(已越线)** → 根因实锤 |
| 08-31 | 8 盘全部刷 D1MU040 → 写入 285MB/s、冷读 800MB/s,重建后满血 |

## 3. 诊断方法与血泪教训(可复用)

### 3.1 不要被 RAID 卡缓存欺骗(本案例最大的坑)

P420i 带 1GB FBWC、90% 写缓存:任何 **≤ 900MB 的写入测试都会被缓存整口吞掉**,dd 返回的是缓存速度,
与盘无关。同样,读取测试若读的是**刚写完的文件**,命中主机页缓存,也是假象。

**正确姿势:**
- 写测试总量 **≥ 2× 控制器写缓存容量**(本例 2GB 分 8 块,看的是缓存灌满后的排水速率);
- 读测试用**冷数据**(阵列上长期存在的旧文件);
- 连续多块测试,观察是否"第一块快、后面全卡"——那就是缓存灌满的特征。

### 3.2 日志信号的出现顺序

1. **最早**:`vobd.log` 的 `esx.problem.vmfs.heartbeat.timedout` + 恢复记录里的 `drv XX.XX`
   (驱动级单次 IO 延迟秒数)——比下面所有信号早约一天;
2. 中期:`vmkernel.log` `ScsiDeviceIO ... failed H:0x8`(HBA 级命令超时)成批爆发;
3. 晚期:ESXi `Restricting cmd ... to quiesced dev`(卷被挂起)、上层业务瘫痪。

硬件侧(IML/SEL/AHS/ssacli 状态)在此缺陷中**全程零告警**——"三层硬件监控全绿 + 吞吐坍塌"
本身就是盘固件级故障的指纹。

### 3.3 逐项排除清单

- 阵列卡固件活锁:升级固件/冷重启只能清除队列积压,越线的盘依然慢 → 不能解释"重启后 2GB 测试仍卡";
- 控制器后台表面扫描:`ssacli ctrl slot=0 modify surfacescanmode=disable` 后复测,卡滞依旧 → 排除;
- 温度:持续写入中逐盘采样温度(卡壳内自救的盘会发热),无热点 → 排除单盘过热;
- 残留进程:测试前 `ps | grep -E 'dd if|esxtop'`,卡死的 esxtop/dd 会持锁导致误诊;
- **RAID50 特性:一个慢成员拖死整个卷**(条带写等最慢的盘),所以"全卷慢"≠"所有盘坏"。

### 3.4 定位嫌疑盘

ssacli 不透出 SMART 的 Power_On_Hours,RAID 模式下盘也无法在线刷固件 → **物理拔盘 + USB 盒外挂**:

```bash
# HPE Smart Array 点亮盘位定位灯(不影响 IO)
ssacli ctrl slot=0 pd 1I:2:2 modify led=on    # off 关闭

# 外挂后读关键数据
smartctl -a /dev/sdX | grep -iE 'Power_On_Hours|Firmware'
```

**判读:POH ≥ 65,535 → 已越线;反常地小(如几十小时)→ 计数器已回绕,同样算越线;
中间值(如 4 万多)→ 该盘健康,是别的盘的问题。**

### 3.5 同批盘的越线规律

同一天上电的盘,POH 只相差几十小时,**越线时刻集中在几天窗口内**——表现为"某天突然集体发病、
逐日加重"。冷重启、换阵列卡固件都无效(盘内计数器不随外部重置归零)。

## 4. 修复操作

### 4.1 固件版本对照

| 固件 | 适用型号 | 容量 (GB) |
|---|---|---|
| **D1MU040** | 5200 ECO / **PRO** / MAX | 240 / 480 / 960 / **1920** |
| D1MU440 | 5200 ECO | 3840 |
| D1MU540 | 5200 PRO | 3840 |
| D1MU840 | 5200 ECO | 7680 |

D1MU030 及更早版本**不含修复**,升级到 D1MU030 无效。D1MU004 可直接升 D1MU040。

### 4.2 校验(刷前必核,任一不符不要刷)

| 文件 | MD5 | SHA256 |
|---|---|---|
| **D1MU040/1.bin** | `e8e51fcac5d602685233a4c53b355bf3` | `9b8c28bf3550a39b16ecb5923881d0aed317c22f80403f859ba7979bf10f6b2a` |
| D1MU440/1.bin | `b64e80fd043e939006c622b0fdaf094d` | `a8e0d5c8ade0ade4f6c5f72ec4f1e7df7c5b40543068f35714c0c55bb4c7199a` |
| D1MU540/1.bin | `7af0a30914407df05ef857bd10abad05` | `7002353d668468764f08480d46a3f2667932595054b3903b2e8385fe33b53036` |
| D1MU840/1.bin | `d7c3cc2a711635887c7a8c181ad5e8f4` | `250827efd94e40419cd45b61758ff3ea242352b2d8020a264ff1344eee24c530` |
| firmware.properties | `760a6d1d5f0b425590266200b80ac1fa` | `16745d6ba62f554dd72ad6a43f34edbe3f2b4815dcd29e6962f7ed0a8449a63a` |

哈希来自 STH 论坛多名用户独立计算交叉验证(三方一致)。msecli 刷写时还会做盘型号/固件签名校验。

### 4.3 刷写步骤(Windows)

1. 安装 Micron / Crucial Storage Executive(自带 `msecli.exe`);
2. 盘接 SATA 直通或 USB 盒(有成功实证,直连更稳);**RAID 卡后面的盘刷不到,必须外挂**;
3. 管理员 CMD:

```cmd
cd "C:\Program Files\Crucial\Crucial Storage Executive"
msecli -F                                                   &:: 确认识别、看清当前固件
msecli -F -U C:\path\D1MU040\1.bin -n Drive1                &:: 确认 Y
```

4. 成功标志:`CMD_STATUS : Success / STATUS_CODE : 0`;
5. **完整断电重启**(关机拔电源 10 秒;USB 盒独立供电可断盒子的电)激活固件;
6. `msecli -F` 确认版本变为 D1MU040。

Linux 对应:`sudo msecli -F -U /path/to/1.bin -n /dev/sdX`,同样需要冷加电激活。

### 4.4 溢出已经发生,刷了还能恢复吗?

**能。** 根因是日志风暴耗 CPU,无永久损伤:多处实测刷 D1MU040 后吞吐立刻回到 300~500MB/s,
与本轮案例(8 盘刷后 285MB/s 写 / 800MB/s 读)一致。刷前 SMART 若显示 POH 回绕成小数字属正常。

## 5. HPE Smart Array 环境注意事项

- **在线刷不了**:盘在 RAID 卡虚拟盘后面,msecli 看不到物理盘 → 拔盘外挂刷;
- **批量刷的建议姿势**:关主机 → 逐块拔出外挂刷写 → **原槽位插回** → 开机。
  固件刷写不动用户数据,原槽位插回不需要重建,不存在"降级期间再坏一块"的风险窗口;
- **重建行为**:热插回曾经离线的成员,逻辑卷先显示 `Ready for Rebuild`(过渡态),随后自动进入
  `Recovering, N%`,无需干预;
- **绝对不要用** `ssacli ... modify reenable forced`:如果盘离线期间阵列发生过任何写入
  (哪怕只是测试和日志),该命令会把过期数据直通回阵列。正确路径永远是"作为备件重建";
- 拔盘前点亮定位灯、拍照记录槽位;同型号盘多时给托架贴标签。

## 6. 固件获取渠道(本仓库不分发固件二进制)

修复固件为 Micron 非公开发布版本,本仓库只提供哈希与渠道信息,请自行下载并按 §4.2 校验:

| 渠道 | 说明 |
|---|---|
| **Hetzner 救援系统**(原始源头) | 注册 Hetzner Cloud,**必须选德国机房**(美国机房只有旧 030),开最便宜的 Intel 云服务器 → 启用 rescue 救援模式 → SSH 进入后固件在 `/root/.oldroot/nfs/firmware_update/crucial/5200/`(同目录 `crucial/msecli` 有 Linux 版刷写工具);拉回本地后删机,全程约 1 欧分 |
| Dropbox 镜像 | <https://www.dropbox.com/scl/fi/se6n61ynror3yr46cuyyp/5200-d1mu-040-440-540-840.zip?rlkey=1mn4rfbdjvlw47c1bd4sw41e0> |
| MEGA 镜像 | <https://mega.nz/folder/iiASiZqR#Wn1N-Eui9NUaitQLFI_rOw> |
| Micron 官方工单(最慢但最正规) | <https://www.micron.com/secure-portal/technical-support-contact>,附上型号/序列号/`smartctl -x` 输出与故障描述,索取 D1MU040 |

镜像链接来自 STH 社区分享,可能失效;失效时用 Hetzner 渠道或工单。下载后**务必核对 §4.2 哈希**。

## 7. 免责声明

- 本仓库**只包含诊断文档、哈希与下载渠道信息,不含任何固件二进制**;
- 固件为 Micron 原厂签名文件,属非公开发布版本,由社区多渠道镜像传播;刷写自担风险,刷前务必核对哈希;
- 固件版权归 Micron Technology 所有,如 Micron 提出要求,本仓库将删除固件文件(诊断文档保留);
- 案例中的主机名/IP/序列号/虚拟机名等敏感信息已做脱敏处理。

## 8. 参考

- ServeTheHome 论坛主帖(固件流出、哈希、刷写实证): <https://forums.servethehome.com/index.php?threads/micron-5200-firmware-bug.55339/>
- Reddit r/DataHoarder 首例报告: <https://www.reddit.com/r/DataHoarder/comments/1sly734/>
- Dell KB 000464492(根因官方说明): <https://www.dell.com/support/kbdoc/en-my/000464492/>
- Micron Storage Executive 下载: <https://www.micron.com/sales-support/downloads/software-drivers/storage-executive-software>

---

# English

> Micron 5200-family SATA SSDs (5100/5300/5400 share the same lineage) hit a firmware defect at
> **65,535 cumulative power-on hours (~7.5 years)**: a debug-logging storm inside the drive starves the
> drive's own controller, and throughput collapses to a few MB/s — while SMART stays 100% clean with zero
> error counts. It is very easily misdiagnosed as a RAID controller / backplane / filesystem problem.
>
> **The fix: flash D1MU040 (for 1.92TB ECO/PRO/MAX) / D1MU440 / D1MU540 / D1MU840. Flashing after the
> rollover has already happened fully restores performance.**

This repository contains: a complete real-world incident post-mortem, reusable diagnosis methods
(especially "how not to be fooled by the RAID controller's cache"), the firmware package with
MD5/SHA256 verification, Windows/Linux flashing steps, and Smart Array-specific caveats.

**If you operate Micron 5xxx SSDs that were first powered on in 2018-2019 and run 24/7 — check
`smartctl` Power_On_Hours right now. They have most likely already crossed (or are about to cross)
65,535 hours.**

## 1. Root Cause (officially confirmed)

- The drive firmware tracks workload-log entries with a **16-bit counter** that overflows at 65,535 hours;
- After the overflow the firmware starts **generating debug log messages continuously**, which consumes
  the drive controller's CPU and starves I/O processing;
- Result: throughput drops to the **~1 MB/s range** with extremely slow responsiveness — but the drive
  **remains accessible, data stays intact, and SMART reports nothing**;
- Dell KB 000464492 and a Micron support ticket both confirm: **affected firmware D1MU004 / D1MU020 /
  D1MU030; fixed in D1MU040.**

> From a Micron support ticket:
> "There is a known firmware issue in firmware versions D1MU004, D1MU020, and D1MU030 that is corrected
> in a NON-PUBLIC released version of firmware D1MU040 ... At the point the counter exceeds 65535 hours,
> the drive becomes near unresponsive with a data throughput of 1 MB/S or less."

The fixed firmware is a Micron-signed build (compiled 2026-01-28, "Clapton" branch) that leaked via an
NDA channel; Micron's EOL product pages do not offer it for download.

## 2. Real Incident Timeline (sanitized)

Environment: 8 × Micron 5200 1.92TB SATA SSDs behind an HPE Smart Array P420i (1GB FBWC, 10/90
read/write ratio), a single ~10.5TB RAID50 volume, ESXi 6.7, 24/7 operation.

| Time | Event |
|---|---|
| 2019-03 | All 8 drives installed (one was replaced in 2021; the replacement has ~22,000 fewer power-on hours) |
| 2026-08-25 early AM | The 7 original drives cross 65,535h one after another (same-batch drives cross within hours of each other) |
| 08-26 11:25 | **First** `vmfs.heartbeat.timedout` in `vobd.log` (driver-level IO latency 24.8s) — 28 hours before the visible outage |
| 08-26 23:56 | H:0x8 command timeouts start appearing in `vmkernel.log`, escalating |
| 08-27 15:27 | Fans ramp to full speed, all VM I/O crawls, ESXi quiesces the volume; **a cold power cycle "fixes" it (actually a cache illusion, see §3)** |
| 08-28 | Controller firmware upgraded 3.54→8.32 (fixes a known livelock, unrelated to root cause); a 2GB write test exposes the true write rate of 7–12MB/s |
| 08-29 | A pull-one-drive A/B test briefly "confirms" a single bad drive — later shown to be another cache illusion |
| 08-30–31 | External SMART via USB enclosure: replacement drive 43,560h (healthy), originals **65,681 / 65,687h (crossed!)** — root cause confirmed |
| 08-31 | All 8 drives flashed to D1MU040 → 285MB/s writes, 800MB/s cold reads; array rebuilt and fully restored |

## 3. Diagnosis Methods and Hard-Won Lessons (reusable)

### 3.1 Do not be fooled by the RAID controller cache (the biggest trap in this case)

The P420i has 1GB FBWC with 90% allocated to writes: **any write test ≤ ~900MB is swallowed whole by
the cache.** The number dd reports is cache speed and says nothing about the drives. Likewise, a read
test against a **just-written file** hits the host page cache — also an illusion.

**Correct methodology:**
- Total write volume **≥ 2× the controller's write cache** (here: 2GB in 8 blocks; what matters is the
  drain rate once the cache is full);
- Read tests must use **cold data** (old files that have been sitting on the array);
- Run several consecutive blocks and watch for "first block fast, everything after crawls" — that is the
  signature of a full cache draining slowly.

### 3.2 The order in which log signals appear

1. **Earliest:** `esx.problem.vmfs.heartbeat.timedout` in `vobd.log`, with `drv XX.XX` in the recovery
   record (driver-level per-IO latency in seconds) — roughly a day before anything else;
2. Middle stage: batch `ScsiDeviceIO ... failed H:0x8` (HBA-level command timeouts) in `vmkernel.log`;
3. Late stage: ESXi `Restricting cmd ... to quiesced dev` (volume quiesced), workloads dead.

Hardware-side monitoring (iLO IML/SEL/AHS, ssacli status) stays **completely silent** throughout this
defect — "all hardware monitoring green + collapsed throughput" is itself the fingerprint of a
drive-firmware-level fault.

### 3.3 Elimination checklist

- Controller firmware livelock: upgrading/rebooting only clears queued backlog; rolled-over drives stay
  slow → cannot explain "still slow in a 2GB test after reboot";
- Controller surface scan: retest after `ssacli ctrl slot=0 modify surfacescanmode=disable`; if stalls
  persist → ruled out;
- Temperature: sample per-drive temperatures during sustained writes (a drive stuck in internal
  self-recovery gets hot); no hot spot → no single overheating drive;
- Leftover processes: before testing, `ps | grep -E 'dd if|esxtop'` — stuck esxtop/dd instances hold
  locks and poison measurements;
- **RAID50 property: one slow member drags down the whole volume** (stripe writes wait for the slowest
  drive), so "whole volume slow" ≠ "all drives bad".

### 3.4 Identifying the suspect drive

ssacli does not expose SMART Power_On_Hours, and drives cannot be flashed in place behind a RAID
controller → **pull the drive and attach it externally via USB enclosure**:

```bash
# Light up the bay locate LED on HPE Smart Array (does not affect IO)
ssacli ctrl slot=0 pd 1I:2:2 modify led=on    # use led=off to turn it off

# Once attached externally, read the key data
smartctl -a /dev/sdX | grep -iE 'Power_On_Hours|Firmware'
```

**Interpretation: POH ≥ 65,535 → crossed; suspiciously small (e.g. a few dozen hours) → counter has
wrapped, also crossed; a mid-range value (e.g. ~43k) → this drive is healthy, look elsewhere.**

### 3.5 How same-batch drives roll over

Drives first powered on the same day differ by only tens of hours of POH, so **their rollover moments
cluster within a window of a few days** — perceived as "suddenly all sick one day, worse the next".
Cold reboots and controller firmware changes do nothing (the in-drive counter never resets).

## 4. The Fix

### 4.1 Firmware version map

| Firmware | Models | Capacities (GB) |
|---|---|---|
| **D1MU040** | 5200 ECO / **PRO** / MAX | 240 / 480 / 960 / **1920** |
| D1MU440 | 5200 ECO | 3840 |
| D1MU540 | 5200 PRO | 3840 |
| D1MU840 | 5200 ECO | 7680 |

D1MU030 and earlier **do not contain the fix** — "upgrading" to D1MU030 is useless. D1MU004 upgrades
directly to D1MU040.

### 4.2 Checksums (verify before flashing; if anything mismatches, do not flash)

| File | MD5 | SHA256 |
|---|---|---|
| **D1MU040/1.bin** | `e8e51fcac5d602685233a4c53b355bf3` | `9b8c28bf3550a39b16ecb5923881d0aed317c22f80403f859ba7979bf10f6b2a` |
| D1MU440/1.bin | `b64e80fd043e939006c622b0fdaf094d` | `a8e0d5c8ade0ade4f6c5f72ec4f1e7df7c5b40543068f35714c0c55bb4c7199a` |
| D1MU540/1.bin | `7af0a30914407df05ef857bd10abad05` | `7002353d668468764f08480d46a3f2667932595054b3903b2e8385fe33b53036` |
| D1MU840/1.bin | `d7c3cc2a711635887c7a8c181ad5e8f4` | `250827efd94e40419cd45b61758ff3ea242352b2d8020a264ff1344eee24c530` |
| firmware.properties | `760a6d1d5f0b425590266200b80ac1fa` | `16745d6ba62f554dd72ad6a43f34edbe3f2b4815dcd29e6962f7ed0a8449a63a` |

Hashes were computed independently by multiple STH users and match (three-way agreement). msecli also
validates drive model and firmware signature during flashing.

### 4.3 Flashing (Windows)

1. Install Micron / Crucial Storage Executive (bundles `msecli.exe`);
2. Attach the drive via direct SATA or a USB enclosure (USB has real-world success stories; direct SATA
   is more reliable). **Drives behind a RAID controller cannot be flashed — external attachment required;**
3. Administrator CMD:

```cmd
cd "C:\Program Files\Crucial\Crucial Storage Executive"
msecli -F                                                   &:: confirm detection and current firmware
msecli -F -U C:\path\D1MU040\1.bin -n Drive1                &:: answer Y
```

4. Success looks like: `CMD_STATUS : Success / STATUS_CODE : 0`;
5. **Full power cycle** (shut down, unplug AC for 10 seconds; for a self-powered USB enclosure, cycling
   the enclosure is enough) to activate the firmware;
6. Verify with `msecli -F` that the version now reads D1MU040.

On Linux: `sudo msecli -F -U /path/to/1.bin -n /dev/sdX`; cold power-on applies the same way.

### 4.4 The rollover already happened — does flashing still help?

**Yes.** The root cause is a CPU-starving log storm with no permanent damage: multiple independent tests
show throughput back at 300–500MB/s immediately after flashing D1MU040, matching this case (8 drives
flashed → 285MB/s writes / 800MB/s reads). If SMART shows a wrapped (tiny) POH value after the fix,
that is normal.

## 5. Smart Array-Specific Notes

- **No in-place flashing:** physical drives sit behind the controller's logical volume, invisible to
  msecli → pull and flash externally;
- **Recommended batch procedure:** power off the host → pull drives one at a time and flash → **reinsert
  into the same bays** → power on. Firmware flashing does not touch user data; same-bay reinsertion needs
  no rebuild and opens no "second failure during degraded mode" window;
- **Rebuild behavior:** hot-reinserting a previously-failed member first shows `Ready for Rebuild`
  (transitional), then automatically enters `Recovering, N%` — no intervention needed;
- **Never** run `ssacli ... modify reenable forced`: if any writes hit the array while the drive was out
  (even just tests and logs), it force-feeds stale data back into the array. The correct path is always
  "rebuild as a spare";
- Light the locate LED before pulling; photograph bay positions; label carriers when running identical
  drives.

## 6. Where to Get the Firmware (this repository distributes no binaries)

The fixed firmware is not publicly released by Micron. This repository only provides checksums and
channel information — download it yourself and verify against §4.2:

| Channel | Notes |
|---|---|
| **Hetzner rescue system** (original source) | Register a Hetzner Cloud server — **German locations only** (US locations only carry the old 030 files), cheapest Intel instance → enable rescue mode → firmware lives at `/root/.oldroot/nfs/firmware_update/crucial/5200/` (`crucial/msecli` in the same tree has the Linux flashing tool); pull the files, delete the server — costs about 1 euro-cent |
| Dropbox mirror | <https://www.dropbox.com/scl/fi/se6n61ynror3yr46cuyyp/5200-d1mu-040-440-540-840.zip?rlkey=1mn4rfbdjvlw47c1bd4sw41e0> |
| MEGA mirror | <https://mega.nz/folder/iiASiZqR#Wn1N-Eui9NUaitQLFI_rOw> |
| Micron support ticket (slowest, most official) | <https://www.micron.com/secure-portal/technical-support-contact> — attach model/serials/`smartctl -x` output and request D1MU040 |

Mirror links come from the STH community and may go stale; fall back to Hetzner or a ticket. **Always
verify hashes from §4.2 after downloading.**

## 7. Disclaimer

- This repository **contains only diagnosis documentation, checksums, and channel information — no
  firmware binaries**;
- The firmware is a Micron-signed, unreleased build circulating through community mirrors; flash at your
  own risk and always verify hashes first;
- Firmware copyright belongs to Micron Technology; if Micron requests it, the firmware files will be
  removed from this repository (the diagnosis documentation stays);
- Hostnames/IPs/serial numbers/VM names from the original incident have been sanitized.

## 8. References

- ServeTheHome thread (firmware leak, hashes, flashing reports): <https://forums.servethehome.com/index.php?threads/micron-5200-firmware-bug.55339/>
- Reddit r/DataHoarder first report: <https://www.reddit.com/r/DataHoarder/comments/1sly734/>
- Dell KB 000464492 (official root cause): <https://www.dell.com/support/kbdoc/en-my/000464492/>
- Micron Storage Executive download: <https://www.micron.com/sales-support/downloads/software-drivers/storage-executive-software>
