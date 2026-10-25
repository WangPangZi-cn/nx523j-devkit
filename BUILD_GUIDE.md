# Nubia Z11 Max (NX523J) — LineageOS 18.1 构建手册

> 设备：努比亚 Z11 Max（NX523J）｜ SoC：MSM8976（Snapdragon 652）｜ 4GB RAM
> 目标系统：LineageOS 18.1（Android 11），精简为纯 WiFi 机
> 本手册基于实际构建成功并稳定运行的环境写成，日期 2026-10。

## 1. 源码树搭建

```bash
# LineageOS 18.1 基线
repo init -u https://github.com/LineageOS/android.git -b lineage-18.1

# 放入 local_manifest（见 local_manifest/nx523j.xml），然后：
repo sync -c -j$(nproc)
```

**设备树的两种获取方式（按优先级）：**

1. **首选：直接解压本包** —— `device-tree_nubia.tar.gz` 解压到源码根目录，
   得到 `device/nubia/msm8976-common` 和 `device/nubia/nx523j`。
   这是我们实际用于构建的树，和补丁严格对应。
2. 备选：从 MoKee 系仓库拉取（见 local_manifest 注释）。上游已停更，分支名请自行核对。

**vendor blob：** 解压 `vendor-blobs_nubia.tar.gz` 到源码根目录，得到
`vendor/nubia/msm8976-common`（约 176MB，压缩后 80MB）。
该目录不是 git 仓库，是从原厂 ROM 抽取的，务必与设备树同代使用。

**kernel：** `kernel/nubia/msm8976` 来自
`https://github.com/nubia-development/android_kernel_nubia_msm8976`（Q 分支），
本 devkit 的构建中内核无任何本地修改，直接用上游即可。

## 2. 应用补丁

见 `patches/APPLY_PATCHES.md`。全部补丁都是在干净的 lineage-18.1 检出上
以 `git diff` 生成的，在对应仓库根目录执行 `git apply xxx.patch` 即可。

## 3. 构建命令

```bash
source build/envsetup.sh
breakfast nx523j userdebug
mka bacon -j$(nproc)
```

产物：
- `out/target/product/nx523j/lineage-18.1-<日期>-UNOFFICIAL-nx523j.zip`
  —— TWRP 可直接刷入的完整卡刷包
- `out/target/product/nx523j/boot.img`、`recovery.img`、`system.img`、`vendor.img`
- 中间产物 target_files：`out/.../obj/PACKAGING/target_files_intermediates/`

## 4. 已踩过的坑（重要，逐条看）

### 4.1 陈旧 target_files 快照（最阴险的坑）
改了任何源码（尤其 build/make 下的 mk）重新 bacon 后，**out 根目录的
system.img / 刷机包可能仍是旧内容的重打包快照**——构建系统认为目标"已最新"
就跳过重打包。症状：改完代码刷入后毫无变化，甚至系统崩溃循环。

**解法（每次改完源码必做其一）：**
```bash
rm -rf out/target/product/nx523j/obj/PACKAGING/target_files_intermediates
# 或干脆 mka installclean
```
发布前务必用 simg2img + debugfs 解开镜像抽查内容，别相信时间戳。

### 4.2 蓝牙 32/64 位 vendor 属性名不一致
- 64 位 libbt-vendor 读 `vendor.qcom.bluetooth.soc`
- 32 位 libbt-vendor 读 `qcom.bluetooth.soc`
原厂库是 32 位的。我们让 HAL 服务 `compile_multilib: "prefer32"` 以匹配
原厂 32 位库，此时生效的是 **`qcom.bluetooth.soc=smd`**（`smd` 表示走
WCN 的 SMD 通道 /dev/smd3/4）。两个属性名都设置了，详见设备树 msm8976.mk。

### 4.3 setprop 不是持久的
运行时 `setprop` 重启即丢。一切属性必须固化进设备树
`PRODUCT_PROPERTY_OVERRIDES`。

### 4.4 蓝牙栈的超时链
蓝牙起不来时按顺序查：
1. `/vendor/lib(64)/libbt-vendor.so` 是否存在 → 没有 impl 库看日志
   `No Bluetooth Address`
2. `ro.bt.bdaddr_path` / `ro.boot.btmacaddr` / `persist.service.bdroid.bdaddr`
   三处 MAC 来源
3. native 层计时器：`bluetooth.enable_timeout_ms`（AOSP 默认 2900ms，
   我们设 15000；懒 HAL 查询很耗时）
4. Java 层计时器：`packages/apps/Bluetooth` 的 `AdapterState.java`
   中 `BLE_START_TIMEOUT_DELAY` / `BREDR_START_TIMEOUT_DELAY`
   （AOSP 默认 4000ms，我们设 15000）
5. HCI 层 HAL 版本：`system/bt/hci/src/hci_layer_android.cc` 会先查
   V1_1 再回落 V1_0，本机只有 V1.0，先查 V1.1 白白多花启动时间
6. **PBAP/MAP 会注册联系人观察者，删了 ContactsProvider 后必须关掉，
   否则开蓝牙即崩**（`packages/apps/Bluetooth/res/values/config.xml`）

### 4.5 音频（扬声器）根因
- tasha 外置编解码平台，HAL **只读** `audio_platform_info_extcodec.xml`，
  缺失时 backend_names 缺失，SPEAKER 映射到 `speaker-dolby` 后端
  （QUIN_MI2S_RX）失败 → 外放无声，听筒/耳机可能正常
- 功放是 TFA9890，DAPM 调试节点：
  `/sys/kernel/debug/asoc/msm8976-tasha-snd-card/tfa98xx.8-0034/dapm/{I2S1,bias_level}`
- 修复：把该 xml 放进 `device/nubia/msm8976-common/audio/` 并在
  msm8976.mk 用 PRODUCT_COPY_FILES 推到 `/vendor/etc/`

### 4.6 精简电话/联系人后
- PBAP/MAP 必须关（见 4.4 第 6 条）
- 删 Seedvault 后 `vendor/lineage/config/common_mobile.mk` 里会留下悬空的
  行尾反斜杠 `\`，会吞掉下一个 ifneq 块，编译期才爆错，注意检查

### 4.7 版本号（Settings → 关于手机）
`ro.build.display.id` 是 `build/make/core/Makefile` 无条件生成的
（非 user 构建用 build_desc），在设备 mk 里设 BUILD_DISPLAY_ID 无效。
需改 Makefile（见 build-make.patch），我们用 `ifeq ($(TARGET_DEVICE),nx523j)`
隔离。

## 5. 本 devkit 的基线状态（2026-10-25，v18）

- 开机待机空闲 RAM：约 2.2GB（目标 ≥2.3GB，精简版实测 1993–2222MB 区间）
- 蓝牙：开关稳定，配对可用，0 崩溃
- 扬声器/听筒/耳机：正常
- WiFi：正常，captive portal 已固化国内检测地址
- 已按设计移除：电话、短信、联系人、GPS、指纹、NFC、SE、Traceur、
  Seedvault、CalendarProvider
