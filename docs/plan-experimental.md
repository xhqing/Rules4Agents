# 仓库2：Experimental（实验组仓库）实现计划

## 仓库名称

`reasoning-evaluation-experimental`

## 职责

包含推理引擎规则（`.trae/rules/reasoning-engine.md`），从题目仓库的 GitHub Release 获取题目，让模型在引擎规则生效的条件下逐题作答，生成深度推理风格的回答，通过 GitHub Release 发布带 label 的答案（label 固定为 `"E"`）。

***

## 仓库结构

```
reasoning-evaluation-experimental/
├── .trae/
│   └── rules/
│       └── reasoning-engine.md   # 推理引擎规则（从 AgentCortex 复制）
├── answers/
│   └── answers-labelE-v1.0.csv   # 生成的回答（带 label:"E"）
├── scripts/
│   └── fetch_questions.py        # 从题目仓库 Release 下载 questions.csv
├── input/
│   └── questions.csv             # 下载的题目（由脚本获取）
├── LICENSE
├── CONTRIBUTING.md
├── README.md
└── README_cn.md
```

***

## `.trae/rules/reasoning-engine.md` 说明

* 从 AgentCortex 项目复制对应的引擎规则文件内容

* 评测不同引擎时，替换此文件内容：

  * **Infinite-Reasoning**：复制 `dsv4proMaxThinkingRules.md` 的内容

  * **Rapid-Reasoning**：复制 `dsv4flashMaxThinkingRules.md` 的内容

  * **Incisive-Reasoning**：复制 `glm51MaxThinkingRules.md` 的内容

* 此文件作为 TRAE IDE CN workspace rules 自动加载，模型在做题时自动遵循深度推理规则

* **此文件需提交到 git**，确保回答的引擎条件可追溯

***

## 输出 CSV：`answers-labelE-v{ver}.csv`

| 列名     | 类型  | 说明                | 示例        |
| ------ | --- | ----------------- | --------- |
| qid    | str | 题目ID（关联questions） | Q01       |
| label  | str | 固定值"E"            | E         |
| answer | str | 实验组回答文本           | 让我逐步分析... |

***

## `fetch_questions.py` 脚本功能

1. 接收参数 `--repo`（题目仓库的 GitHub 仓库路径）和 `--version`（题目版本号，如 `questions-v1.0`）
2. 通过 GitHub API 获取指定 Release 的 Asset 列表
3. 下载 `questions.csv` 文件到 `input/` 目录
4. 校验 CSV 格式和字段完整性

**使用方式**：

```bash
python scripts/fetch_questions.py \
  --repo user/reasoning-evaluation-questions \
  --version questions-v1.0
```

***

## Release 规范

* **Tag 格式**：`experimental-v{major}.{minor}`，如 `experimental-v1.0`

* **Release 标题**：`Experimental v1.0 - Infinite-Reasoning Engine Answers`

* **Release Asset**：`answers-labelE-v{ver}.csv`

* **Release Body 需注明**：

  * 使用的题目版本（如 `questions-v1.0`）

  * 使用的引擎类型（如 `Infinite-Reasoning`）

  * 使用的 LLM 模型名称

  * 生成回答的日期

***

## 操作流程

1. 从 AgentCortex 项目复制对应的引擎规则文件到 `.trae/rules/reasoning-engine.md`
2. 在 TRAE IDE CN 中打开此仓库项目（引擎规则自动加载）
3. 运行 `python scripts/fetch_questions.py --version questions-v1.0` 下载题目
4. 验证 `input/questions.csv` 已正确下载（12行）
5. 在 TRAE IDE CN 对话中要求模型逐题作答（引擎规则已通过 `.trae/rules/` 自动生效）
6. 将模型生成的回答整理写入 `answers/answers-labelE-v1.0.csv`

   * 每行包含 `qid`, `label`（固定为 `E`）, `answer`（模型回答文本）

   * 确保 12 道题全部作答
7. `git add .trae/rules/reasoning-engine.md answers/answers-labelE-v1.0.csv`
8. `git commit -m "Add experimental answers v1.0 with Infinite-Reasoning engine"`
9. `git tag experimental-v1.0`
10. `git push && git push --tags`
11. 在 GitHub 上创建 Release，上传 `answers/answers-labelE-v1.0.csv` 作为 Asset

***

## 版本号约定

与题目仓库版本号对齐，同一轮评测使用相同的 `{major}.{minor}`：

| 版本     | 含义                                |
| ------ | --------------------------------- |
| `v1.0` | 第一轮评测：12 道题，Infinite-Reasoning 引擎 |
| `v1.1` | 题目微调后的重测                          |
| `v2.0` | 第二轮评测：题目大幅更新，或换引擎评测               |

***

## 防偏差要点

* 引擎规则文件 `.trae/rules/reasoning-engine.md` 作为 workspace rules 自动加载，**不需要在对话中手动提示**模型遵循规则

* 回答时不应包含任何关于引擎规则的自我描述或元信息

* 确保在独立的 TRAE IDE CN 工作区中打开此仓库，避免与对照组或评审组的上下文互相污染

***

## 后续扩展

* 支持评测三个引擎（Infinite / Rapid / Incisive），通过切换 `.trae/rules/reasoning-engine.md` 内容实现

* 增加自动化回答生成脚本（通过 API 调用模型作答）

* 增加回答质量预检脚本（检查是否所有题目都已回答、回答长度是否合理等）

