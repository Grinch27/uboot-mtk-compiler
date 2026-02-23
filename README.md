# uboot-mtk-compiler

Github Actions for u-boot complie targets hanwckf/bl-mt798x.

本项目旨在将 u-boot 编译环境部署到 Github Actions 上，主要面向 mt7981 和 mt7986 平台，提供自动化的编译流程。

## 从 hanwckf/bl-mt798x 过渡到 OpenWrt U-Boot layout

原有的 u-boot 编译环境基于 hanwckf/bl-mt798x 项目。随着 OpenWrt 对 mt7981 和 mt7986 平台的支持日益完善，建议使用者迁移到 OpenWrt 的 u-boot，以获得更好的维护和更新支持。

- [hanwckf_uboot](https://github.com/hanwckf/bl-mt798x "bl-mt798x"): 解锁 SSH -> 刷 `FIP` 分区
- OpenWrt U-Boot layout：解锁 SSH -> 安装内核 `kmod-mtd-rw` 模块 -> 刷 `BL2` 和 `FIP` 分区

## 原 Xiaomi AX3000T

原版 Xiaomi AX3000T (`MediaTek MT7981BA`, `MediaTek MT7976CN`, `MediaTek MT7531AE`)

<details>
<summary><strong>0. 获取 SSH</strong></summary>

#### 从互联网途径获取到的 AX3000T root 初始密码计算代码

在 python 环境中运行，根据设备上印刷的 SN 码计算得到 root 初始密码。

```python
import sys
import hashlib

salt = {
    'r1d': 'A2E371B0-B34B-48A5-8C40-A7133F3B5D88',
    'others': '6D2DF50A-250F-4A30-A5E6-D44FB0960AA0',
}


def get_pass(sn: str):
    # SN码加盐后进行md5加密取前八位字符串返回
    return hashlib.md5((sn + get_salt(sn)).encode()).hexdigest()[:8]


def get_salt(sn):
    # 通过判断字符串中是否含有/来获取对应的盐
    if '/' in sn:
        return salt['others'].lower()
    return salt['r1d']


def main():
    try:
        sn = sys.argv[1]
    except IndexError:
        sn = input('请输入SN码: ')
    print(get_pass(sn))


if __name__ == '__main__':
    main()
```

#### 在登录了原厂管理界面后，检查管理网页的网址

复制存储网址中的 `stok` 的值，该值为临时值，例如

http://192.168.31.1/cgi-bin/luci/;stok=1234567890abcdefg/web/home#router

其中 `1234567890abcdefg` 就是本次 `stok` 的值。

#### 使用 shell 脚本开启 dropbear SSH 服务

将调用 enable_dropbear 后的 `stok` 值进行实际修改，运行后即可开启 dropbear SSH 服务。

```bash
enable_dropbear() {
    local stok=""
    local ip="192.168.31.1"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -s|--stok)
                stok="$2"
                shift 2
                ;;
            -i|--ip)
                ip="$2"
                shift 2
                ;;
            *)
                break
                ;;
        esac
    done
    local base="http://${ip}/cgi-bin/luci/;stok=${stok}/api/misystem/arn_switch"
    # curl in linux shell
    curl -fsS -X POST "$base" -d "open=1&model=1&level=%0Anvram%20set%20ssh_en%3D1%0A"
    curl -fsS -X POST "$base" -d "open=1&model=1&level=%0Anvram%20commit%0A"
    curl -fsS -X POST "$base" -d "open=1&model=1&level=%0Ased%20-i%20's%2Fchannel%3D.*%2Fchannel%3D%22debug%22%2Fg'%20%2Fetc%2Finit.d%2Fdropbear%0A"
    curl -fsS -X POST "$base" -d "open=1&model=1&level=%0A%2Fetc%2Finit.d%2Fdropbear%20start%0A"
}
# 示例：根据实际情况修改 stok 和 ip
enable_dropbear --stok 1234567890abcdefg --ip 192.168.31.1
```

此时，可以通过 SSH 登录到设备，默认用户名为 root，密码为前面计算得到的初始密码。

```bash
# 清理旧ssh密钥
ssh-keygen -R 192.168.31.1
# 登录设备
ssh root@192.168.31.1
```

</details>

<details>
<summary><strong>1. 备份</strong></summary>

在进行任何修改之前，建议先备份设备的现有固件和配置，以防止意外情况发生。

#### 查看设备 MTD 分区信息

如非特殊说明，以下命令均在主机的 shell 终端中执行。

```bash
# 查看分区情况
ssh root@192.168.31.1 'cat /proc/mtd'

# 备份所有分区到 /tmp 目录
# 注意，请根据实际的 mtd 分区数量调整以下命令
backup_mtd() {
    local ip="192.168.31.1"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"
                shift 2
                ;;
            *)
                break
                ;;
        esac
    done
    declare -A MTDS=(
        ["/dev/mtd0"]="spi0.bin"
        ["/dev/mtd1"]="BL2.bin"
        ["/dev/mtd2"]="Nvram.bin"
        ["/dev/mtd3"]="Bdata.bin"
        ["/dev/mtd4"]="Factory.bin"
        ["/dev/mtd5"]="FIP.bin"
        ["/dev/mtd6"]="crash.bin"
        ["/dev/mtd7"]="crash_log.bin"
        ["/dev/mtd8"]="ubi.bin"
        ["/dev/mtd9"]="ubi1.bin"
        ["/dev/mtd10"]="overlay.bin"
        ["/dev/mtd11"]="data.bin"
        ["/dev/mtd12"]="KF.bin"
    )
    local hash_remote
    local hash_local
    local filename
    for mtd in "${!MTDS[@]}"; do
        filename="${MTDS[$mtd]}"
        echo "Backup ${ip}: ${mtd} -> ${filename}"
        hash_remote=$(ssh root@"$ip" "sha256sum ${mtd}" | awk '{print $1}')
        ssh root@"$ip" "dd if=${mtd} bs=4M status=none" | dd of="./${filename}" bs=4M status=progress
        hash_local=$(sha256sum "./${filename}" | awk '{print $1}')
        [[ "$hash_remote" == "$hash_local" ]] || { echo "校验失败: ${mtd}"; return 1; }
    done
}
# 示例：根据实际情况修改 ip
backup_mtd --ip 192.168.31.1
```

</details>

<details>
<summary><strong>2. 刷写 MTD 分区</strong></summary>

在实践中，hanwckf_uboot 刷写 `FIP` 分区即可用于后续系统固件的输入。
但 OpenWrt U-Boot layout 需要刷写 `BL2` 和 `FIP` 分区，且往往需要安装 `kmod-mtd-rw` 模块以支持写入操作。
因而，推荐的操作顺序是：

- 将 hanwckf_uboot 刷写至 `FIP` 分区。
- 完成后重启设备并进入恢复模式，导入 hanwckf_uboot 支持的 OpenWrt 固件(集成 `kmod-mtd-rw` 模块)进行系统安装。
- 成功安装(集成 `kmod-mtd-rw` 模块)的 OpenWrt 系统后，登录设备，使用 `kmod-mtd-rw` 模块将 OpenWrt U-Boot layout 刷写 `BL2` 和 `FIP` 分区。
- 完成后重启设备并进入恢复模式，导入 OpenWrt 固件进行系统安装。

#### 将 hanwckf_uboot 刷写至 FIP 分区

首先，确保已经从 hanwckf/bl-mt798x 项目编译并获取到所需的 u-boot 固件文件，例如 `mt7981_ax3000t-fip-fixed-parts-multi-layout.bin`。

并将该文件上传至设备的 `/tmp` 目录下，以下提供一个上传文件并校验的示例脚本：

```bash
upload_file() {
    local ip="192.168.31.1"
    local file=""
    local remote_dir="/tmp"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -r|--remote_dir)
                remote_dir="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -f "${file}" ]] || { echo "[跳过] 未找到本地文件: ${file}"; return 1; }
    local filename="$(basename "${file}")"
    local remote_path="${remote_dir%/}/${filename}"
    local hash_local
    local hash_remote

    hash_local=$(sha256sum "${file}" | awk '{print $1}')
    echo -e "local: ${file}\nsha256=${hash_local}"

    if ! ssh root@"${ip}" "cat > '${remote_path}'" < "${file}"; then
        echo "local -> remote: 传输失败: ${file}"
        return 1
    fi

    hash_remote=$(ssh root@"${ip}" "sha256sum '${remote_path}' 2>/dev/null" | awk '{print $1}')
    [[ -n "${hash_remote}" ]] || { echo "remote: 未获取到 sha256: ${remote_path}"; return 1; }

    if [[ "${hash_remote}" == "${hash_local}" ]]; then
        echo "[OK] 校验通过 ${filename} sha256=${hash_remote}"
    else
        echo "[不匹配] ${filename} 本地=${hash_local} 远端=${hash_remote}"
        return 1
    fi
}
# 示例 将文件上传至设备 /tmp 目录，并校验
upload_file --file "./mt7981_ax3000t-fip-fixed-parts-multi-layout.bin"
```

上传完成后，使用以下命令将 FIP 分区刷写：

!!! 注意请先使用 `ssh root@192.168.31.1 'cat /proc/mtd'` 检查 `FIP` 分区名称大小写情况: 一般为 `FIP` 或 `fip`，请在命令中保持名称 `FIP` 或 `fip` 与对应实际一致 !!!

```bash
# 刷写 FIP 分区
flash_mtd() {
    local ip="192.168.31.1"
    local file=""
    local partition=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -p|--partition)
                partition="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -n "${file}" ]] || { echo "缺少参数: --file"; return 1; }
    [[ -n "${partition}" ]] || { echo "缺少参数: --partition"; return 1; }

    local remote_file="/tmp/$(basename "${file}")"
    set -x
    ssh root@"${ip}" "mtd erase ${partition}" || return 1
    ssh root@"${ip}" "mtd write '${remote_file}' ${partition}" || return 1
    ssh root@"${ip}" "mtd verify '${remote_file}' ${partition}"
}

# 查看分区情况
ssh root@192.168.31.1 "cat /proc/mtd"

flash_mtd --ip "192.168.31.1" --file "./mt7981_ax3000t-fip-fixed-parts-multi-layout.bin" --partition "FIP"
```

此步骤可跳过：清除 pstore

```bash
# 清除pstore防止启动到恢复模式
ssh root@192.168.31.1 "rm -f /sys/fs/pstore/*"
```

!!! 请认真确认成功刷写 FIP 分区后再进行下一步操作，避免刷入失败后重启导致设备无法启动 !!!

完成后，`reboot`重启设备并进入恢复模式，导入支持 hanwckf_uboot 的 OpenWrt 固件进行系统安装。

```bash
ssh root@192.168.31.1 "reboot"
```

#### 刷写 hanwckf_uboot 支持的 OpenWrt 系统固件(集成 `kmod-mtd-rw` 模块)

此步骤较为简单，请自行参考 [hanwckf_uboot](https://github.com/hanwckf/bl-mt798x "bl-mt798x") 项目中的相关说明，完成过渡性 OpenWrt 系统固件(集成 `kmod-mtd-rw` 模块)的刷写。例如：

- [hanwckf/immortalwrt-mt798x](https://github.com/hanwckf/immortalwrt-mt798x "immortalwrt-mt798x")

拉取该项目源码，编译时确保`.config`文件内核模块中包含 `kmod-mtd-rw` 模块，编译得到系统固件后，在恢复模式界面中进行刷写。

注意：请使用当前 `hanwckf_uboot` 支持的分区布局的 OpenWrt 固件进行刷写，例如使用[hanwckf/immortalwrt-mt798x](https://github.com/hanwckf/immortalwrt-mt798x "immortalwrt-mt798x")进行编译的：

- https://github.com/Grinch27/openwrt-compiler/releases/tag/openwrt-21.02%2B5.4.284%2Bimmortalwrt-mt798x_7981
- https://github.com/Grinch27/openwrt-compiler/releases/tag/openwrt-21.02%2B5.4.284%2Bimmortalwrt-mt798x_7986

#### 使用 kmod-mtd-rw 模块刷写适配 ubootmod 的 BL2 和 FIP 分区（OpenWrt U-Boot layout）

默认情况下，OpenWrt 系统并不支持直接写入 MTD 分区，需要安装 `kmod-mtd-rw` 模块以启用该功能。该模块一般不支持热插拔安装，建议在编译 OpenWrt 固件时将其集成进去。

首先，确保已经从 OpenWrt 项目编译并获取到所需的 u-boot 固件文件，例如：

- openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin
- openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-bl31-uboot.fip

其中， `preloader.bin` 对应 BL2 分区， `bl31-uboot.fip` 对应 FIP 分区。

将上述文件上传至设备的 `/tmp` 目录下，以下提供一个上传文件并校验的示例脚本：

```bash
upload_file() {
    local ip="192.168.31.1"
    local file=""
    local remote_dir="/tmp"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -r|--remote_dir)
                remote_dir="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -f "${file}" ]] || { echo "[跳过] 未找到本地文件: ${file}"; return 1; }
    local filename="$(basename "${file}")"
    local remote_path="${remote_dir%/}/${filename}"
    local hash_local
    local hash_remote

    hash_local=$(sha256sum "${file}" | awk '{print $1}')
    echo -e "local: ${file}\nsha256=${hash_local}"

    if ! ssh root@"${ip}" "cat > '${remote_path}'" < "${file}"; then
        echo "local -> remote: 传输失败: ${file}"
        return 1
    fi

    hash_remote=$(ssh root@"${ip}" "sha256sum '${remote_path}' 2>/dev/null" | awk '{print $1}')
    [[ -n "${hash_remote}" ]] || { echo "remote: 未获取到 sha256: ${remote_path}"; return 1; }

    if [[ "${hash_remote}" == "${hash_local}" ]]; then
        echo "[OK] 校验通过 ${filename} sha256=${hash_remote}"
    else
        echo "[不匹配] ${filename} 本地=${hash_local} 远端=${hash_remote}"
        return 1
    fi
}
# 示例 将文件上传至设备 /tmp 目录，并校验
upload_file --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin"
upload_file --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-bl31-uboot.fip"
```

上传完成后，使用以下命令将 BL2 和 FIP 分区刷写，根据实际 cat /proc/mtd 显示的 BL2 和 FIP 实际名称进行调整（注意大小写）：

```bash
# 清理旧ssh密钥
ssh-keygen -R 192.168.1.1

# 查看分区情况
ssh root@192.168.1.1 "cat /proc/mtd"

# dev:    size   erasesize  name
# mtd0: 00100000 00020000 "BL2"
# mtd1: 00040000 00020000 "Nvram"
# mtd2: 00040000 00020000 "Bdata"
# mtd3: 00200000 00020000 "Factory"
# mtd4: 00200000 00020000 "FIP"
# mtd5: 00040000 00020000 "crash"
# mtd6: 00040000 00020000 "crash_log"
# mtd7: 00040000 00020000 "KF"
# mtd8: 07000000 00020000 "ubi"

# 加载 mtd-rw 模块以启用写入功能
ssh root@192.168.1.1 "insmod mtd-rw i_want_a_brick=1"

# 注意，请根据实际 cat /proc/mtd 显示的BL2和FIP实际名称进行调整（注意大小写）

flash_mtd() {
    local ip="192.168.1.1"
    local file=""
    local partition=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -p|--partition)
                partition="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -n "${file}" ]] || { echo "缺少参数: --file"; return 1; }
    [[ -n "${partition}" ]] || { echo "缺少参数: --partition"; return 1; }

    local remote_file="/tmp/$(basename "${file}")"
    set -x
    ssh root@"${ip}" "mtd erase ${partition}" || return 1
    ssh root@"${ip}" "mtd write '${remote_file}' ${partition}" || return 1
    ssh root@"${ip}" "mtd verify '${remote_file}' ${partition}"
}

# 查看分区情况
ssh-keygen -R '192.168.1.1'
ssh root@192.168.1.1 "cat /proc/mtd"

# 刷写 BL2 分区
flash_mtd --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin" --partition "BL2"

# 刷写 FIP 分区
flash_mtd --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-bl31-uboot.fip" --partition "FIP"

# 清除pstore防止启动到恢复模式（此步命令可跳过）
rm -f /sys/fs/pstore/*

ssh root@192.168.1.1 "reboot"
```

到此为止，适配 ubootmod 的 BL2 和 FIP 分区（OpenWrt U-Boot layout）已刷写完成。可进入恢复模式，导入适配 ubootmod 的 OpenWrt 固件进行系统安装。

</details>

<details>
<summary><strong>3. 刷入 OpenWrt 系统固件</strong></summary>

设备进入恢复模式，主机启用 tftp 服务，实现将`initramfs-recovery.itb`固件传输至设备

安装并启用 tftp 服务

```bash
# 更新软件包列表，安装tftpd-hpa服务器
sudo apt update
sudo apt install -y tftpd-hpa

# 创建TFTP目录
sudo mkdir -p /srv/tftp
sudo chmod 777 /srv/tftp

# 编辑TFTP配置文件
sudo vi /etc/default/tftpd-hpa
# 将配置文件内容修改为：
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

将适配 ubootmod 的 OpenWrt 固件（例如 `openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-initramfs-recovery.itb`）复制到 `/srv/tftp` 目录下：

此处文件名请根据 ubootmod 的 env 环境变量进行对应放置。

并将主机 IP 地址设置为`192.168.1.254`，重启 tftp 服务：

```bash
# 重启TFTP服务
sudo systemctl restart tftpd-hpa
# 确认服务运行状态
sudo systemctl status tftpd-hpa
```

重启设备并根据设备对应的恢复处理方式进入恢复模式，稍等片刻后，tftp 服务端会显示文件传输日志，表示设备已成功从 tftp 服务器获取到固件文件。或者可根据主机网络流量监控确认文件传输情况。

等待设备完成固件写入后，设备自动，完成系统安装。并清理相关临时工具。

```bash
# 停止当前运行的tftpd-hpa服务
sudo systemctl stop tftpd-hpa
# 禁用服务开机自启动
sudo systemctl disable tftpd-hpa
# 确认服务已停止并禁用
sudo systemctl status tftpd-hpa
```

</details>

## 以 mt7981_nokia_ea0326gmp 为例(简易教程)

前提：默认用户已经实现 SSH 登录。可复用的代码请参考前篇 AX3000T。

<details>
<summary><strong>1. 备份</strong></summary>

在进行任何修改之前，建议先备份设备的现有固件和配置，以防止意外情况发生。

#### 查看设备 MTD 分区信息

如非特殊说明，以下命令均在主机的 shell 终端中执行。

```bash
# 查看分区情况
ssh root@192.168.10.1 'cat /proc/mtd'

# 备份所有分区到 /tmp 目录
# 注意，请根据实际的 mtd 分区数量调整以下命令

# 示例：根据实际情况修改 ip
backup_mtd --ip 192.168.10.1
```

</details>

<details>
<summary><strong>2. 刷写 MTD 分区（hanwckf_uboot）</strong></summary>

简单略过重复说明。

#### 将 hanwckf_uboot 刷写至 FIP 分区

简单略过重复说明。

```bash
upload_file() {
    local ip="192.168.31.1"
    local file=""
    local remote_dir="/tmp"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -r|--remote_dir)
                remote_dir="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -f "${file}" ]] || { echo "[跳过] 未找到本地文件: ${file}"; return 1; }
    local filename="$(basename "${file}")"
    local remote_path="${remote_dir%/}/${filename}"
    local hash_local
    local hash_remote

    hash_local=$(sha256sum "${file}" | awk '{print $1}')
    echo -e "local: ${file}\nsha256=${hash_local}"

    if ! ssh root@"${ip}" "cat > '${remote_path}'" < "${file}"; then
        echo "local -> remote: 传输失败: ${file}"
        return 1
    fi

    hash_remote=$(ssh root@"${ip}" "sha256sum '${remote_path}' 2>/dev/null" | awk '{print $1}')
    [[ -n "${hash_remote}" ]] || { echo "remote: 未获取到 sha256: ${remote_path}"; return 1; }

    if [[ "${hash_remote}" == "${hash_local}" ]]; then
        echo "[OK] 校验通过 ${filename} sha256=${hash_remote}"
    else
        echo "[不匹配] ${filename} 本地=${hash_local} 远端=${hash_remote}"
        return 1
    fi
}
# 示例 将文件上传至设备 /tmp 目录，并校验
upload_file --file "./mt7981_nokia_ea0326gmp-fip-fixed-parts.bin"
```

上传完成后，使用以下命令将 FIP 分区刷写：

!!! 注意请先使用 `ssh root@192.168.10.1 'cat /proc/mtd'` 检查 `FIP` 分区名称大小写情况: 一般为 `FIP` 或 `fip`，请在命令中保持名称 `FIP` 或 `fip` 与对应实际一致 !!!

```bash
# 刷写 FIP 分区
flash_mtd() {
    local ip="192.168.10.1"
    local file=""
    local partition=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -p|--partition)
                partition="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -n "${file}" ]] || { echo "缺少参数: --file"; return 1; }
    [[ -n "${partition}" ]] || { echo "缺少参数: --partition"; return 1; }

    local remote_file="/tmp/$(basename "${file}")"
    ssh root@"${ip}" "mtd erase ${partition}" || return 1
    ssh root@"${ip}" "mtd write '${remote_file}' ${partition}" || return 1
    ssh root@"${ip}" "mtd verify '${remote_file}' ${partition}"
}

flash_mtd --ip "192.168.10.1" --file "./mt7981_nokia_ea0326gmp-fip-fixed-parts.bin" --partition "FIP"
```

此步骤可跳过：清除 pstore

```bash
# 清除pstore防止启动到恢复模式
ssh root@192.168.10.1 "rm -f /sys/fs/pstore/*"
```

!!! 请认真确认成功刷写 FIP 分区后再进行下一步操作，避免刷入失败后重启导致设备无法启动 !!!

完成后，`reboot`重启设备并进入恢复模式，导入支持 hanwckf_uboot 的 OpenWrt 固件进行系统安装。

```bash
ssh root@192.168.10.1 "reboot"
```

#### 刷写 hanwckf_uboot 支持的 OpenWrt 系统固件(集成 `kmod-mtd-rw` 模块)

简单略过重复说明。

</details>

<details>
<summary><strong>3. 刷写 MTD 分区（OpenWrt U-Boot layout）</strong></summary>

#### 使用 kmod-mtd-rw 模块刷写适配 ubootmod 的 BL2 和 FIP 分区（OpenWrt U-Boot layout）

简单略过重复说明。

```bash
upload_file() {
    local ip="192.168.31.1"
    local file=""
    local remote_dir="/tmp"
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -r|--remote_dir)
                remote_dir="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -f "${file}" ]] || { echo "[跳过] 未找到本地文件: ${file}"; return 1; }
    local filename="$(basename "${file}")"
    local remote_path="${remote_dir%/}/${filename}"
    local hash_local
    local hash_remote

    hash_local=$(sha256sum "${file}" | awk '{print $1}')
    echo -e "local: ${file}\nsha256=${hash_local}"

    if ! ssh root@"${ip}" "cat > '${remote_path}'" < "${file}"; then
        echo "local -> remote: 传输失败: ${file}"
        return 1
    fi

    hash_remote=$(ssh root@"${ip}" "sha256sum '${remote_path}' 2>/dev/null" | awk '{print $1}')
    [[ -n "${hash_remote}" ]] || { echo "remote: 未获取到 sha256: ${remote_path}"; return 1; }

    if [[ "${hash_remote}" == "${hash_local}" ]]; then
        echo "[OK] 校验通过 ${filename} sha256=${hash_remote}"
    else
        echo "[不匹配] ${filename} 本地=${hash_local} 远端=${hash_remote}"
        return 1
    fi
}
# 示例 将文件上传至设备 /tmp 目录，并校验
upload_file --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-nokia_ea0326gmp-preloader.bin"
upload_file --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-nokia_ea0326gmp-bl31-uboot.fip"
```

上传完成后，使用以下命令将 BL2 和 FIP 分区刷写，根据实际 cat /proc/mtd 显示的 BL2 和 FIP 实际名称进行调整（注意大小写）：

```bash
# 查看分区情况
ssh root@192.168.1.1 "cat /proc/mtd"
# dev:    size   erasesize  name
# mtd0: 00100000 00020000 "bl2"
# mtd1: 00080000 00020000 "u-boot-env"
# mtd2: 00200000 00020000 "factory"
# mtd3: 00200000 00020000 "fip"
# mtd4: 00200000 00020000 "config"
# mtd5: 00200000 00020000 "config2"
# mtd6: 07680000 00020000 "ubi"

# 注意，请根据实际 cat /proc/mtd 显示的BL2和FIP实际名称进行调整（注意大小写）

flash_mtd() {
    local ip="192.168.1.1"
    local file=""
    local partition=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            -p|--partition)
                partition="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -n "${file}" ]] || { echo "缺少参数: --file"; return 1; }
    [[ -n "${partition}" ]] || { echo "缺少参数: --partition"; return 1; }

    local remote_file="/tmp/$(basename "${file}")"
    ssh root@"${ip}" "mtd erase ${partition}" || return 1
    ssh root@"${ip}" "mtd write '${remote_file}' ${partition}" || return 1
    ssh root@"${ip}" "mtd verify '${remote_file}' ${partition}"
}

# 清理旧ssh密钥
ssh-keygen -R '192.168.1.1'
# 查看分区情况
ssh root@192.168.1.1 "cat /proc/mtd"

# 加载 mtd-rw 模块以启用写入功能
ssh root@192.168.1.1 "insmod mtd-rw i_want_a_brick=1"

# 刷写 BL2 分区
flash_mtd --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-nokia_ea0326gmp-preloader.bin" --partition "bl2"

# 刷写 FIP 分区
flash_mtd --ip "192.168.1.1" --file "./openwrt-mediatek-filogic-nokia_ea0326gmp-bl31-uboot.fip" --partition "fip"

# 清除pstore防止启动到恢复模式（此步命令可跳过）
ssh root@192.168.1.1 "rm -f /sys/fs/pstore/*"

ssh root@192.168.1.1 "reboot"
```

简单略过重复说明。

</details>

<details>
<summary><strong>4. 刷入 OpenWrt 系统固件</strong></summary>

设备进入恢复模式，主机启用 tftp 服务，实现将`initramfs-recovery.itb`固件传输至设备

安装并启用 tftp 服务

```bash
# 更新软件包列表，安装tftpd-hpa服务器
sudo apt update
sudo apt install -y tftpd-hpa

# 创建TFTP目录
sudo mkdir -p /srv/tftp
sudo chmod 777 /srv/tftp

# 编辑TFTP配置文件
sudo vi /etc/default/tftpd-hpa
# 将配置文件内容修改为：
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

将适配 ubootmod 的 OpenWrt 固件（例如 `openwrt-mediatek-filogic-nokia_ea0326gmp-initramfs-recovery.itb`）复制到 `/srv/tftp` 目录下：

此处文件名请根据 ubootmod 的 env 环境变量进行对应放置。

并将主机 IP 地址设置为`192.168.1.254`，重启 tftp 服务：

```bash
# 重启TFTP服务
sudo systemctl restart tftpd-hpa
# 确认服务运行状态
sudo systemctl status tftpd-hpa
```

重启设备并根据设备对应的恢复处理方式进入恢复模式，稍等片刻后，tftp 服务端会显示文件传输日志，表示设备已成功从 tftp 服务器获取到固件文件。或者可根据主机网络流量监控确认文件传输情况。

等待设备完成固件写入后，设备自动，完成系统安装。并清理相关临时工具。

```bash
# 停止当前运行的tftpd-hpa服务
sudo systemctl stop tftpd-hpa
# 禁用服务开机自启动
sudo systemctl disable tftpd-hpa
# 确认服务已停止并禁用
sudo systemctl status tftpd-hpa
```

</details>
