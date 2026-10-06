# Hisense VIDAA 电视开机广告逆向研究

> 对两台海信 VIDAA 电视的开机广告机制做了完整逆向：从固件分区提取、广告链路分析到无 root/有 root 两种禁用方案。
> 全部操作在本人自有设备上进行；报告中的序列号 / MAC 地址均已脱敏。

## 研究对象

| 设备 | 方案 | 系统 | 固件 | 权限 |
|---|---|---|---|---|
| **Hisense LED55EC750US**（55"） | MStar MSD6A648（mooney） | Android 6.0 / VIDAA 4.1 | 01.02.41.01 | adb root（hsc 域） |
| **Hisense 43V1F-R**（43"） | MTK MT5862 / MTK9255 | Android 9 | 02.02.60.01 | adb shell（无 root） |

## 核心结论（TL;DR）

1. **广告不是固件内置的，是云端投放的**：广告总控 `com.hisense.service.ad` 从 `ad-download.hismarttv.com` 拉取策略（广告位 `ad_on_c`，每 24h 评估）与素材，落地到本机缓存后由原生 `bootanimation` 在每次开机/待机唤醒时播放，**播完自动删除缓存**（"一次性投放"机制，所以平时翻目录是空的）。
2. **冷开机与待机唤醒（STR）是两条素材**：`third_party_bootvideo.mp4`（冷开机）与 `third_party_fast_bootvideo.mp4`（STR 唤醒）。
3. **禁用广告 App ≠ 立刻没广告**：已下载到缓存的视频仍会被播放，必须一并清理缓存目录（详见报告 §7）。
4. **无 root 也能禁**：adb shell 自带组件管理权限，`pm disable-user --user 0` 即可禁用广告全家桶；两台电视实测有效。

## 快速禁用（亲测有效的两条路）

**A. 通用（两台都有效，可逆）**

```bash
adb connect <TV-IP>:5555
adb shell pm disable-user --user 0 com.hisense.service.ad      # 广告总控/下载
adb shell pm disable-user --user 0 com.hisense.business.ad     # 淘TV开屏
adb shell pm disable-user --user 0 com.hisense.service.hproxy  # 下载代理
# 恢复：pm enable <包名>
```

**B. MStar 老机型追加：hosts 断源**

```bash
adb shell "mount -o remount,rw /system && \
  printf '127.0.0.1 ad-download.hismarttv.com\n::1 ad-download.hismarttv.com\n' >> /system/etc/hosts && \
  sync && mount -o remount,ro /system"
```

**C. 别忘了清缓存**（否则已下载的广告照放）：
- MStar 机：`/data/local/misc/third_party_*`
- MTK 机（43V1F-R）：`/bootad/local/misc/third_party_*.mp4`

## 仓库结构

```
docs/
  开机广告分析报告.md   # 完整逆向报告（脱敏版）：分区表、开机流程、广告链路、
                        # 提权审计记录、两台机器的禁用方案与踩坑记录
```

## 声明

- 本研究仅针对本人自有设备，用于个人定制与安全研究，不构成对任何厂商服务的攻击建议。
- Hisense、VIDAA 及文中提及的第三方名称与商标归各自权利人所有。
- 固件镜像与广告视频含版权内容，本仓库不予分发；文中证据以脱敏截图/字符串/配置摘录形式呈现。
