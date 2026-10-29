# hermes-bundle

把 [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) 的**稳定版**打成
可移植 payload（绿色、解压即用、换目录也能跑），挂到本仓库 Releases。

- 不含任何源码改动，也不是 fork —— 构建完全在 GitHub Actions 云端完成
- 当前支持：**Windows（win32-x64）**
- 规划中：macOS、Linux

## 目录

```
.github/workflows/bundle-windows.yml   # 唯一的构建流程
UPSTREAM_TAG.win32-x64                 # 由 CI 维护，记录本平台已构建的上游 tag
```

## 一次性配置

1. 仓库保持 **public**（Actions 对 public 仓库免费无限，Windows runner 也不计费）。
2. Settings → Actions → General → Workflow permissions 选 **Read and write**。
   GitHub 对新仓库的默认值是 **read-only**，此时 `gh release create` 会 403。
   workflow 里的探测步骤只打印警告、不拦截 —— `GITHUB_TOKEN` 是 installation token，
   `GET /repos` 返回的 `permissions.push` 对它恒为 false，不能拿来当判据。
3. 如果 `main` 有分支保护，放行 `github-actions[bot]` 的推送；否则最后一步失败 ——
   不影响发版，只是下一轮会重复构建一次。

## 触发

三种方式都**不需要在网页上登录**：

| 方式 | 说明 |
|---|---|
| `schedule` | 每 6 小时查一次上游 |
| `push` 到 `main` | 改 `.github/` 下的文件即可（`.md` 与 `UPSTREAM_TAG.*` 已被忽略，不会触发） |
| `workflow_dispatch` | 网页点 Run workflow，或带 token 打 API；勾 `force` 强制重建 |

上游**每天**都会发 `v0.22.5+canary.YYYYMMDDThhmmssZ` 这类预发布。流程只看
`/releases/latest` 302 指向的稳定版，canary 不会触发构建。

跳过条件（任一成立即不构建）：`UPSTREAM_TAG.win32-x64` 已等于上游 tag，
或同名 Release 里已有 `*-win32-x64.7z.001` 资产。

> 改了 workflow 想立刻验证时，`push` 会因 `UPSTREAM_TAG` 已标记而秒跳过；
> 用 `workflow_dispatch` 带 `force: true` 才是真重建。

## 构建链路

`pm bundle` 的输入是一份**已安装**的 Hermes，所以是五步：

```
1. install.ps1 -Channel stable -NonInteractive   → 装好的基准（= 上游最新稳定版）
2. hermes --version（失败则 hermes pm repair）    → 确认基准可用、拿到版本号
3. hermes pm bundle --out <dir> --cache <dir>    → 可移植 payload
4. write_install_stamp.py                        → 补 install-stamp.json
5. 7z a -mx=5 -v1900m + gh release upload        → 分卷发布
```

`install.ps1` 只有 `-Channel stable|main`、没有 `-Tag`，版本由上游 stable 通道决定；
构建后用 `hermes --version` 复核，不一致就只发版不写 `UPSTREAM_TAG.*`，下一轮自动重试。

## 产物（v0.22.5 实测）

| 项 | 值 |
|---|---|
| payload 未压缩 | **5,822 MB**（`tools` + `venv` + `uv-cache` + `hermes-agent`） |
| 压缩后 | **1,484 MB**（lzma2 压缩率 25.5%） |
| 分卷 | **1 卷**（1900 MiB 上限，实际远低于；payload 变大才会自动分成 2 卷） |
| 耗时 | **26 分钟**（install 6 + bundle 10 + 压缩 5 + 发布 5） |

payload 自带启动器是**相对路径**的，换盘符、换目录都能跑；`HERMES_HOME` 由使用者另外指定。

## 下载后怎么用

1. 下载全部分卷（`.7z.001`、`.7z.002` …）和 `SHA256SUMS.txt`
2. 校验：`Get-FileHash *.7z.001` 与 `SHA256SUMS.txt` 比对
3. 解压：`7z x hermes-payload-vX.Y.Z-win32-x64.7z.001` —— 7-Zip 会自动串联所有分卷，**不需要**先合并
4. 解出 `payload/`，用 `payload\bin\hermes.exe` 启动

## 构建原理与已知坑

### install.ps1 装出来的 launcher 必须先 `pm repair`

`install.ps1` 用 **bootstrap python** 跑 `pm install`。按 `pm/environments.py` 的规则，
**非 selected 解释器的 lazy sync 不写 selection** —— 环境目录建了
（`installs/<hash>/environments/<env-hash>/`），但没 commit。launcher 又跑在 store python 上，
于是 `_require_own_dependencies()` 直接拒绝启动：

```
hermes: no dependency environment is committed for this install; run `hermes pm repair`
```

源码里 `NoCommittedEnvironment` 的 docstring 写得很直白：
**「no retry fixes it, only `hermes pm repair`」**。所以流程里必须先试 `--version`，
失败就 `pm repair` 再试一次。CI 上这一步几秒就过（网络直连）；本机走代理会因
`uv sync` 超时 1800s 而失败，属环境问题。

### `pm bundle` 不写 `install-stamp.json`

缺这个文件时 payload 的 `--version` 报 **`vunknown`**，`payload: bundled` /
`updateMechanism: external` 等字段全部失效。而 **payload 里也没有 `scripts/`**
（`git archive` 排掉了），所以必须用 install 那份 checkout 里的脚本、配 payload 自己的 python：

```powershell
$env:HERMES_DESKTOP_VARIANT = 'bundled'
$env:HERMES_PAYLOAD_TAG     = 'vX.Y.Z'          # bundled 必须有合法 tag，否则脚本拒绝
cd <install>\hermes-agent
<payload>\tools\python-...\python.exe scripts\write_install_stamp.py `
    --output <payload>\hermes-agent\install-stamp.json `
    --commit <install HEAD> --base-version X.Y.Z --distance 0 `
    --source bundle --distribution desktop-app --update-mechanism external `
    --runtime-dir ../tools
```

产出 `displayVersion` / `payload` / `distribution` / `updateMechanism` / `runtimeDir` / `tag`。
workflow 生成后会立刻用 payload 自带的 launcher 跑 `--version` 断言 `vX.Y.Z`，不通就失败。

### 16 个 pinned 工具一个都裁不掉

`pm/lock.json` 里 win32-x64 的非 internal 包共 16 个：

```
agent-browser  bws  chromium  cua-driver  ffmpeg  gh  git
llamacpp-cpu   llamacpp-cuda  llamacpp-hip  llamacpp-vulkan
node           npm  python    ripgrep      uv
```

四个 `llamacpp-*`（CPU/CUDA/HIP/Vulkan 变体）通常用不上，但 `pm bundle` **没有排除参数**。
要裁只能构建后删目录并同步改 `tools/facts.json`、`manifest.json` 与 `pm/lock.json`，
代价是 `pm doctor` 报 `✗`。

### all-extras 需要 MSVC（+ Rust 链接器）

`pilk`（extra `silk`）要 C 编译器，`obstore`（extra `daytona`）要 maturin 链接。
`windows-latest` 自带 VS 2022 Build Tools 与 Rust，所以 CI 不用额外准备；
本机构建则需要 `vcvars64.bat` 与构建在同一 cmd 进程里。

### 验收清单

构建完至少跑这三条，缺一条都可能交付一个「看着正常但起不来」的包：

| 命令 | 期望 |
|---|---|
| `bin\hermes.exe --version` | `Hermes Agent vX.Y.Z · stable`（不是 `vunknown`） |
| `bin\hermes.exe pm doctor` | `exit=0`，16 个工具全 `✓` |
| 目录改名后再跑上面两条 | 仍 `exit=0`（launcher 走相对推算，证明可搬） |

## 加 macOS / Linux 时

每个平台一份独立 workflow（`bundle-macos.yml` / `bundle-linux.yml`），共用同一个 release tag，
靠资产名里的平台后缀区分（`-darwin-arm64` / `-linux-x64`）。

两条硬约束：

- 跳过判据必须**按平台看资产**（如 `*-darwin-arm64.tar.zst.001`），不能只看「release 是否存在」，
  否则后加的平台永远不会构建。
- 平台的构建入口不同：`install.ps1` → `install.sh`，`7z` → `tar.zst` + 分卷，
  `runs-on` 换成 `macos-14` / `ubuntu-24.04`。

## 已知限制

- 磁盘全部走 `C:`：`windows-latest` 的 `D:` 只有约 14 GB，而 runner 的 `C:` 实测只剩约 30 GB，
  6 GB payload + 2.5 GB 分卷 + 2.25 GB 安装已经吃掉大半。
- 全量包含 `uv-cache/`（约 1.1 GB），只为「插件依赖离线重建」；删掉能显著减小体积但 `pm doctor` 会缺项。
- 内含 git(GPLv2)、ffmpeg(GPL/LGPL)、chromium(BSD) 等第三方二进制，许可随各自发行说明；Hermes 本体为 MIT。
- payload 是只读快照，更新方式是换整包（`hermes update` 会拒绝：`This Hermes runs from inside the desktop app bundle.`）。
