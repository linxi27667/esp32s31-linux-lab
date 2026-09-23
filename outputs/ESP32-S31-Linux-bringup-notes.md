# ESP32-S31 Function Core Board-1 Linux + Ethernet + SSH

## 版本

- Linux 6.18.0，Espressif S31 BSP，Buildroot 2025.02
- 16 MiB NOR 全量镜像，写入地址 `0x0`
- 镜像：`ESP32-S31-FunctionCoreBoard1-Ethernet-SSH.bin`
- SHA-256：`7894007f88987f5c3f9a2cbfa5f2d56dbaa13690a5d767b0c9689aed28d2a329`

## 已完成的硬件验证

- 在 COM8 / 115200 串口看到 ROM → SPL → OpenSBI → U-Boot → Linux 6.18 → Buildroot cramfs 启动链。
- S31 GMAC 被 STMMAC 驱动识别为 DWMAC1000；YT8531 PHY 驱动成功绑定，接口为 `eth0`，工作模式 `rgmii-id`。
- 已修复非一致性 DMA：默认 coherent DMA 池来自 S31 64 KiB SRAM。启动日志显示 `DMA: default coherent area is set`、`Register MEM_TYPE_PAGE_POOL RxQ-0`，描述符分配通过。
- 串口 shell 检查确认 `/usr/sbin/dropbear -s -g -p 22 -R` 正在运行，`/proc/net/tcp` 中 TCP 22 为 LISTEN。
- DHCP 客户端已启动并在后台等待租约。

## 当前待用户接线确认

串口检查结果：`/sys/class/net/eth0/carrier` 为 `0`，`operstate` 为 `down`，接口显示 `NO-CARRIER`，因此当前没有有效的以太网物理链路，尚未取得 DHCP 地址，也尚未完成客户端 SSH 登录验证。将板载 RJ45 接到有 DHCP 的路由器或交换机后，在串口运行 `ip -4 addr show eth0`，然后用 README 中的 SSH 命令连接。

## 安全与恢复

- Dropbear 禁用密码登录，仅接受镜像中的公钥。
- 对应私钥保存在本机 `outputs/s31-ssh/id_ed25519`，不提交 GitHub。
- 工厂 Flash 完整备份保存在本机 `outputs/ESP32-S31-FunctionCoreBoard1-factory-flash-backup.bin`，不提交 GitHub。
- 根文件系统只读，Dropbear 主机密钥在 `/run` 临时生成，重启后会变化。

串口启动和接口状态记录分别见 `ESP32-S31-Linux-serial-log.txt`、`ESP32-S31-Linux-network-status.txt`。
