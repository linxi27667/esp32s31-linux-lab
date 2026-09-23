# ESP32-S31 Function Core Board-1 Linux Network Lab

为 ESP32-S31 Function Core Board-1 的 Espressif Linux 6.18 BSP 增加板载有线以太网和 SSH。仓库按用户要求设为私有。

## 当前硬件验证结果

- Linux 6.18 从 NOR 启动，Buildroot 根文件系统为只读 cramfs。
- STMMAC 已识别 S31 GMAC（Synopsys DWMAC1000），YT8531 PHY 驱动已绑定，网卡名为 `eth0`，模式为 `rgmii-id`。
- S31 DMA 缓存操作已通过 OpenSBI SBI ecall 工作；64 KiB SRAM 描述符池已作为默认 coherent DMA 池。内核日志出现 `DMA: default coherent area is set` 和 `Register MEM_TYPE_PAGE_POOL RxQ-0`，不再有 DMA 描述符分配失败。
- Dropbear 进程正在运行，串口检查 `/proc/net/tcp` 显示 TCP 22 处于 LISTEN。SSH 只允许预置公钥认证，未启用密码登录。
- 当前 `eth0` 的 `carrier=0`、`operstate=down`，DHCP 尚未拿到地址；需要先把板子的 RJ45 接到有 DHCP 的路由器/交换机。接上后，串口运行 `ip -4 addr show eth0` 查看地址。

验证记录见 `outputs/ESP32-S31-Linux-serial-log.txt` 和 `outputs/ESP32-S31-Linux-network-status.txt`。

## 连接 SSH

本机私钥位于 `outputs/s31-ssh/id_ed25519`，已经写入该固件对应公钥。私钥不会提交到 GitHub。

```powershell
ssh -i .\outputs\s31-ssh\id_ed25519 -o StrictHostKeyChecking=accept-new root@<板子的IPv4地址>
```

根文件系统只读，因此 Dropbear 主机密钥目前在 `/run` 临时生成，每次重启会变化。若客户端提示旧主机密钥冲突，执行 `ssh-keygen -R <板子的IPv4地址>` 后重连。

## 构建

内核补丁、Buildroot 外部树、rootfs overlay 和板级 defconfig 分别在 `buildroot-external/` 与 `configs/`。本机已用 Buildroot 2025.02、Espressif S31 BSP 分支构建；Linux 源码分支为 `integration/v6.18-esp32s31`。

在 Linux/WSL 中准备 Buildroot 2025.02 源码后运行：

```sh
make -C /path/to/buildroot O=/path/to/output \
  BR2_EXTERNAL="$PWD/buildroot-external" \
  espressif_esp32s31_function_core_board_nor_defconfig
make -C /path/to/buildroot O=/path/to/output \
  BR2_EXTERNAL="$PWD/buildroot-external" -j8
```

完整的预构建 Flash 镜像在 `outputs/ESP32-S31-FunctionCoreBoard1-Ethernet-SSH.bin`，大小 13,930,496 字节，适用于 16 MiB NOR，写入偏移为 `0x0`。当前镜像 SHA-256：

```text
7894007f88987f5c3f9a2cbfa5f2d56dbaa13690a5d767b0c9689aed28d2a329
```

## 烧录与恢复

确保 COM8 是该板的 USB-UART 串口，波特率 115200（串口控制台）或 460800（烧录）：

```powershell
python -m esptool --chip esp32s31 --port COM8 --baud 460800 write-flash 0x0 .\outputs\ESP32-S31-FunctionCoreBoard1-Ethernet-SSH.bin
```

原始 16 MiB 工厂 Flash 备份保存在本机 `outputs/ESP32-S31-FunctionCoreBoard1-factory-flash-backup.bin`，不会上传。需要恢复时：

```powershell
python -m esptool --chip esp32s31 --port COM8 --baud 460800 write-flash 0x0 .\outputs\ESP32-S31-FunctionCoreBoard1-factory-flash-backup.bin
```

## 本地文件与密钥

`work/` 保存本机 Buildroot 输入包、工具与临时日志，整个目录不进入 Git。SSH 私钥和工厂 Flash 备份也在 `.gitignore` 中；推送前会检查暂存列表没有这两类文件。
