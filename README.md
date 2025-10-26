# uboot-mtk-compiler

Github Actions for u-boot complie targets hanwckf/bl-mt798x.

本项目旨在将 u-boot 编译环境部署到 Github Actions 上，主要面向 mt7981 和 mt7986 平台，提供自动化的编译流程。

## 从 hanwckf/bl-mt798x 过渡到 OpenWrt

以原版 Xiaomi AX3000T (MediaTek MT7981BA, MediaTek MT7976CN, MediaTek MT7531AE) 为例：

原有的 u-boot 编译环境基于 hanwckf/bl-mt798x 项目。随着 OpenWrt 对 mt7981 和 mt7986 平台的支持日益完善，建议使用者迁移到 OpenWrt 的 u-boot，以获得更好的维护和更新支持。

### 0. hanwckf_uboot vs HZFrodo_ubootmod

- [hanwckf_uboot](https://github.com/hanwckf/bl-mt798x "bl-mt798x"): 解锁 SSH -> 刷 FIP 分区
- HZFrodo_ubootmod：解锁 SSH -> 安装内核 kmod-mtd-rw 模块 -> 刷 BL2 和 FIP 分区

### 1. 从原厂到 hanwckf/bl-mt798x
