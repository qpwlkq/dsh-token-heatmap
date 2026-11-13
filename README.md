# dsh-token-heatmap

把 DeepSeek Harness 的 AI token 用量画成 GitHub 同款贡献热力图，输出可直接嵌进 README 的 SVG。

```
┌──────────────────────────────────────────────┐
│ 8.0M AI tokens in the last year              │
│  Nov Dec Jan ...                          Nov│
│ Mon ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│ Wed ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▒▒░░░░░░  │
│ Fri ░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓░░░▒▒░░░░░░  │
│ 7 active days · peak 6.3M/day                │
└──────────────────────────────────────────────┘
```

## 为什么不是点亮 GitHub 原生格子

GitHub 的贡献图只统计 commit / issue / PR / review / discussion，没有写入自定义数值的 API，颜色还是相对你自己峰值的 5 档分位。所以 token 量无法精确映射进去。这个工具走另一条路：**自己渲染 SVG，放进 README**——数值精确、色阶可控、和原生格子零冲突。

## 快速开始

需要 Node ≥ 22.19（用到内置 zstd）。

```sh
cd token-heatmap

# 看一眼统计，不写文件
node src/cli.js stats

# 生成 token-usage.json + 亮/暗两张 SVG
node src/cli.js render --out dist
open dist/token-heatmap.svg
```

`stats` 输出示例：

```
home      /Users/you/Library/Application Support/dsh-desktop/harness
time zone Asia/Shanghai
window    2025-11-09 -> 2026-11-12 (53 weeks)
logs      16 files, 16 sessions
tokens    7,955,959 (8.0M)  in 355K  out 181K  cache read 7.4M  cache write 0
requests  174
active    7 days, peak 6.3M/day
```

### 找哪个 home

按顺序取第一个可用的：

1. `--home <dir>`
2. `$DSH_HOME`（桌面版启动的进程里有）
3. `~/Library/Application Support/dsh-desktop/harness`（macOS 桌面版）
4. `~/.dsh`

## 装进 DSH

插件只做一件事：注册 `/token-heatmap` 命令，把结果写到 `$DSH_HOME/token-heatmap/`。

在 profile 的用户 patch 层（`$DSH_HOME/profiles/<name>/cordis.patch.yml`）里加一条指向绝对路径：

```yaml
- insert:
    - id: token-heatmap
      name: '/absolute/path/to/token-heatmap/src/plugin.js'
```

保存后 profile 会热加载（`patchReload: live`），输入 `/token-heatmap` 即可。

也可以按 bundle 安装（需要源码仓库的 `dsh` CLI）：

```sh
dsh plugin --profile web add /absolute/path/to/token-heatmap
```

插件没有任何配置项；要换输出目录、时区等，用 CLI。

## 接到 GitHub 个人主页

1. 建 `<你的用户名>/<你的用户名>` 这个 profile 仓库。
2. 把 `token-heatmap/` 里的文件复制到仓库根目录（`src/`、`scripts/`、`package.json` 等）。
3. 跑一次 `node src/cli.js render --out docs`。
4. profile 仓库的 `README.md` 里放：

```html
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./docs/token-heatmap-dark.svg">
  <img alt="AI token usage in the last year" src="./docs/token-heatmap.svg">
</picture>
```

GitHub 会通过 camo 代理并缓存这张图，所以**更新有延迟**（通常几分钟到几小时）。要立刻看到变化，在引用地址后加一个每次渲染都变的查询参数，例如 `<img src="./docs/token-heatmap.svg?v=2026-11-12">`。

## 每日自动同步

`scripts/sync.sh` 会渲染、提交、推送：

```sh
chmod +x scripts/sync.sh
scripts/sync.sh
```

macOS 用 launchd（`~/Library/LaunchAgents/com.you.token-heatmap.plist`）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.you.token-heatmap</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-lc</string>
    <string>/absolute/path/to/repo/scripts/sync.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>0</integer><key>Minute</key><integer>10</integer></dict>
  <key>StandardOutPath</key><string>/tmp/token-heatmap.log</string>
  <key>StandardErrorPath</key><string>/tmp/token-heatmap.err</string>
</dict>
</plist>
```

```sh
launchctl load ~/Library/LaunchAgents/com.you.token-heatmap.plist
```

或者 crontab（注意用 node 的绝对路径）：

```cron
10 0 * * * /usr/local/bin/node /absolute/path/to/repo/src/cli.js render --out /absolute/path/to/repo/docs && cd /absolute/path/to/repo && git add docs && git commit -m "chore: token heatmap" && git push
```

`.github/workflows/render.yml` 是可选的：它只在 `docs/token-usage.json` 变化时重新渲染 SVG，不采集数据（采集必须在有 session 日志的机器上跑）。

## 数据口径

- **总量** = 未缓存输入 + 输出 + 缓存读 + 缓存写，四个桶分别保留在 `token-usage.json` 里。
- **只算 provider 上报的 usage**：`assistant/message` 的 `usage`，以及 `assistant/chunk` 的 usage 块。compaction 直接调用 `ctx.llm.stream()`，不写 usage，所以那部分 token 不计入。
- **一次请求只算一次**：同一 `(turn, step)` 先有 usage chunk、后有最终 message 时，后者替换前者，和 `dsh-token-meter` 的 `tokenUsage` 投影一致；跨午夜替换会把 token 挪到后一天。
- **按本地时区归日**：默认用系统时区，`--tz Asia/Shanghai` 可覆盖。
- **色阶**：非零日的 25/50/75 分位，四档；不是绝对阈值，所以换机器、换用量水平都不会全灰。
- **窗口**：固定 53 列（和 GitHub 一致），最后一天默认是今天。

## 隐私

工具只读事件里的 `time` 和 token 数字，不读 prompt、不读 tool 内容。导出的 `token-usage.json` 只含日期、计数、时区和标题，不含 session id、路径或任何消息文本。提交到公开仓库前可以自己看一眼这个文件。

## 目录结构

```
token-heatmap/
  src/
    zstd.js      多帧 zstd 解码（Node 单次 API 只解第一帧）
    sessions.js  发现 session.v*.jsonl.zstd，按最高版本取一份
    usage.js     usage 事件 -> 按天聚合
    heatmap.js   53 周网格、月份标签、分位色阶
    svg.js       SVG 渲染（亮/暗两套）
    render.js    采集 + 写文件的编排
    cli.js       命令行入口
    plugin.js    DSH 插件：/token-heatmap 命令
  test/core.test.js
  scripts/sync.sh
  .github/workflows/render.yml
```

## 已知限制

- 只统计本机当前 home 的日志。多台机器、多个 home 要分别导出后再手工合并。
- 只有 `assistant/*` 上的 usage 会被统计；估算出来的 token（没有 provider usage 的请求）不计入。
- 分位色阶是相对的：如果你某天爆了 10M，其他天会被压到浅色。
- GitHub 对 README 里的 SVG 做消毒，脚本和交互事件不会执行；`<title>` 悬停提示在 `<img>` 里也不生效，所以图里只保留静态信息。
