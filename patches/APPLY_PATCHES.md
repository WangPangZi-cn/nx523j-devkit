# 补丁应用说明

所有补丁在干净的 LineageOS 18.1 检出上以 `git diff` 生成，
在**对应仓库根目录**执行：

```bash
cd <仓库路径>
git apply --check <devkit>/patches/<补丁名>.patch   # 先预检
git apply <devkit>/patches/<补丁名>.patch
```

| 补丁文件 | 应用到的仓库 | 内容 |
|---|---|---|
| `build-make.patch` | `build/make` | ① 精简：handheld_system.mk / base_system.mk / aosp_base_telephony.mk / aosp_product.mk / full_base.mk / handheld_product.mk / mainline.mk 移除 Telephony、ContactsProvider、CalendarProvider、Nfc、SecureElement、Traceur 等；② `core/Makefile`：BUILD_DISPLAY_ID 中文版本号（仅 TARGET_DEVICE=nx523j 生效） |
| `vendor-lineage.patch` | `vendor/lineage` | 移除 Seedvault；common.mk / common_mobile.mk / telephony.mk 的精简与悬空 `\` 修复 |
| `packages-apps-Bluetooth.patch` | `packages/apps/Bluetooth` | ① AdapterState.java BLE/BREDR 启动超时 4000→15000ms；② 关闭 PBAP/MAP（依赖 ContactsProvider） |
| `system-bt.patch` | `system/bt` | hci_layer_android.cc：V1_0 HAL 优先（本机只有 V1.0，避免懒查询耗时） |
| `hardware-interfaces.patch` | `hardware/interfaces` | bluetooth@1.0 HAL 服务 `compile_multilib: "prefer32"`，匹配原厂 32 位 libbt-vendor |
| `packages-modules-NetworkStack.patch` | `packages/modules/NetworkStack` | captive portal 检测地址固化为 `connect.rom.miui.com/generate_204`（三处默认值） |

**应用补丁后必须清掉打包缓存再编译**（否则产物可能是旧快照）：

```bash
rm -rf out/target/product/nx523j/obj/PACKAGING/target_files_intermediates
```
