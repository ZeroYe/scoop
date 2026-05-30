# ZeroYe Scoop Bucket

[![Tests](https://github.com/ZeroYe/scoop/actions/workflows/ci.yml/badge.svg)](https://github.com/ZeroYe/scoop/actions/workflows/ci.yml) [![Excavator](https://github.com/ZeroYe/scoop/actions/workflows/excavator.yml/badge.svg)](https://github.com/ZeroYe/scoop/actions/workflows/excavator.yml)

我的个人 [Scoop](https://scoop.sh) 软件仓库。

## 包含的软件

| 软件 | 说明 |
| --- | --- |
| [tailspin](https://github.com/bensadeh/tailspin) | 日志文件高亮工具 |
| [nali](https://github.com/nxtrace/nali) | 离线查询 IP 地理信息和 CDN 提供商的工具 |

## 安装方法

```pwsh
scoop bucket add ZeroYe-scoop https://github.com/ZeroYe/scoop
scoop install ZeroYe-scoop/<软件名>
```

## 自动更新

本仓库已配置 [Excavator](https://github.com/ScoopInstaller/Excavator)，每 4 小时自动检查更新。
