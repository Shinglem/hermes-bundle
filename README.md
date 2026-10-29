# hermes-bundle

把 [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) 的**稳定版**打成
可移植 payload（绿色、解压即用、换目录也能跑），挂到本仓库 Releases。

- 不含任何源码改动，也不是 fork —— 构建完全在 GitHub Actions 云端完成
- 当前支持：**Windows（win32-x64）**
- 规划中：macOS、Linux

## 目录

```
.github/workflows/bundle-windows.yml   # Windows 构建流程
UPSTREAM_TAG.win32-x64                 # 由 CI 维护，记录本平台已构建的上游 tag
```

## 一次性配置

1. 仓库保持 **public**（Actions 对 public 仓库免费无限，Windows runner 也不计费）。
2. Settings → Actions → General → Workflow permissions 选 **Read and write**。
   GitHub 对新仓库的默认值是 **read-only**，此时 `gh release create` 会 403，发版无从谈起。
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

跳过条件（任一成立即不构建）：

- `UPSTREAM_TAG.win32-x64` 已等于上游 tag
- 同名 Release 里已有 `*-win32-x64.7z.001` 资产

## 构建链路

`pm bundle` 的输入是一份**已安装**的 Hermes，所以是两步：

```
官方 install.ps1 -Channel stable   → 一份装好的基准（= 上游最新稳定版）
hermes pm bundle --out <dir>       → 可移植 payload
7z a -mx=5 -v1900m                 → 分卷
gh release upload                  → 发布
```

`install.ps1` 只有 `-Channel stable|main`、没有 `-Tag`，版本由上游 stable 通道决定；
构建后用 `hermes --version` 复核，不一致就只发版不写 `UPSTREAM_TAG.*`，下一轮自动重试。

## 产物

| 项 | 值 |
|---|---|
| payload 未压缩 | 6,022 MB（`tools` 3,644 + `venv` 1,133 + `uv-cache` 1,109 + `hermes-agent` 138） |
| 压缩后 | 约 2.5 GB（lzma2，实测压缩率 23–37%；`uv-cache` 是已压缩 wheel，压不动） |
| 分卷 | 2 卷，每卷 1900 MiB（GitHub 单文件硬上限 2 GiB） |
| 耗时 | ≈ 70 min/次（install 15 + bundle 25 + 压缩 20 + 上传 5） |

payload 自带启动器是**相对路径**的，换盘符、换目录都能跑；`HERMES_HOME` 由使用者另外指定。

## 下载后怎么用

1. 下载**全部**分卷（`.7z.001`、`.7z.002` …）和 `SHA256SUMS.txt`
2. 校验：`Get-FileHash *.7z.001` 与 `SHA256SUMS.txt` 比对
3. 解压：`7z x hermes-payload-vX.Y.Z-win32-x64.7z.001` —— 7-Zip 会自动串联所有分卷，**不需要**先合并
4. 解出 `payload/`，用 `payload\bin\hermes.exe` 启动

## 加 macOS / Linux 时

每个平台一份独立 workflow（`bundle-macos.yml` / `bundle-linux.yml`），共用同一个 release tag，
靠资产名里的平台后缀区分（`-darwin-arm64` / `-linux-x64`）。

两条硬约束：

- 跳过判据必须**按平台看资产**（如 `*-darwin-arm64.tar.zst.001`），不能只看「release 是否存在」，
  否则后加的平台永远不会构建。
- 平台的构建入口不同：`install.ps1` → `install.sh`，`7z` → `tar.zst` + 分卷，
  `runs-on` 换成 `macos-14` / `ubuntu-24.04`。

## 已知限制

- 磁盘全部走 `C:`：`windows-latest` 的 `D:` 只有约 14 GB，装不下 2.25 GB 安装 + 6 GB payload + 2.5 GB 分卷。
- 全量包含 `uv-cache/`（约 1.1 GB，压缩后仍约 1.05 GB），只为「插件依赖离线重建」；
  删掉能把产物压到约 1.4 GB 单文件，但 `pm doctor` 会有缺失。
- 内含 git(GPLv2)、ffmpeg(GPL/LGPL)、chromium(BSD) 等第三方二进制，许可随各自发行说明；Hermes 本体为 MIT。
- payload 是只读快照，更新方式是换整包。
