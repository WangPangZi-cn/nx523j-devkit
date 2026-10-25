# 已知问题、设计取舍与调试地图

> 基于 v18（2026-10-25）。改动这些行为时请先读 BUILD_GUIDE.md 的对应章节。

## 1. 按设计移除的功能（不是 bug）

| 功能 | 移除位置 | 备注 |
|---|---|---|
| 电话/短信框架 | build/make handheld_system.mk 等 | 无 SIM 常驻进程已清 |
| 联系人存储 | build/make | ContactsProvider 已删 |
| 日历存储 | build/make | CalendarProvider 已删 |
| NFC/安全元件 | build/make | 硬件本无 NFC |
| GPS | 设备树/overlay | 纯 WiFi 机不需要 |
| 指纹 | 设备树 | goodix 相关 HAL 未编入 |
| Seedvault 备份 | vendor/lineage common_mobile.mk | 注意悬空 `\` 陷阱 |
| Traceur | build/make | 收 trace 用，精简 |

想恢复某功能：先在 patches 里找到对应的删除 hunk 反向应用，再确认它
没有隐藏依赖（如 PBAP→ContactsProvider）。

## 2. 已知行为说明

- **蓝牙只支持 V1.0 HAL**：`system/bt` 被改为 V1_0 优先；若换更新的
  vendor 库需要把该 hunk 回退并重新测超时。
- **PBAP/MAP 服务端关闭**：蓝牙文件传输（OPP）可用，通讯录共享不可用
  （设计如此，依赖已被删的 ContactsProvider）。
- **蓝牙 MAC 显示**：从 `persist.service.bdroid.bdaddr` 等来源读取，
  可能显示非常规地址；如需固化真实 MAC，把 `ro.bt.bdaddr_path` 指向
  一个持久化文件或在设备树属性里写死。
- **音频只走 extcodec 配置**：改音频路由先看
  `audio_platform_info_extcodec.xml`，HAL 不读普通 audio_platform_info.xml。

## 3. 调试地图（最快的入口）

### 音频
```bash
# DAPM 状态（功放 TFA9890）
cat /sys/kernel/debug/asoc/msm8976-tasha-snd-card/tfa98xx.8-0034/dapm/I2S1
cat /sys/kernel/debug/asoc/msm8976-tasha-snd-card/tfa98xx.8-0034/dapm/bias_level
# HAL 生效的配置
adb shell "cat /vendor/etc/audio_platform_info_extcodec.xml | grep -A3 backend"
# 日志
adb logcat -s AudioFlinger AudioPolicyManager audioserver
```
无声故障链：extcodec xml 缺失 → SPEAKER 后端名缺失 → QUIN_MI2S_RX
不开 → TFA9890 无偏置。

### 蓝牙
```bash
adb logcat -s bt_stack bluetooth bt_vendor
getprop | grep -E 'bluetooth|bt|bdaddr'
```
启动失败按 BUILD_GUIDE.md 4.4 的六层链路逐层排查。

### 内存基线
```bash
adb shell free -m          # v18 待机 free 约 2200MB+
adb shell dumpsys meminfo  # 按进程排序
```

### 网络检测（captive portal）
```bash
adb shell settings get global captive_portal_http_url
# v18 默认值已固化在 NetworkStack（不是 settings 数据库），重刷不丢
```

## 4. 性能基线（v18 实测）

- 待机空闲 RAM：约 2.2GB（free -m）
- 蓝牙开关：稳定，无崩溃
- 开机到桌面：与 v17 持平
- 抖音等大型应用：正常播放、有声
