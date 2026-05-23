# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库简介

这是一个用于逆向工程工具的 Scoop bucket（`scoop-magic`），遵循标准 Scoop bucket 结构和规范。采用 Unlicense 许可证（公共领域）。

## 常用命令

所有命令均为 `bin/` 目录下的 PowerShell 脚本，它们会转发到 Scoop 安装目录（`$env:SCOOP_HOME/bin/`）中的对应脚本执行。

```powershell
# 运行 bucket 测试（使用 Pester v4）
bin\test.ps1

# 检查单个或所有 manifest 的新版本
bin\checkver.ps1 [<manifest-name>]

# 列出缺少 checkver（不支持自动更新）的 manifest
bin\missing-checkver.ps1

# 验证下载 URL 是否可访问
bin\checkurls.ps1

# 重新下载并验证哈希值
bin\checkhashes.ps1

# 格式化 JSON manifest
bin\formatjson.ps1

# 自动更新并创建 PR（默认上游为 lukesampson/scoop-extras:master）
bin\auto-pr.ps1 [-upstream <repo:branch>]
```

检查单个 manifest 版本示例：`bin\checkver.ps1 smali`

## 项目架构

- `bucket/` — JSON manifest 文件，每个包一个文件。每个 manifest 定义版本、下载 URL、哈希值、安装逻辑和自动更新规则。Schema: https://raw.githubusercontent.com/lukesampson/scoop/master/schema.json
- `bin/` — 轻量包装脚本，解析 `$env:SCOOP_HOME` 后转发到 Scoop 内置脚本，以 `bucket/` 目录为操作目标。
- `deprecated/` — 已废弃的 manifest（当前为空）。
- `Scoop-Bucket.Tests.ps1` — 根目录下的 Pester 测试入口，对仓库根目录执行 Pester 测试。

## Manifest 规范

- 行尾必须为 CRLF（由 `.gitattributes` 和 `.editorconfig` 强制要求）。
- JSON manifest 使用 4 空格缩进。
- 每个 manifest 应包含 `checkver` 和 `autoupdate` 块以支持自动版本跟踪。
- `homepage` 和 `description` 字段为必填项。

## CI

AppVeyor 在 PowerShell 5 和 6（pwsh）环境下运行。CI 会克隆 Scoop 核心，然后对该 bucket 运行测试。配置见 `appveyor.yml`。
