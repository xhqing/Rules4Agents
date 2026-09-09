# Changelog

All notable changes to the AgentCortex project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### 变更（项目迁移收尾：CLAUDE.md 子项目清单路径更新）

- **为什么改**：本项目现址在 `~/Developer/`（`~/Documents/Projects/` 旧址已弃用，2026-09-08 迁移收尾时发现上级 NeuralCoreAgent 的子项目清单与本仓 `.claude/CLAUDE.md` 仍指旧路径），避免后续会话被引导到不存在的位置。
- **改了什么**：`.claude/CLAUDE.md` 中引用的自身路径由 `~/Documents/Projects/AgentCortex` 更新为 `~/Developer/AgentCortex`。

### 变更（Visitors 徽章更名 Visits/day (14d)：alt 文本与 xhqing 集中统计新 label 对齐）

- **为什么改**：用户要求（2026-08-17）访问量徽章名需表达「最近半月日均访问量」口径——xhqing 集中统计侧的 badge JSON label 已从 `Visitors` 改为 `Visits/day (14d)`（`Visits/day` 是 shields.io 表达日均的惯例写法、`(14d)` 标注 14 天滚动窗口），各仓 README 的徽章 alt 文本同步对齐，避免 alt 与徽章实际显示文字脱节。
- **改了什么**：README 徽章区 `alt="Visitors"` → `alt="Visits/day (14d)"`，仅改 alt 文本，endpoint URL、数据源、徽章口径均不变（口径改动记 xhqing 仓库 CHANGELOG，本仓只改 alt）。

### 变更（Visitors 徽章 alt 文本首字母大写：README 访问量徽章命名统一）

- **为什么改**：用户指令（2026-08-16）「Visitors 徽章全局统一，首字母大写」——配合全局 `~/.claude/CLAUDE.md`「徽章英文首字母必须大写」新规，集中统计上线时挂的访问量徽章 `alt="visitors"` 为小写存量，与 badge JSON label（`Visits/day`）及大写规范不一致，本次一次收口。
- **改了什么**：README（EN/CN）徽章区 visitors 徽章 `alt="visitors"` → `alt="Visitors"`，仅改 alt 显示文本，endpoint URL 与数据源不变。

### 新增（README 访问量徽章——舰队集中式访问统计）

- **为什么改**：全舰队上线集中式「真去重」访问统计（图片徽章方案无法去重，走官方 Traffic API 路线）：统计集中部署在 xhqing 仓库（`scripts/update_traffic.py` + 每日 GitHub Action），各 fleet 仓库只需在 README 挂徽章、零运行负担。
- **改了什么**：README（EN/CN）徽章区新增 visitors 徽章（shields.io endpoint 指向 `xhqing/xhqing` 仓库 `traffic/badges/<repo>.json`，由每日采集的官方 Traffic API 数据更新）。徽章数字含义：按日去重访客的累计（GitHub 只提供每日 uniques，跨天不去重），自 2026-08-16 起累计。

### Changed
- Changed project license from PolyForm Noncommercial License 1.0.0 to MIT License: open up the project for commercial use and redistribution, making it easier to promote and adopt. Updated LICENSE.md to the MIT License text; synced the license badge, file table, and LICENSE section in README.md and README_cn.md; simplified the contribution licensing section in CONTRIBUTING.md and CONTRIBUTING_cn.md (the noncommercial restrictions, patent clause, and extra commercial grant to maintainers are no longer needed under MIT)
- Enhanced the LLM-as-Judge evaluation plan (`.trae/documents/evaluation-plan/llm-as-judge-evaluation-plan.md`) to make the evaluation of the Rapid-Reasoning Engine statistically sound and budgeted: added an evaluation condition baseline enforcing the same "max reasoning effort" (reasoning_effort=max, temperature=1.0, top_p=0.95) for both experimental and control groups so the engine rule is the only variable (motivated by DeepSeek-V4-Flash-0731's built-in max-level deep reasoning, which would otherwise confound the attribution); added sample-size planning (12/30/60 tiers with Cohen's dz power analysis and Wilcoxon/t-test decision criteria); added length-bias control (answer_tokens covariate, explicit judge rule, and matched-length robustness check) since the engine's exhaustive-extraction rules systematically lengthen answers; added a budget estimation section based on official DeepSeek-V4-Flash-0731 API pricing (input $0.14/M cache-miss, output $0.28/M), estimating ~$0.03 for a minimal 12-question run, ~$0.25 for the recommended 30-question ×3-averaged run, and up to ~$0.95 for a fully robust 60-question ×3×3 run with one retest round — concluding token cost is not the bottleneck, the recommended tier can detect a 1-point/10 improvement

## [1.0.0] - 2026-05-27

### Added
- Tool calling rules V0 (English and Chinese)
- Tool calling rules V1 (English and Chinese)
- DeepSeek V4 Pro Max Thinking rules (Infinite-Reasoning Engine, English and Chinese)
- DeepSeek V4 Flash Max Thinking rules (Rapid-Reasoning Engine, English and Chinese)
- GLM 5.1 Max Thinking rules (Incisive-Reasoning Engine, English and Chinese)
- CONTRIBUTING.md and CONTRIBUTING_cn.md guidelines
- README.md and README_cn.md project documentation
- LICENSE.md (PolyForm Noncommercial License 1.0.0)
- .gitignore (exclude monetization-plan.md, an internal business plan that must not be committed to the public repository; also exclude tmp/ and .DS_Store)

### Changed
- Expanded project scope from tool calling rules to a comprehensive agent rules collection
- Enhanced tool calling rules from V0 to V1 with systematic improvements
- Updated documentation with detailed comparison of three reasoning engines
- Refreshed CONTRIBUTING, LICENSE, and README files

## [0.2.0] - 2026-05-25

### Added
- DeepSeek V4 Pro deep reasoning rules (initial version)
- DeepSeek V4 Flash deep reasoning rules
- Sublicense clause to CONTRIBUTING.md
- Markdown code block wrapping for all rule files

### Changed
- Updated LICENSE, README, and README_cn documentation
- Added CONTRIBUTING.md file

## [0.1.0] - 2026-05-23

### Added
- Initial deep reasoning rules framework
- Project refactoring to support multiple rule types

### Changed
- Expanded project positioning to agent-oriented rules collection

## [0.0.1] - 2026-05-11

### Added
- Initial project structure
- Tool calling rules V0 (English and Chinese)
- Tool calling rules V1 (English and Chinese)
- AGPL-3.0 LICENSE file
- README.md and README_cn.md documentation

---

## Version History Summary

- **1.0.0**: Complete agent rules collection with three reasoning engines
- **0.2.0**: Added deep reasoning rules for DeepSeek models
- **0.1.0**: Project refactoring and deep reasoning framework
- **0.0.1**: Initial release with tool calling rules