# IoT Platform — 加密发布仓库

本仓库**只存放加密后的发布产物**，不包含任何源码。源码位于一个私有仓库，由 CI 构建后加密推送至此。

> 目录中看不到明文源码是设计使然：本仓库公开，而内容在离开构建环境前已被加密。

## 发布产物

每个 [Release](../../releases) 包含三个资产：

| 资产 | 内容 | 加密方式 | 解密方 |
|---|---|---|---|
| `frontend-dist.tar.gz.enc` | 前端静态产物（Vite 构建） | OpenSSL AES-256-CBC，PBKDF2 20 万次迭代 | 目标服务器持有的对称口令 |
| `backend-src.tar.gz.gpg` | 后端 FastAPI 应用源码 | OpenPGP（Ed25519 签名 / CV25519 加密） | 目标服务器持有的 GPG 私钥 |
| `manifest.json` | 版本、提交、各资产 SHA256、安装路径映射 | **明文** | 部署脚本校验用 |

`manifest.json` 保持明文，以便部署端在解密前先校验完整性；其中不含任何敏感信息。

## 安全模型

采用混合密钥方案，两类的权衡不同：

- **后端包（非对称）**：CI 只持有 GPG **公钥**，因此构建侧（以及任何拿到 CI 密钥的人）**无法解密**后端包。私钥仅存在于目标服务器的 `~/.iot-release/gnupg`。
- **前端包（对称）**：口令同时存在于 CI secret 与目标服务器。这是对称方案的固有权衡——好处是加密解密快且实现简单，代价是 CI 侧具备解密能力。

因此**真正的机密（生产配置、数据库口令、第三方密钥）从不进入发布包**，它们以 `.env.production` 形式只保留在目标服务器上，CI 打包时会显式扫描并拒绝任何 `.env*`、`*.key`、`*production*` 文件混入。

## 部署

目标服务器上已安装 `iot-release` 工具：

```bash
iot-release list                       # 列出可用版本
iot-release deploy                     # 部署最新版本
iot-release deploy v20260930-... --dry-run   # 仅下载/校验/解密，不改动线上
iot-release rollback                   # 回滚到上一版本
iot-release status                     # 查看当前部署状态
```

部署脚本会依次执行：下载 → 按 `manifest.json` 校验 SHA256 → 解密 → 备份当前版本 → 安装 → 重建镜像 → 数据库迁移 → 健康检查，健康检查失败会明确提示回滚命令。

## 密钥位置（目标服务器）

```
~/.iot-release/
├── gnupg/                  GPG 私钥环（后端包解密）
├── frontend-aes.key        前端包 AES 口令
├── release-priv-BACKUP.asc 私钥备份（务必离线另存）
└── deploy.log              部署日志
```

目录权限 `700`，密钥文件 `600`。

## 触发发布

发布由私有源码仓库的 GitHub Actions 完成：

- push 到 `main` 分支自动触发
- 或手动触发 `Encrypted Release` workflow

CI 会保留最近 5 个发布，更早的自动清理。
