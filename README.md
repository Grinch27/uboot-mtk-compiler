# uboot-mtk-compiler

Github Actions for u-boot complie targets hanwckf/bl-mt798x.

本项目旨在将 u-boot 编译环境部署到 Github Actions 上，主要面向 mt7981 和 mt7986 平台，提供自动化的编译流程。

## 从 hanwckf/bl-mt798x 过渡到 OpenWrt U-Boot layout

以原版 Xiaomi AX3000T (MediaTek MT7981BA, MediaTek MT7976CN, MediaTek MT7531AE) 为例：

原有的 u-boot 编译环境基于 hanwckf/bl-mt798x 项目。随着 OpenWrt 对 mt7981 和 mt7986 平台的支持日益完善，建议使用者迁移到 OpenWrt 的 u-boot，以获得更好的维护和更新支持。

- [hanwckf_uboot](https://github.com/hanwckf/bl-mt798x "bl-mt798x"): 解锁 SSH -> 刷 FIP 分区
- OpenWrt U-Boot layout：解锁 SSH -> 安装内核 kmod-mtd-rw 模块 -> 刷 BL2 和 FIP 分区

### 0. 获取 SSH

#### 从互联网途径获取到的 AX3000T root 初始密码计算代码

在 python 环境中运行，根据设备上印刷的 SN 码计算得到 root 初始密码。

```python
import sys
import hashlib

salt = {
    'r1d': 'A2E371B0-B34B-48A5-8C40-A7133F3B5D88',
    'others': '6D2DF50A-250F-4A30-A5E6-D44FB0960AA0',
}


def main():
    try:
        sn = sys.argv[1]
    except IndexError:
        sn = input('请输入SN码: ')

    print(get_pass(sn))


def get_pass(sn: str):
    # SN码加盐后进行md5加密取前八位字符串返回
    return hashlib.md5((sn + get_salt(sn)).encode()).hexdigest()[:8]


def get_salt(sn):
    # 通过判断字符串中是否含有/来获取对应的盐
    if '/' in sn:
        return salt['others'].lower()
    return salt['r1d']


if __name__ == '__main__':
    main()
```

#### 在登录了原厂管理界面后，检查管理网页的网址

复制存储网址中的 stok 的值，该值为临时值，例如

http://192.168.31.1/cgi-bin/luci/;stok=1234567890abcdefg/web/home#router

其中 1234567890abcdefg 就是本次 stok 的值。

### 使用 shell 脚本开启 dropbear SSH 服务

将调用 enable_dropbear 后的 stok 值进行实际修改，运行后即可开启 dropbear SSH 服务。

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

### 1. 备份

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
        hash_remote=$(ssh root@"$ip" "busybox sha256sum ${mtd}" | awk '{print $1}')
        ssh root@"$ip" "dd if=${mtd} bs=4M status=none" | dd of="./${filename}" bs=4M status=progress
        hash_local=$(sha256sum "./${filename}" | awk '{print $1}')
        [[ "$hash_remote" == "$hash_local" ]] || { echo "校验失败: ${mtd}"; return 1; }
    done
}
# 示例：根据实际情况修改 ip
backup_mtd --ip 192.168.31.1
```

### 2. 刷写 MTD 分区

在实践中，hanwckf_uboot 刷写 FIP 分区即可用于后续系统固件的输入。
但 OpenWrt U-Boot layout 需要刷写 BL2 和 FIP 分区，且往往需要安装 kmod-mtd-rw 模块以支持写入操作。
因而，推荐的操作顺序是：

- 将 hanwckf_uboot 刷写至 FIP 分区。
- 完成后重启设备并进入恢复模式，导入 hanwckf_uboot 支持的 OpenWrt 固件(集成 kmod-mtd-rw 模块)进行系统安装。
- 成功安装(集成 kmod-mtd-rw 模块)的 OpenWrt 系统后，登录设备，使用 kmod-mtd-rw 模块将 OpenWrt U-Boot layout 刷写 BL2 和 FIP 分区。
- 完成后重启设备并进入恢复模式，导入 OpenWrt 固件进行系统安装。

#### 将 hanwckf_uboot 刷写至 FIP 分区

```bash
upload_file() {
    local ip="192.168.31.1"
    local file=""
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--ip)
                ip="$2"; shift 2 ;;
            -f|--file)
                file="$2"; shift 2 ;;
            *)
                break ;;
        esac
    done

    [[ -f "${file}" ]] || { echo "[跳过] 未找到本地文件: ${file}"; return 1; }
    local filename="$(basename "${file}")"
    local path_local="${file}"
    local path_remote="/tmp/${filename}"
    local hash_local
    local hash_remote

    hash_local=$(sha256sum "${path_local}" | awk '{print $1}')
    echo "[上传] ${path_local} (sha256=${hash_local})"

    if ! ssh root@"${ip}" "cat > '${path_remote}'" < "${path_local}"; then
        echo "local -> remote: 传输失败: ${path_local}"
        return 1
    fi

    hash_remote=$(ssh root@"${ip}" "sha256sum '${path_remote}' 2>/dev/null" | awk '{print $1}')
    [[ -n "${hash_remote}" ]] || { echo "remote: 未获取到 sha256: ${path_remote}"; return 1; }

    if [[ "${hash_remote}" == "${hash_local}" ]]; then
        echo "[OK] 校验通过 ${filename} sha256=${hash_remote}"
    else
        echo "[不匹配] ${filename} 本地=${hash_local} 远端=${hash_remote}"
        return 1
    fi
}
# 示例
upload_file --file "mt7981_ax3000t-fip-fixed-parts-multi-layout.bin"
```
