# Nubia Z11 Max (NX523J) 刷机指南 — lineage_nx523j 极限精简版 v18

> 适合机型：努比亚 Z11 Max（NX523J，MSM8976，4GB RAM）。**其他机型勿刷。**
> 系统特性：LineageOS 18.1（Android 11），纯 WiFi 机——无电话、短信、
> 联系人、GPS、指纹、NFC。

## 刷入前须知

- 机器须已解锁 Bootloader（本机此前已解锁；解锁会清空数据）
- 刷机会清空数据，请先备份
- 保持电量 > 50%

## 方式一：TWRP 卡刷（推荐）

1. 下载本包：`lineage-18.1-20261025-UNOFFICIAL-nx523j.zip`
2. 电脑 adb 进入 TWRP（本机 recovery 已预装 TWRP 3.3.1-0）：
   ```bash
   adb reboot recovery
   ```
   （若你的机器还是原厂 recovery，先 `adb reboot bootloader`，然后
   `fastboot flash recovery twrp-adbfree.img`，再 `adb reboot recovery`）
3. TWRP 中：Wipe → Advanced Wipe → 勾选 **Dalvik、System、Data、Cache**
   （跨版本/从原厂刷入必须双清；想要保留数据可只刷不清，但不保证干净）
4. Install → 选择 zip → 滑动刷入
5. 刷完 Reboot → System，首次开机约 2–3 分钟

## 方式二：adb sideload

```bash
adb reboot recovery          # 进入 TWRP
# TWRP: Advanced → ADB Sideload → 滑动开始
adb sideload lineage-18.1-20261025-UNOFFICIAL-nx523j.zip
```

## 方式三：手动 fastboot（救砖 / 不进 TWRP 也能刷 boot）

```bash
adb reboot bootloader
fastboot flash boot boot_v18.img     # 可选，卡刷包内已含 boot
fastboot reboot
```

## 校验（发布者已核）

| 文件 | SHA256 |
|---|---|
| lineage-18.1-20261025-UNOFFICIAL-nx523j.zip | `9df06a3986a63a45710acf975bf4415b6a5ebbcca78b613a098773422ee17e8a` |
| boot_v18.img | `fee91fb50c4b6316eab1935faeaa8c346bff0b3500feeed39d5e9afd5b376ef2` |

## 刷完后

- 设置 → 关于手机 → 版本号应显示：`lineage_nx523j-18.1-王胖子极限精简v18`
- WiFi 首次连接时的"网络检测"走国内地址（connect.rom.miui.com），不卡验证
- 蓝牙可正常开关与配对
- 无 SIM 卡功能属正常（本 ROM 设计如此）
