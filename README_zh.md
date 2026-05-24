# stock-research-skill

[English](./README.md) | [简体中文](./README_zh.md)

![License](https://img.shields.io/github/license/kevinkda/stock-research-skill)
![Translation](https://img.shields.io/badge/i18n-EN%20%2B%20zh--CN-blue)
![Skills](https://img.shields.io/badge/skills-2-blue)
![Release](https://img.shields.io/github/v/release/kevinkda/stock-research-skill)
![Releases](https://img.shields.io/github/release-date/kevinkda/stock-research-skill?label=last%20release)

一个 Cursor / Claude Code **skill 包**，将 **3 个 MCP server** 编排成多步
研究 playbook —— [`schwab-marketdata-mcp`](https://github.com/kevinkda/schwab-marketdata-mcp)、
[`sec-edgar-mcp`](https://github.com/kevinkda/sec-edgar-mcp)、
[`polygon-news-mcp`](https://github.com/kevinkda/polygon-news-mcp) ——
服务 [`kevinkda/stock-personal`](https://github.com/kevinkda/stock-personal)
投研工作流。

本 skill 是**只读文档**；真正的 API 流量由上述 3 个 MCP server 各自承担。

---

## 总览

本仓库提供**一个跨 MCP skill**，含两个语言变体（中文为主 + 英文镜像），
便于 agent 根据用户语言切换：

- **`stock-research`**（中文主版）—— 多步 playbook，把 schwab 的 shakeout
  信号 + polygon 的新闻情绪 + sec-edgar 的 Form 4 申报数据交叉关联。
- **`stock-research-en`**（英文镜像）—— scope 与中文版一致，英文行文。

两个变体共享：

- 同一套 activation handshake（3 个 `health_check` + git/gh 私有仓闸门）。
- 同一套治理规则（仅写私有仓；从不直接 commit 到 main；从不 force push；
  按 MCP server 版本范围自动 gating）。
- 仅 `language_directive` frontmatter 字段不同。

---

## Skill 变体

| Skill | 语言 | 何时使用 |
| --- | --- | --- |
| [`stock-research`](stock-research/SKILL.md) | 简体中文（主版） | 跨 MCP 投研 playbook（`shakeout-with-news`、`insider-alert`）。 |
| [`stock-research-en`](stock-research-en/SKILL.md) | English（镜像） | scope 与 `stock-research` 相同，英文行文。 |

> 触发本 skill 的关键词：`shakeout 配新闻` / `扫 watchlist 内部人` /
> `shakeout-with-news` / `insider-alert`。
> 如果用户只需调用单一 MCP server，请改用对应单库 skill
> （[`schwab-marketdata-skill`](https://github.com/kevinkda/schwab-marketdata-skill)、
> 未来的 `sec-edgar-skill` / `polygon-news-skill`）。

---

## 与 MCP server 的兼容性

| 本 skill 仓 | 兼容的 MCP server |
| --- | --- |
| `v0.1.x` | `schwab-marketdata-mcp >=0.3,<0.4` + `sec-edgar-mcp >=0.2,<0.3` + `polygon-news-mcp >=0.2,<0.3` |

版本范围编码在每份 `SKILL.md` 的 `mcp_dependencies` frontmatter。
Activation handshake 会逐一调用 `health_check()`，任一 `server_version`
落到范围外即拒绝继续。

---

## 安装

具体机制视客户端版本而定。常见布局：

### Cursor

Cursor 扫描 `~/.cursor/skills/` 加 Settings → Skills 中用户添加的目录。
symlink（或复制）你想要的文件夹——可装中文主版 / 英文镜像 / 两者都装：

```bash
# 中文主版（本仓默认）
ln -s "$(pwd)/stock-research"     ~/.cursor/skills/stock-research

# 英文镜像（可选；可附加或替代主版）
ln -s "$(pwd)/stock-research-en"  ~/.cursor/skills/stock-research-en
```

### Claude Code

Claude Code 读 `~/.claude/skills/<name>/SKILL.md`，symlink 同样适用：

```bash
ln -s "$(pwd)/stock-research"     ~/.claude/skills/stock-research
ln -s "$(pwd)/stock-research-en"  ~/.claude/skills/stock-research-en
```

> **前置条件**：3 个 MCP server 都已注册到客户端。每个 server 的注册步骤
> 见各自仓的 `docs/REGISTER.md`：
>
> - [schwab-marketdata-mcp/docs/REGISTER.md](https://github.com/kevinkda/schwab-marketdata-mcp/blob/main/docs/REGISTER.md)
> - [sec-edgar-mcp/docs/REGISTER.md](https://github.com/kevinkda/sec-edgar-mcp/blob/main/docs/REGISTER.md)
> - [polygon-news-mcp/docs/REGISTER.md](https://github.com/kevinkda/polygon-news-mcp/blob/main/docs/REGISTER.md)

本仓的 [`docs/REGISTER.md`](docs/REGISTER.md) 提供组合激活清单。

---

## License

MIT License —— 见 [LICENSE](./LICENSE)。

---

## Acknowledgements

本 skill 包是以下 3 个 MCP server 的跨库配套：

- **[schwab-marketdata-mcp](https://github.com/kevinkda/schwab-marketdata-mcp)**
  —— 只读 Schwab Market Data MCP server（12 个工具、OAuth、DuckDB 缓存）。
- **[sec-edgar-mcp](https://github.com/kevinkda/sec-edgar-mcp)** ——
  只读 SEC EDGAR MCP server（filings / Form 4 / Form 13F XBRL）。
- **[polygon-news-mcp](https://github.com/kevinkda/polygon-news-mcp)** ——
  只读 Polygon 新闻 MCP server（ticker_news、sentiment_aggregate）。

skill markdown 设计模式参考 [`schwab-marketdata-skill`](https://github.com/kevinkda/schwab-marketdata-skill)
配套仓和 Anthropic 公开的 skill pack。

本项目**与 Charles Schwab Corporation、美国 SEC、Polygon.io 无任何关联或
背书**。每个 MCP server 各自遵守其上游服务条款。Schwab Market Data
不可二次分发；所有衍生的 markdown 仅留在私有
`kevinkda/stock-personal` 仓内。

---

## 相关链接

- [stock-personal](https://github.com/kevinkda/stock-personal) ——
  私有投研日记仓；本 skill 仅写入其 `research/` 目录。
- [schwab-marketdata-skill](https://github.com/kevinkda/schwab-marketdata-skill)
  —— 单 MCP 的 playbook 集（`shakeout-analysis-v2`、`voo-qqq-tracker`、
  `watchlist-snapshot`、`summary-md-refresh`、`option-chain-research`）。
