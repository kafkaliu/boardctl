# boardctl — AI agent 的通用板卡操作工具

凭一个 endpoint（`host:port + 用户 + 凭据`），让 AI agent（和人）经网络操作**任意 arm64 Linux 开发板**——虚拟板或真实板：

- **使用面**（任意机器可用）：`ssh` / `deploy`（arm64 交叉编译 + systemd）/ `console` / `status`
- **管理面**（经 `--via ssh://宿主机` 代理）：start / stop / flash / usb / port（由宿主机上的 [vmpi](https://github.com/kafkaliu/vmpi) 执行）
- **agent skill**：附 Agent Skills 规范的 skill，教 AI agent 板卡工作流与安全守则
- 真板友好：电源/烧录口等物理操作不越界；破坏性命令需显式确认

```bash
boardctl auth                  # 首次：密码登录并注入本机公钥（等价 ssh-copy-id）
boardctl ssh                   # 登录默认板子
boardctl deploy ./myapp --svc myapp
boardctl --board real-opi status
```

> 定位与完整规格见 [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)（文档先行：确认的需求先写入文档再开发）。
> 首个集成场景：操作 [vmpi](https://github.com/kafkaliu/vmpi) 模拟的虚拟 Orange Pi Zero 3（含 NanoKVM-USB 直通工作流）。
