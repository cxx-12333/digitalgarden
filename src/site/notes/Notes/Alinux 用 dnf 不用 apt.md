---
{"dg-publish": true, "dg-permalink": "alinux-dnf-not-apt", "permalink": "/alinux-dnf-not-apt/", "title": "Alinux 用 dnf 不用 apt", "tags": ["alinux", "dnf", "linux"], "created": "2026-08-14", "updated": "2026-08-14", "dg-note-properties": {"title": "Alinux 用 dnf 不用 apt", "type": "pitfall", "domain": "tech", "created": "2026-08-14", "updated": "2026-08-14", "tags": ["alinux", "dnf", "linux"]}}
---



# Alinux 用 dnf 不用 apt

> Alibaba Cloud Linux（alinux）是 RHEL 系，包管理器是 `dnf`/`yum`，**没有 apt**。

## 要点

- 安装软件：`dnf install <pkg>`（`yum` 是别名，也能用）
- apt / apt-get 在 alinux 上不存在，别照搬 Debian/Ubuntu 的命令
- 这台 ECS（主机名 wangshu）就是 alinux，装东西一律走 dnf

## 相关

- 环境事实：本机 python3.11 无 pip 模块，PEP 668 外部管理，用 venv 或 uv
