# nx523j-devkit — Nubia Z11 Max (NX523J) LineageOS 18.1 开发者套件

> 一切后来者需要的、能在 Android 11 上继续开发 nx523j 的东西都在这里。
> 生成于 2026-10-25，对应发布版 v18
> （`lineage-18.1-20261025-UNOFFICIAL-nx523j.zip`，版本号
> `lineage_nx523j-18.1-王胖子极限精简v18`）。

## 内容清单

| 文件/目录 | 说明 |
|---|---|
| `BUILD_GUIDE.md` | **先读这个**。从零搭建源码树到出包的完整手册，含 7 个实战坑 |
| `patches/` | 6 个 git 补丁，覆盖 AOSP 侧全部修改（蓝牙×3、音频依赖、captive portal、精简、版本号），用法见 `patches/APPLY_PATCHES.md` |
| `device-tree_nubia.tar.gz` | 设备树（device/nubia/msm8976-common + nx523j），MoKee 系 Android 11 延续版，权威来源 |
| `vendor-blobs_nubia.tar.gz` | vendor 闭源 blob（vendor/nubia/msm8976-common），与设备树同代 |
| `kernel_nubia_nx523j.tar.gz` / `.tar.zst` | 内核源码 3.10.108（干净源码含固件，无编译产物），权威来源；zst 为 zstd 高压缩版（89MB），`zstd -d` 解开后再 tar |
| `local_manifest/nx523j.xml` | repo manifest 模板（含上游仓库核实情况说明） |
| `FLASH_GUIDE.md` | 面向普通用户的刷机说明（可单独转发） |
| `KNOWN_ISSUES.md` | 设计取舍清单 + 调试地图 + 性能基线 |

## 最短复现路径

```bash
# 1. LineageOS 18.1 源码树
repo init -u https://github.com/LineageOS/android.git -b lineage-18.1
repo sync -c -j$(nproc)

# 2. 解包三个 tar（见 local_manifest/nx523j.xml 头部注释；内核若为 .zst 版先 zstd -d）

# 3. 打 6 个补丁（见 patches/APPLY_PATCHES.md）

# 4. 清打包缓存并编译
rm -rf out/target/product/nx523j/obj/PACKAGING/target_files_intermediates
source build/envsetup.sh && breakfast nx523j userdebug && mka bacon -j$(nproc)
```

## 上游核实状态（诚实声明）

- kernel 上游仓库存在且已核实分支列表
  （github.com/nubia-development/android_kernel_nubia_msm8976，最高 q 分支），
  但与本地树的 commit 对应关系未逐条核实 → 用 tar 包最稳
- 设备树上游（MoKee）已停更且无公开的 Android 11 仓库 → tar 包是唯一来源
- vendor blob 从原厂 ROM 抽取，只存在于本包

**如果这个 devkit 帮到了你，建议把设备树/内核重新入库到 GitHub 并补上
本包缺失的 git 历史——这是后人能帮上忙的前提。**
