# AGENTS.md

面向在本仓库工作的 AI coding agent。

这是一个 **Scoop bucket**：Windows 包管理器 [Scoop](https://scoop.sh) 的软件源。每个 `bucket/*.json` 描述一个应用，`scoop install <app>` 直接消费它。

## 仓库结构

| 路径 | 作用 |
|---|---|
| `bucket/<app>.json` | 应用清单。**文件名即应用名**，也是 `scoop install <app>` 使用的名字 |
| `README.md` | 软件总表。**版本列由 CI 自动同步，不要手动维护** |
| `bin/test.ps1` | 本地与 CI 的测试入口（Pester），退出码 = 失败用例数 |
| `Scoop-Bucket.Tests.ps1` | 加载 Scoop 官方 bucket 测试套件 |
| `.github/workflows/ci.yml` | push / PR 时在 WindowsPowerShell 与 PowerShell 两个 job 跑 `bin/test.ps1` |
| `.github/workflows/excavator.yml` | 每 15 分钟自动检测上游新版本并提交 |
| `.github/workflows/update-readme.yml` | 每 15 分钟把 manifest 版本写回 README |
| `.gitattributes` | `* text=auto eol=crlf` |

## 硬性规则（CI 强制，违反必红）

CI 会跑 `Scoop-00File.Tests.ps1`（文件卫生）与官方 schema 校验：

1. **换行符必须是 CRLF**，且文件必须以换行结尾。
2. **不得有 UTF-8 BOM**。
3. **不得有行尾空白**；缩进只能用空格，不能用 TAB。
4. **必须通过 Scoop 官方 schema**（`$SCOOP_HOME/schema.json`）。
5. 缩进统一 **4 个空格**（现有 22 个清单全部如此）。

## 新增一个应用

1. **选对发布资产**：优先 portable ZIP。若上游只提供会装到 `%LOCALAPPDATA%` 的安装器（Inno Setup / NSIS），它会绕过 Scoop 的版本管理，应改用 ZIP；只有上游确实只发 portable `.exe` 时，才配合 `installer` + `post_install` 用 7z 解包（见 `dlss5-swapper.json`、`open-design.json`）。
2. **下载并计算 SHA256**：
   ```powershell
   Invoke-WebRequest '<asset-url>' -OutFile "$env:TEMP\a.zip"
   (Get-FileHash "$env:TEMP\a.zip" -Algorithm SHA256).Hash.ToLower()
   ```
   若上游同时发布了校验清单（如 `manifest.json`、`SHA256SUMS`），**交叉核对一致**后再写入。
3. **写 `bucket/<app>.json`**（模板见下）。
4. **在 `README.md` 表格追加一行**（格式见下）。
5. **跑验证**（见下），确认退出码为 0。
6. 提交。

### 清单模板

```json
{
    "version": "1.2.3",
    "description": "One-line English summary of the app",
    "homepage": "https://example.com",
    "license": "MIT",
    "architecture": {
        "64bit": {
            "url": "https://github.com/<owner>/<repo>/releases/download/v1.2.3/<asset>-1.2.3-windows-x86_64.zip",
            "hash": "<sha256>"
        }
    },
    "bin": "<app>.exe",
    "shortcuts": [
        [
            "<app>.exe",
            "<Display Name>"
        ]
    ],
    "checkver": {
        "github": "https://github.com/<owner>/<repo>"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/<owner>/<repo>/releases/download/v$version/<asset>-$version-windows-x86_64.zip",
                "hash": {
                    "mode": "download"
                }
            }
        }
    }
}
```

### 字段顺序（沿用现有惯例）

`version` → `description` → `homepage` → `license` → `notes` → `suggest` / `env_set` → `architecture` → `pre_install` / `installer` / `post_install` → `bin` → `shortcuts` → `checkver` → `autoupdate`

### 字段用法

| 字段 | 惯例 |
|---|---|
| `checkver` | 应用名与 GitHub 仓库同名时用简写 `"checkver": "github"`（现有 9 例）；否则用 `{"github": "https://github.com/owner/repo"}`（现有 13 例）；非 GitHub 或 tag 格式特殊时用 `{"regex": "..."}` |
| `autoupdate.hash` | 默认 `{"mode": "download"}`。上游提供校验文件时改用 `{"url": "$url.sha256"}`（见 `opencodex.json`）或 `{"url": "$baseurl/SHA256SUMS", "regex": "$sha256\\s+$basename"}`（见 `magpie-ai.json`） |
| `architecture` | 默认只写 `64bit`；上游确有 arm64 资产才加 `arm64`（现有 11 例双架构） |
| `bin` | 单命令用字符串；多命令用数组；需要改命令名时用 `[["path.exe", "alias"]]` |
| `shortcuts` | GUI 应用必须写，否则开始菜单没有入口 |
| `notes` | 字符串数组，写用户需要知道的事项 |
| `env_set` | 需要给应用注入环境变量时用（见 `opencodex.json`） |
| `suggest` | 推荐依赖，如 `{".NET Runtime": "extras/windowsdesktop-runtime"}` |

## README 表格

`update-readme.yml` 用下面这个正则逐行匹配，因此**格式必须严格一致**：

```
^\| (\d+) \| \[([^\]]+)\]\(([^)]+)\) \| (.+?) \| (.+?) \|
```

对应写法：

```
| 22 | [zeron](https://github.com/zeronsh/zeron) | AI 编码代理控制平面（桌面 + CLI） | 🤖 AI 工具 | 0.2.102 |
```

- 序号递增，不复用已删除的编号。
- **方括号里必须是 manifest 文件名**（CI 靠它定位 `bucket/<name>.json` 取版本）。
- 说明与类别里**不能出现 `|`**。
- 版本列由 CI 重写，写错会被自动纠正，但仍应写对。
- 现有类别取值：`🤖 AI 工具`、`🛠 系统工具`、`🌐 网络工具`、`🎨 开发工具`、`🔧 开发工具`、`🖥 终端工具`、`🗄 数据库工具`、`✍️ 文本编辑`、`🎮 游戏工具`、`🎵 音乐播放`。

## 验证（提交前必做）

```powershell
$env:SCOOP_HOME = "$(scoop prefix scoop)"

# 1) 完整 CI 测试（推荐）。退出码 = 失败用例数，必须为 0
.\bin\test.ps1

# 2) 只校验 schema（无需 Pester，秒级完成）
Add-Type -Path "$env:SCOOP_HOME\supporting\validator\bin\Scoop.Validator.dll"
$v = New-Object Scoop.Validator("$env:SCOOP_HOME/schema.json", $true)
$v.Validate("bucket\<app>.json"); $v.Errors.Count   # 必须输出 0

# 3) 检查 checkver / autoupdate 能否正确解析
& "$env:SCOOP_HOME\bin\checkver.ps1" <app> bucket

# 4) 真实安装验证（最有说服力：校验 hash、解压、建 shim 与快捷方式）
scoop install <app>
scoop uninstall <app>
```

> 第 1 步需要 **Pester ≥ 5.2.0** 与 **BuildHelpers ≥ 2.0.1**；系统自带的 Pester 3.4.0 不满足要求：
> ```powershell
> Install-Module Pester -RequiredVersion 5.2.0 -Scope CurrentUser -Force
> Install-Module BuildHelpers -RequiredVersion 2.0.1 -Scope CurrentUser -Force
> ```

> **本地换行符陷阱**：`bin/test.ps1` 直接读工作区文件。若编辑器把某个 JSON 存成了 LF，本地会报 `file newlines are CRLF` 失败，而 CI 不会（checkout 时按 `.gitattributes` 转成 CRLF）。遇到这种不一致，先用 `git ls-files --eol bucket/*.json` 确认，不要急着改内容。

## 提交信息

沿用 `<manifest 名>: <说明>` 的格式：

| 场景 | 格式 | 例 |
|---|---|---|
| 新增应用 | `<app>: Add version <x.y.z>` | `zeron: Add version 0.2.102` |
| 版本更新（Excavator 自动） | `<app>: Update to version <x.y.z>` | `magpie-ai: Update to version 0.1.694` |
| README 同步（CI 自动） | `docs: sync versions in README` | — |
| 其他改动 | `<app>: <简短说明>` 或 `chore:` / `ci:` / `docs:` | `opencode-v2: publish the v2 CLI as 'opencode' instead of 'opencode2'` |

## 自动化：不要与之冲突

- **Excavator** 每 15 分钟扫描上游并直接提交版本与 hash 更新；manifest 被自动改动属正常现象。
- **update-readme.yml** 每 15 分钟重写 README 版本列。
- 因此：**不要手动维护 README 版本列**，也不要让过期的 manifest 改动长期滞留本地。

## 陷阱

- **`${version}` 会让 schema 校验失败**：`autoupdate` 的 URL 里必须用 `$version`。`${version}` 含 `{` / `}`，不满足 URI 格式。
- **文件名必须与 README 方括号内名称完全一致**，否则 CI 取不到版本，表格会显示 `?`。
- **新建文件务必 CRLF 且以换行结尾**，见上文硬性规则。
- **`checkver` 走 GitHub API**，未认证时容易限流；批量检查建议配置 `GITHUB_TOKEN`。

