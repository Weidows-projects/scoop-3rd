# AGENT: Scoop 第三方软件库 Manifest 自动创建指南

## 项目概述

此仓库是 [scoop-apps](https://github.com/kkzzhizhou/scoop-apps) 的扩展库, 主要维护日常/开发使用的 Windows 软件.

- **Bucket 名称**: `3rd`
- **仓库地址**: https://github.com/Weidows-projects/scoop-3rd
- **Manifest 目录**: `bucket/` (204 个 manifest)
- **废弃目录**: `deprecated/` (25 个 manifest)
- **CI**: GitHub Actions 自动检查更新 (Excavator, 每日 UTC 0:00)

---

## 核心流程: 从 GitHub 链接到 Manifest

收到用户提供的 GitHub 链接后, 按以下步骤操作:

### 步骤 1: 分析 Release 页面

访问 `https://github.com/{owner}/{repo}/releases/latest`, 获取:

- **最新版本号** (tag 名称)
- **下载资源列表** (Assets)
  - 文件名及后缀 (.zip, .7z, .exe, .msi, .nupkg 等)
  - 多架构支持 (x64, x86, arm64)
  - 是否为安装包或便携版

### 步骤 2: 确定下载 URL 模式

记录 Release 资源的 URL 模板, 观察版本号在其中的位置:

```text
https://github.com/owner/repo/releases/download/v{version}/app-{version}-x64.zip
```

### 步骤 3: 确定版本提取方式 (checkver)

优先使用 GitHub API:

```json
"checkver": {
    "github": "https://github.com/owner/repo"
}
```

如果 GitHub 不适用, 从网页提取:

```json
"checkver": {
    "url": "https://example.com/download",
    "regex": "Version ([\\d.]+)"
}
```

### 工作目录

下载和分析的临时目录统一使用:

```
D:\Downloads\scoop\
```

### 步骤 4: 分析软件包类型

下载 Release 资源到工作目录并分析:

| 后缀 | 处理方式 | 示例 |
|------|----------|------|
| `.zip` / `.7z` / `.tar.gz` | 直接解压 (Scoop 自动处理) | `lan-mouse`, `EasySpider` |
| `.exe` 安装包 | 添加 `#/dl.7z` 重命名, 再用 `Expand-7zipArchive` 解包 | `Cursor`, `He3` |
| `.exe` 原版安装 | 用 `installer` 字段静默安装 | `warp-cloudflare` (msiexec) |
| `.msi` | 直接解压 (Scoop 自动处理) 或 `msiexec` 安装 | `biliup-app`, `warp-cloudflare` |
| `.nupkg` | 解压后清理 `_rels`, `package`, `*Content*.xml` | `psmodule-PowerShellAI` |
| InnoSetup 安装包 | `"innosetup": true` 自动解包 | 常见于 `*_setup.exe` |
| 需要 `#/dl.7z` 的 | NSIS 安装包, 需提取 `$PLUGINSDIR/app-64.7z` | `Cursor`, `He3` |

### 步骤 5: 确定安装目录内容

解压后检查:

1. **可执行文件**: 主 exe 名称 (用于 `bin` 和 `shortcuts`)
2. **子目录结构**: 是否需要 `extract_dir` 指定解压子目录
3. **配置文件/数据目录**: 用于 `persist` 持久化
4. **需要额外处理的文件**: 如 `$PLUGINSDIR` 清理

### 步骤 6: 编写 Manifest

使用下方 [Manifest 完整字段参考](#manifest-完整字段参考) 生成 JSON.

---

## Manifest 完整字段参考

### 基础字段 (必填)

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | string | 应用版本号, 如 `"1.2.3"` |
| `description` | string | 一行简短描述, 不含程序名 (如果与应用文件名相同) |
| `homepage` | string | 程序官网 |
| `license` | string/object | SPDX 标识符, 或 `{ "identifier", "url" }` 对象 |

### 下载与解压

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | string/array | 下载 URL, 可用 `#/dl.7z` 重命名文件 |
| `hash` | string/array | SHA256 哈希, 前缀 `sha512:`, `md5:`, `sha1:` 指定算法 |
| `extract_dir` | string | 压缩包内解压的子目录 |
| `extract_to` | string | Scoop 会将所有内容解压到指定目录 |
| `innosetup` | boolean | `true` 时自动处理 InnoSetup 安装包 |
| `architecture` | object | 架构特定配置, 包含 `64bit`, `32bit`, `arm64` |

### 架构子字段

`architecture.64bit` / `architecture.32bit` / `architecture.arm64` 可包含:

| 子字段 | 说明 |
|--------|------|
| `url` | 架构特定下载 URL |
| `hash` | 架构特定哈希 |
| `extract_dir` | 架构特定解压目录 |
| `bin` | 架构特定二进制 |
| `shortcuts` | 架构特定快捷方式 |
| `pre_install` / `post_install` | 架构特定脚本 |
| `installer` / `uninstaller` | 架构特定安装/卸载指令 |
| `checkver` | 架构特定版本检查 |

### 可执行文件

| 字段 | 类型 | 说明 |
|------|------|------|
| `bin` | string/array | 添加到 PATH 的可执行文件. 别名格式: `[ "原.exe", "别名", "--参数" ]` |
| `shortcuts` | array | 开始菜单快捷方式. 格式: `[ "target.exe", "显示名称", "参数", "图标路径" ]` |
| `env_add_path` | string/array | 需要添加到 PATH 的子目录 (相对安装目录) |
| `env_set` | object | 设置环境变量, `{ "VAR_NAME": "value" }` |

### 持久化

| 字段 | 类型 | 说明 |
|------|------|------|
| `persist` | string/array | 需要持久化的文件或目录, 如 `"data"`, `"config.json"`, `["Data", "config.json"]` |

> **注意**: `%APPDATA%` 和 `%LOCALAPPDATA%` 下的路径 (如 `%APPDATA%\bilimusic`) **不需要** persist, 除非有特殊需求.

### 依赖与建议

| 字段 | 类型 | 说明 |
|------|------|------|
| `depends` | string/array | 运行时依赖, 自动安装. 格式: `"7zip"` 或 `"extras/obs-studio"` |
| `suggest` | object | 建议安装的可选应用. 格式: `{ "OBS": ["extras/obs-studio", "extras/obs-studio-small"] }` |

### 脚本

| 字段 | 类型 | 说明 |
|------|------|------|
| `pre_install` | string/array | 安装前执行的 PowerShell 脚本 |
| `post_install` | string/array | 安装后执行的 PowerShell 脚本 |
| `pre_uninstall` | string/array | 卸载前执行的 PowerShell 脚本 |
| `post_uninstall` | string/array | 卸载后执行的 PowerShell 脚本 |
| `installer` | object | 安装器指令. 子字段: `file` (安装程序), `script` (替代脚本), `args` (参数数组), `keep` (保留安装程序) |
| `uninstaller` | object | 卸载指令, 同 `installer` |

### 自动更新

| 字段 | 类型 | 说明 |
|------|------|------|
| `checkver` | string/object | 版本检查规则 |
| `autoupdate` | object | 自动更新规则 |

### 其他

| 字段 | 类型 | 说明 |
|------|------|------|
| `notes` | string/array | 安装后显示的消息 |
| `psmodule` | object | 安装为 PowerShell 模块. `{ "name": "ModuleName" }` |
| `##` | string/array | 注释 |

---

## Manifest 字段顺序规范 (推荐)

为了保持一致性, 建议按以下顺序排列字段:

```
version
description
homepage
license
suggest          # 靠前, 便于查看依赖建议
architecture
shortcuts / bin  # GUI 用 shortcuts, CLI 用 bin
checkver
autoupdate
persist          # 如需要
depends          # 如需要
pre_install / post_install / ...  # 脚本类靠后
```

---

## 脚本环境变量

在 `pre_install`, `post_install`, `installer.script`, `uninstaller.script` 中可用:

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `$dir` | 安装目录 (pre_install 中为版本路径, post_install 中为 `current` 路径) | `C:\Users\...\scoop\apps\app\1.2.3` |
| `$persist_dir` | 持久化数据目录 | `C:\Users\...\scoop\persist\app` |
| `$version` | 当前安装版本 | `1.2.3` |
| `$architecture` | CPU 架构 | `64bit` 或 `32bit` |
| `$app` | 应用名 (manifest 文件名) | `exampleapp` |
| `$global` | 是否全局安装 | `$true` 或 `$false` |
| `$manifest` | 完整 manifest 对象 | PowerShell 哈希表 |
| `$cmd` | 当前子命令 | `install`, `update`, `uninstall` |
| `$scoopdir` | Scoop 根目录 | `C:\Users\...\scoop` |
| `$cachedir` | 缓存目录 | `C:\Users\...\scoop\cache` |

---

## checkver 配置模式

### 模式 1: GitHub 仓库 (最常用)

```json
"checkver": {
    "github": "https://github.com/owner/repo"
}
```

Scoop 自动匹配 tag, 正则 `/\/releases\/tag\/(?:v|V)?([\d.]+)/`, 忽略预发布版.

### 模式 2: 自定义 URL + 正则

```json
"checkver": {
    "url": "https://example.com/download",
    "regex": "Version ([\\d.]+)"
}
```

### 模式 3: JSON API

```json
"checkver": {
    "url": "https://api.example.com/version",
    "jsonpath": "$.version"
}
```

### 模式 4: 复杂正则 (带命名捕获组)

```json
"checkver": {
    "url": "https://github.com/owner/repo/releases/latest",
    "regex": "v(?<version>[\\d\\w.]+)/app-(?<short>[\\d.]+).*\\.zip"
}
```

### 模式 5: 反向匹配

```json
"checkver": {
    "url": "https://example.com/download",
    "regex": "([\\d.]+)",
    "reverse": true
}
```

---

## autoupdate 配置模式

### 模式 1: 简单 URL 替换

```json
"autoupdate": {
    "url": "https://example.com/download/v$version/app.zip",
    "hash": { "mode": "download" }
}
```

### 模式 2: 多架构

```json
"autoupdate": {
    "architecture": {
        "64bit": { "url": "https://.../app-$version-x64.zip" },
        "32bit": { "url": "https://.../app-$version-x86.zip" }
    },
    "hash": { "mode": "download" }
}
```

### 模式 3: 带 extract_dir 更新

```json
"autoupdate": {
    "url": "https://.../app-$version.zip",
    "extract_dir": "app-$version",
    "hash": { "mode": "download" }
}
```

### 模式 4: 从 GitHub expanded_assets 提取哈希 (推荐, 避免下载文件)

GitHub Release 页面的 `expanded_assets` 包含每个 asset 的 SHA256, 可直接提取:

```json
"autoupdate": {
    "architecture": {
        "64bit": {
            "url": "https://github.com/owner/repo/releases/download/v$version/app.exe",
            "hash": {
                "mode": "extract",
                "url": "https://github.com/owner/repo/releases/expanded_assets/v$version",
                "regex": "app\\.exe.*?sha256:([a-f0-9]{64})"
            }
        }
    }
}
```

**关键点**:
- `expanded_assets` URL 必须使用 **带 `v` 前缀** 的 tag: `v$version`
- 正则需使用 **捕获组** `([a-f0-9]{64})` 提取 hash
- 适用于近 1-2 年的 GitHub Release, 大多数新增软件均支持

### 可用版本变量

| 变量 | 示例 (`3.7.1.2`) |
|------|-------------------|
| `$version` | `3.7.1.2` |
| `$underscoreVersion` | `3_7_1_2` |
| `$dashVersion` | `3-7-1-2` |
| `$cleanVersion` | `3712` |
| `$majorVersion` | `3` |
| `$minorVersion` | `7` |
| `$patchVersion` | `1` |
| `$buildVersion` | `2` |
| `$matchHead` | `3.7.1` (前 2-3 段) |
| `$matchTail` | `-rc.1` (剩余部分) |

### URL 变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `$url` | 完整 URL (不含 fragment) | `http://example.com/path/file.exe` |
| `$baseurl` | URL 不含文件名 | `http://example.com/path` |
| `$basename` | 文件名 | `file.exe` |

### 哈希处理模式

| `mode` | 说明 |
|--------|------|
| `download` | 下载文件并本地计算哈希 (最常用回退) |
| `extract` | 从网页/文本中提取 (默认) |
| `json` | 从 JSON 文件中通过 JSONPath 提取 |
| `xpath` | 从 XML 中通过 XPath 提取 |
| `fosshub` | FossHub 自动处理 |
| `sourceforge` | SourceForge 自动处理 |
| `metalink` | 从 Metalink 提取 |
| `rdf` | 从 RDF 文件提取 |

---

## 常见场景与模板

### 场景 1: 标准 GitHub Release (zip/7z)

```json
{
    "version": "1.0.0",
    "description": "应用描述",
    "homepage": "https://github.com/owner/repo",
    "license": "MIT",
    "suggest": {
        "FFmpeg": ["ffmpeg"]
    },
    "architecture": {
        "64bit": {
            "url": "https://github.com/owner/repo/releases/download/v1.0.0/app-1.0.0-x64.zip",
            "hash": "sha256hash..."
        }
    },
    "bin": "app.exe",
    "shortcuts": [
        ["app.exe", "AppName"]
    ],
    "checkver": {
        "github": "https://github.com/owner/repo"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/owner/repo/releases/download/v$version/app-$version-x64.zip",
                "hash": {
                    "mode": "extract",
                    "url": "https://github.com/owner/repo/releases/expanded_assets/v$version",
                    "regex": "app-\\$version-x64\\.zip.*?sha256:([a-f0-9]{64})"
                }
            }
        }
    }
}
```

### 场景 2: NSIS 安装包 (需解包)

```json
{
    "version": "1.0.0",
    "description": "应用描述",
    "homepage": "https://example.com",
    "license": "MIT",
    "architecture": {
        "64bit": {
            "url": "https://example.com/download/setup.exe#/dl.7z",
            "hash": "sha256hash..."
        }
    },
    "pre_install": [
        "Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\"",
        "Remove-Item -Recurse -Force \"$dir\\`$PLUGINSDIR\""
    ],
    "bin": "app.exe",
    "shortcuts": [["app.exe", "AppName"]],
    "checkver": { "url": "https://example.com", "regex": "([\\d.]+)" },
    "autoupdate": {
        "url": "https://example.com/download/setup.exe#/dl.7z",
        "hash": { "mode": "download" }
    }
}
```

### 场景 3: 需要持久化配置

```json
{
    "version": "1.0.0",
    "description": "应用描述",
    "homepage": "https://github.com/owner/repo",
    "license": "MIT",
    "architecture": {
        "64bit": {
            "url": "https://github.com/owner/repo/releases/download/v1.0.0/app-1.0.0-x64.zip",
            "hash": "sha256hash..."
        }
    },
    "bin": "app.exe",
    "shortcuts": [["app.exe", "AppName"]],
    "persist": ["config", "data"],
    "checkver": { "github": "https://github.com/owner/repo" },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/owner/repo/releases/download/v$version/app-$version-x64.zip",
                "hash": {
                    "mode": "extract",
                    "url": "https://github.com/owner/repo/releases/expanded_assets/v$version",
                    "regex": "app-\\$version-x64\\.zip.*?sha256:([a-f0-9]{64})"
                }
            }
        }
    }
}
```

### 场景 4: 依赖其他软件 (如 OBS 插件)

```json
{
    "version": "1.0.0",
    "description": "OBS 插件",
    "homepage": "https://github.com/owner/repo",
    "license": "GPL-2.0",
    "suggest": {
        "OBS": ["extras/obs-studio", "extras/obs-studio-small"]
    },
    "architecture": {
        "64bit": {
            "url": "https://github.com/owner/repo/releases/download/v1.0.0/plugin-1.0.0-win64.zip",
            "hash": "sha256hash..."
        }
    },
    "pre_install": [
        "'obs-studio', 'OBSStudio-Portable' | ForEach-Object {",
        "   $obs = \"$(appdir $_ $global)\"",
        "   if (Test-Path \"$obs\") {",
        "       Copy-Item \"$dir\\data\\obs-plugins\\*\" \"$obs\\current\\data\\obs-plugins\" -Recurse",
        "       Copy-Item \"$dir\\obs-plugins\\64bit\\*\" \"$obs\\current\\obs-plugins\\64bit\" -Recurse",
        "   }",
        "}"
    ],
    "pre_uninstall": [
        "'obs-studio', 'OBSStudio-Portable' | ForEach-Object {",
        "    $obs = \"$(appdir $_ $global)\"",
        "    if (Test-Path $obs) {",
        "        Remove-Item \"$obs\\current\\data\\obs-plugins\\plugin-name\" -Force -Recurse",
        "        Remove-Item \"$obs\\current\\obs-plugins\\64bit\\plugin-name.*\" -Force",
        "    }",
        "}"
    ],
    "checkver": { "github": "https://github.com/owner/repo" },
    "autoupdate": {
        "url": "https://github.com/owner/repo/releases/download/v$version/plugin-$version-win64.zip",
        "hash": { "mode": "download" }
    }
}
```

### 场景 5: 需要运行时缓存重定向 (Junction)

```json
"pre_install": [
    "function Create-Junction { param ([string]$runtimeCache, [string]$runtimeCachePersist)",
    "  if (-not (Test-Path $runtimeCache)) { return }",
    "  if (Test-Path $runtimeCachePersist) {",
    "    Remove-Item $runtimeCache -Force -Recurse -ErrorAction SilentlyContinue",
    "    New-Item -Type Junction -Path $runtimeCache -Target $runtimeCachePersist | Out-Null",
    "  } else {",
        "    mkdir $runtimeCache -ErrorAction SilentlyContinue",
    "    Move-Item $runtimeCache $runtimeCachePersist -Force",
    "    New-Item -Type Junction -Path $runtimeCache -Target $runtimeCachePersist | Out-Null",
    "  }",
    "}",
    "foreach ($folder in @('AppDataFolder')) {",
    "  Create-Junction -runtimeCache \"$env:APPDATA\\$folder\" -runtimeCachePersist \"$persist_dir\\APPDATA\\$folder\"",
    "}"
],
"persist": "APPDATA"
```

### 场景 6: MSI 安装 (需管理员)

```json
{
    "architecture": {
        "64bit": {
            "url": "https://example.com/app.msi#/setup.msi_",
            "hash": "sha256hash..."
        }
    },
    "pre_install": [
        "Rename-Item $dir\\setup.msi_ app.msi -ErrorVariable LockError -ErrorAction Stop",
        "sudo msiexec /i $dir\\app.msi /qn"
    ],
    "uninstaller": {
        "script": [
            "sudo Start-Process 'msiexec' -Wait -Verb 'RunAs' -WindowStyle 'Hidden' -ArgumentList @('/x', \"$dir\\app.msi\", '/qn')"
        ]
    },
    "depends": "sudo"
}
```

### 场景 7: PowerShell 模块

```json
{
    "architecture": {
        "64bit": {
            "url": "https://psg-prod-eastus.azureedge.net/packages/module.version.nupkg",
            "hash": "sha256hash..."
        }
    },
    "pre_install": "Remove-Item \"$dir\\_rels\", \"$dir\\package\", \"$dir\\*Content*.xml\" -Recurse",
    "psmodule": { "name": "ModuleName" },
    "checkver": {
        "url": "https://www.powershellgallery.com/packages/ModuleName",
        "regex": "<h2>([\\d.]+)</h2>"
    },
    "autoupdate": {
        "url": "https://psg-prod-eastus.azureedge.net/packages/module.$version.nupkg",
        "hash": { "mode": "download" }
    }
}
```

### 场景 8: 单 exe / 无架构区分 (GUI 程序用 shortcuts)

```json
{
    "version": "1.2.0",
    "description": "An IThumbnailProvider for Windows explorer that uses FFmpeg to generate thumbnails for various video files.",
    "homepage": "https://github.com/megakraken/FFmpegThumbnails",
    "license": "GPL-2.0",
    "suggest": {
        "FFmpeg": ["ffmpeg"]
    },
    "architecture": {
        "64bit": {
            "url": "https://github.com/megakraken/FFmpegThumbnails/releases/download/v1.2.0/FFmpegThumbnails.exe",
            "hash": "1f147917335ba0fe2f1a43a44e977766cf32abe2f5f7664ae26ec09e17a4cc22"
        }
    },
    "shortcuts": [
        ["FFmpegThumbnails.exe", "FFmpegThumbnails"]
    ],
    "checkver": {
        "github": "https://github.com/megakraken/FFmpegThumbnails"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/megakraken/FFmpegThumbnails/releases/download/v$version/FFmpegThumbnails.exe",
                "hash": {
                    "mode": "extract",
                    "url": "https://github.com/megakraken/FFmpegThumbnails/releases/expanded_assets/v$version",
                    "regex": "FFmpegThumbnails\\.exe.*?sha256:([a-f0-9]{64})"
                }
            }
        }
    }
}
```

---

## 常用 PowerShell 脚本片段

### 清理解压后的多余文件

```powershell
# 删除 $PLUGINSDIR (NSIS 安装包解压后)
Remove-Item -Recurse -Force "$dir\`$PLUGINSDIR"

# 删除所有带 $* 的特殊目录
Remove-Item -Recurse -Force "$dir\`$*"

# 删除 nupkg 元数据
Remove-Item "$dir\_rels", "$dir\package", "$dir\*Content*.xml" -Recurse
```

### 检查并创建持久化目录

```powershell
if (!(Test-Path "$persist_dir\data")) {
    New-Item "$persist_dir\data" -Type Directory -Force | Out-Null
}
```

### 复制已有配置到持久化目录

```powershell
if (Test-Path "$env:USERPROFILE\.config\app") {
    Copy-Item -Path "$env:USERPROFILE\.config\app\*" -Destination "$persist_dir\data" -Recurse
}
```

---

## 命名规范

- Manifest 文件名: `bucket/{app-name}.json`
- 使用小写字母 + 连字符 (`-`), 不包含空格
- 优先使用 GitHub 仓库名作为文件名 (如 `bili-music.json`)
- 如果软件原名有大写/特殊字符, 保留常见命名 (如 `Clash-for-Windows_Chinese.json`, `Edge-Webdriver.json`)
- 参考现有 manifest 命名风格

---

## 遇到问题时的处理流程

1. **无法确定下载链接**: 访问 Releases 页面, 查看所有 Assets
2. **哈希值未提供**: 用户下载后提供, 或使用 `hash`: `{ "mode": "download" }` 让 Scoop 自动计算
3. **解压后找不到 exe**: 解压完整目录树, 确认 exe 位置, 可能需要 `extract_dir`
4. **安装包无法直接解压**: 尝试 `#/dl.7z` 重命名, 或 `innosetup: true`, 或 `Expand-7zipArchive` 手动解包
5. **版本号提取失败**: 查看 GitHub Releases 页面 tag 格式, 看是否有 `v` 前缀、`-rc` 等
6. **需要用户协助**: 记录问题, 请求用户提供更多信息

---

## 本地测试流程 (推荐每次修改后执行)

```powershell
# 1. 验证 JSON 语法
python -c "import json; json.load(open('bucket/xxx.json',encoding='utf-8')); print('JSON OK')"

# 2. checkver 测试 (强制更新, 验证 autoupdate.hash.extract)
cd C:\Users\weidows\scoop\buckets\3rd
.\bin\checkver.ps1 <AppName> -Force

# 3. 预期输出应包含:
#    Searching hash for xxx.exe in https://github.com/.../expanded_assets/vX.Y.Z
#    Found: <sha256> using Extract Mode
#    Writing updated xxx manifest
```

---

## 参考链接

- [Scoop Wiki: Manifests](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests)
- [Scoop Wiki: Creating an App Manifest](https://github.com/ScoopInstaller/Scoop/wiki/Creating-an-app-manifest)
- [Scoop Wiki: Autoupdate](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifest-Autoupdate)
- [Scoop Wiki: Pre/Post Scripts](https://github.com/ScoopInstaller/Scoop/wiki/Pre-Post-(un)install-scripts)
- [Scoop Wiki: Persistent Data](https://github.com/ScoopInstaller/Scoop/wiki/Persistent-data)
- [本仓库 Manifest 示例](bucket/) (204 个参考)