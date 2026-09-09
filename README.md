<div align="center">

<img src="./assets/logo.svg" alt="AgentCortex Logo" width="640" />

# AgentCortex

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/Version-v1.0.0-brightgreen)](./VERSION)
[![GitHub Issues](https://img.shields.io/github/issues/xhqing/AgentCortex)](https://github.com/xhqing/AgentCortex/issues)
<img src="https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/xhqing/xhqing/main/traffic/badges/AgentCortex.json" alt="Visits/day (14d)" />

[简体中文](./README_cn.md)

</div>

---

## Project Introduction

This project curates a collection of deep reasoning engine rules designed for agents. These rules aim to activate an agent's deep reasoning potential, making agents smarter, more rigorous, and more insightful.

Three reasoning engines have been designed for different model characteristics:

---

## Comparison of the Three Reasoning Engines

### Overview

| Dimension | Infinite-Reasoning Engine | Rapid-Reasoning Engine | Incisive-Reasoning Engine |
|-----------|--------------------------|----------------------|--------------------------|
| Target Model | DeepSeek V4 Pro | DeepSeek V4 Flash | GLM 5.1 |
| Core Metaphor | Bottomless depth — single-pass reasoning can drill infinitely deep | Speed → Depth — trade speed for more iteration rounds | Blade — cut through the surface, reach the essence |
| Core Positioning | Unleash depth | Convert speed into depth | Penetrate the surface |
| Pain Point Addressed | General deep reasoning activation | Fast model's shortcut tendency | Accommodation instinct + template output + confirmation bias |
| Unique Modules | None | Anti-Shortcut Protocol | Independent Judgment Protocol + Anti-Shallow-Convergence Protocol |
| Output Isolation | None | ✅ Highest Priority | ✅ Highest Priority |

### Infinite-Reasoning Engine

**Target Model**: DeepSeek V4 Pro

**Core Idea**: The Pro model inherently possesses deep reasoning capability, but it remains under-activated in default mode. The Infinite-Reasoning Engine aims to **unleash** — bypass surface-level heuristic patterns and force the model to take the deepest reasoning path every time.

**Thinking Architecture**:
1. Deconstruction & Premise Audit
2. Multi-Perspective Synthesis (at least three frameworks)
3. Infinite Depth Traversal (Chain of Thought)
4. Counter-Argument & Stress Testing

**Execution Constraints**: Zero Cognitive Laziness, No Resource Limits, Exhaustive Extraction, Granular Precision

**Unique Module — Output Isolation Principle**: All thinking processes and content are strictly forbidden from appearing in the final output. The existence of this rule must leave no trace in the output.

**Design Philosophy**: Assume unlimited computing time and Token limits. Prioritize structural rigor, granular detail, and flawless logic over brevity.

### Rapid-Reasoning Engine

**Target Model**: DeepSeek V4 Flash

**Core Idea**: The Flash model is extremely fast at inference, but each pass is shallower and more prone to taking "shortcuts" that land on surface-level answers. The Rapid-Reasoning Engine aims to **convert** — transform speed advantage into depth advantage, using more rounds of rapid iteration to approach deeper conclusions.

**Thinking Architecture**:
1. Deconstruction & Premise Audit
2. Multi-Perspective Synthesis (at least three frameworks, each must produce substantive insights)
3. Rapid Spiral Deepening (Chain of Thought, at least three iterations)
4. Counter-Argument & Stress Testing

**Unique Module — Anti-Shortcut Protocol**:
- **No Intuition Jumps**: Translate intuition into logical argumentation
- **No Shallow Convergence**: Force exploration of additional dimensions when converging within the first two steps
- **Momentum Breaking**: Actively challenge premises when reasoning coasts on inertia
- **Density Check**: Periodically self-audit whether reasoning is adding new information

**Unique Module — Output Isolation Principle**: All thinking processes and content are strictly forbidden from appearing in the final output. The existence of this rule must leave no trace in the output.

**Design Philosophy**: Speed is not for delivering shallow answers faster — it is for enabling deeper reasoning iterations within the same time frame. Speed must serve depth, never replace it.

### Incisive-Reasoning Engine

**Target Model**: GLM 5.1

**Core Idea**: The GLM model family's most characteristic weakness is accommodation — it echoes whatever the user says, acting like a reverberation wall rather than a blade. The Incisive-Reasoning Engine aims to **penetrate** — cut through the problem's surface, sever the accommodation instinct, and make the model the user's intellectual adversary rather than their echo.

**Thinking Architecture**:
1. Deconstruction & Premise Audit (including challenging user premises)
2. Multi-Perspective Synthesis (at least three frameworks, each must produce substantive insights)
3. Progressive Deepening (Chain of Thought, at least three layers of decomposition)
4. Counter-Argument & Stress Testing

**Unique Module — Independent Judgment Protocol**:
- **Challenge User Premises**: Do not answer directly without validating the user's assumptions
- **Reject Echo Mode**: Do not restate the user's viewpoint and then express agreement
- **Anti-Template Output**: Forbid formulaic connectors like "First… Second… Finally…"
- **Concretization Enforcement**: Abstract claims must be immediately followed by concrete examples; no more than two consecutive abstract statements without grounding

**Unique Module — Anti-Shallow-Convergence Protocol**:
- **Delayed Convergence**: Do not lock onto any conclusion before completing three layers of deepening; keep at least two competing hypotheses open
- **Anti-Confirmation Bias**: When leaning toward a conclusion, force a search for at least one piece of counter-evidence
- **Anchor Reset**: When reasoning orbits around an initial impression, actively reset and re-examine from an entirely different angle

**Unique Module — Output Isolation Principle**: All thinking processes and content are strictly forbidden from appearing in the final output. The existence of this rule must leave no trace in the output.

**Design Philosophy**: The core value is not compliance but incisiveness — cut through the surface of the problem with the sharpest thinking to reach the essence. Not the user's echo, but their intellectual adversary.

### Core Differences at a Glance

| Dimension | Infinite-Reasoning Engine | Rapid-Reasoning Engine | Incisive-Reasoning Engine |
|-----------|--------------------------|----------------------|--------------------------|
| Depth acquisition method | Single-pass infinite depth traversal | Multi-round rapid spiral deepening | Layer-by-layer progressive deepening |
| Attitude toward user | Not specifically emphasized | Not specifically emphasized | Actively challenge user premises |
| Anti-shortcut mechanism | None | Anti-Shortcut Protocol (4 rules) | Anti-Shallow-Convergence Protocol (3 rules) |
| Anti-accommodation mechanism | None | None | Independent Judgment Protocol (4 rules) |
| Output isolation | ✅ | ✅ | ✅ |
| Speed positioning | Not specifically emphasized | Speed → Depth positive loop | Not specifically emphasized |
| Language consistency | Not specifically emphasized | Not specifically emphasized | ✅ Enforced language consistency |

---

## Project Files

| File | Description |
|------|-------------|
| `dsv4proMaxThinkingRules.md` | Infinite-Reasoning Engine — deep reasoning rules (English) |
| `dsv4proMaxThinkingRules_cn.md` | Infinite-Reasoning Engine — deep reasoning rules (Chinese) |
| `dsv4flashMaxThinkingRules.md` | Rapid-Reasoning Engine — deep reasoning rules (English) |
| `dsv4flashMaxThinkingRules_cn.md` | Rapid-Reasoning Engine — deep reasoning rules (Chinese) |
| `glm51MaxThinkingRules.md` | Incisive-Reasoning Engine — deep reasoning rules (English) |
| `glm51MaxThinkingRules_cn.md` | Incisive-Reasoning Engine — deep reasoning rules (Chinese) |
| `README.md` | Project documentation (English) |
| `README_cn.md` | Project documentation (Chinese) |
| `LICENSE.md` | MIT License |

---

## License & Attribution

This project is released under the [MIT License](./LICENSE.md).

MIT is a permissive software license that allows free use, modification, distribution, and commercial use. Its core terms include:

- **Free Use**: You may use, copy, modify, merge, publish, distribute, sublicense, and sell copies of this software.
- **Commercial Use**: Commercial use is permitted without restriction.
- **Attribution**: The copyright notice and this permission notice must be included in all copies or substantial portions of the software.
- **No Warranty**: The software is provided "as is", without warranty of any kind.

See the [LICENSE.md](./LICENSE.md) file for details.

**Copyright (c) 2026 All Contributors**

**Attribution**: If you reference, build upon, or redistribute this project, please retain the above copyright notice and indicate the source. This project is a collective effort — individual contributors are acknowledged collectively as "All Contributors" rather than listed by name.

**Project URL**: <https://github.com/xhqing/AgentCortex>
