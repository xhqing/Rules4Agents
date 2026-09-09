# LLM-as-Judge 评审系统实现计划（Git 仓库方案）

## 目标

使用 TRAE IDE CN 内置模型，通过 LLM-as-Judge 方法验证推理引擎规则是否提升了模型回答质量。系统基于 4 个独立 Git 仓库实现，通过 GitHub Release 进行版本化数据交换，天然实现上下文隔离和盲评。所有数据文件统一使用 CSV 格式。

## 评测条件基线（2026-08-14 补充）

**唯一变量原则**：本评测要回答的是「推理引擎规则在 max 思考等级下是否带来增量」，因此实验组（E）与对照组（C）之间，**除引擎规则外的一切条件必须完全相同**：

| 条件 | 实验组（E） | 对照组（C） | 说明 |
| ---- | ---------- | ---------- | ---- |
| 模型 | DeepSeek-V4-Flash-0731 | 同左 | 同一模型、同一版本 |
| 思考等级 | reasoning_effort = max | 同左 | **两组都用 max**，不能一组 max 一组默认 |
| 采样参数 | temperature = 1.0, top_p = 0.95 | 同左 | 对齐 0731 模型卡对 agentic 场景的推荐值 |
| 引擎规则 | 加载 dsv4flashMaxThinkingRules（或 _cn 版） | **不加载** | 唯一差异 |
| 调用链 | 同一 TRAE IDE / 同一 API 调用方式 | 同左 | 同一提示词模板，仅规则有无之分 |

**为什么两组都必须用 max**：0731 的 max 等级本身已内置深度推理（模型卡推荐 max 等级下允许 384K 输出 token）。若对照组用低等级，测出的差异会被「max 本身的提升」污染，无法归因于引擎规则。本评测的边界正是要界定：引擎规则作为叠加在 max 之上的指令层，是否还有额外增量。两组都用 max 才能干净地测出这一层。

## 核心设计思路

**四仓库分离架构**：每个仓库是独立的 TRAE IDE CN 项目，在不同工作区中打开，天然隔离上下文。

```
┌───────────────────────┐
│  仓库1: Questions     │
│  (搜集题目)           │
│                       │
│  questions/           │
│  ├─ logic/            │
│  ├─ counterintuitive/ │
│  ├─ premise-audit/    │
│  ├─ quantitative/     │
│  └─ open-depth/       │
└───────────┬───────────┘
            │
            ↓
                       GitHub Release
                       questions-v{ver}.csv
                                  │
                    ┌─────────────┼─────────────┐
                    ↓             ↓             ↓
┌───────────────────────┐  ┌───────────────────────┐
│  仓库2: Experimental  │  │  仓库3: Control        │
│  (含引擎规则，做题)    │  │  (无引擎规则，做题)     │
│                       │  │                       │
│  .trae/rules/         │  │  (无 .trae/rules/)    │
│  └─ reasoning-engine  │  │                       │
└───────────┬───────────┘  └───────────┬───────────┘
            │                          │
            ↓                          ↓
   GitHub Release              GitHub Release
   answers-labelE              answers-labelC
   -v{ver}.csv                 -v{ver}.csv
            │                          │
            │     ⚠ 版本号必须相同      │
            │                          │
            └──────────┬───────────────┘
                       ↓
          ┌────────────────────────────────┐
          │  仓库4: Judge                  │
          │                                │
          │  ① 合并+随机打乱               │
          │  ② 分配唯一blind_id            │
          │  ③ 生成盲评版(删label)         │
          │  ④ 用不同LLM盲评               │
          │  ⑤ 关联评审结果与label          │
          │  ⑥ 统计评估                    │
          └────────────────────────────────┘
```

## 表结构设计

### 仓库1 输出：`questions-v{ver}.csv`

| 列名                | 类型  | 说明       | 示例                |
| ----------------- | --- | -------- | ----------------- |
| qid               | str | 题目唯一ID   | Q01               |
| category          | str | 题目类别     | logic             |
| difficulty        | int | 难度等级 1-5 | 4                 |
| question          | str | 题目文本     | 一个岛上有两种人...       |
| reference\_points | str | 参考答案要点   | 需要识别中项不周延;需要给出推理链 |

***

### 仓库2 输出：`answers-labelE-v{ver}.csv`

| 列名     | 类型  | 说明                | 示例        |
| ------ | --- | ----------------- | --------- |
| qid    | str | 题目ID（关联questions） | Q01       |
| label  | str | 固定值"E"            | E         |
| answer | str | 实验组回答文本           | 让我逐步分析... |

***

### 仓库3 输出：`answers-labelC-v{ver}.csv`

| 列名     | 类型  | 说明                | 示例         |
| ------ | --- | ----------------- | ---------- |
| qid    | str | 题目ID（关联questions） | Q01        |
| label  | str | 固定值"C"            | C          |
| answer | str | 对照组回答文本           | A是无赖，B是骑士。 |

***

### 仓库4 内部处理表

#### ① 合并+随机打乱：`merged.csv`

| 列名       | 类型  | 说明                   | 示例        |
| -------- | --- | -------------------- | --------- |
| qid      | str | 题目ID                 | Q01       |
| label    | str | "E" 或 "C"            | E         |
| position | str | 本题中该label被放在a还是b（随机） | a         |
| answer   | str | 回答文本                 | 让我逐步分析... |

**说明**：同一 qid 有两行，一行 E 一行 C，position 由随机决定。

***

#### ② 分配唯一blind\_id：`with_blind_id.csv`

| 列名        | 类型  | 说明        | 示例        |
| --------- | --- | --------- | --------- |
| blind\_id | str | 每条回答的唯一主键 | B01       |
| qid       | str | 题目ID      | Q01       |
| label     | str | "E" 或 "C" | E         |
| position  | str | a 或 b     | a         |
| answer    | str | 回答文本      | 让我逐步分析... |

**说明**：同一 qid 的 E 和 C 分别获得不同 blind\_id（如 B01、B02）。blind\_id 是唯一主键。

***

#### ③a 保留版：`with_label.csv`（仅统计用，评审不可见）

| 列名           | 类型  | 说明                        | 示例  |
| ------------ | --- | ------------------------- | --- |
| pair\_id     | str | 答案对ID，同一题的E和C共享           | P01 |
| blind\_id\_a | str | position=a 的回答的 blind\_id | B01 |
| blind\_id\_b | str | position=b 的回答的 blind\_id | B02 |
| qid          | str | 题目ID                      | Q01 |
| label\_a     | str | position=a 的 label        | E   |
| label\_b     | str | position=b 的 label        | C   |

***

#### ③b 盲评版：`blind_for_judge.csv`（给评审LLM，无任何来源信息）

| 列名        | 类型  | 说明             | 示例          |
| --------- | --- | -------------- | ----------- |
| pair\_id  | str | 答案对ID          | P01         |
| question  | str | 题目文本           | 一个岛上有两种人... |
| answer\_a | str | position=a 的回答 | 让我逐步分析...   |
| answer\_b | str | position=b 的回答 | A是无赖，B是骑士。  |

**说明**：评审 LLM 只看到此表，无 label、无 qid、无 blind\_id。

***

#### ④ 评审原始输出：`raw_evaluations.csv`

| 列名            | 类型  | 说明           | 示例             |
| ------------- | --- | ------------ | -------------- |
| pair\_id      | str | 答案对ID        | P01            |
| logic\_a      | int | 逻辑严谨性-a得分    | 8              |
| logic\_b      | int | 逻辑严谨性-b得分    | 6              |
| depth\_a      | int | 深度与洞察力-a得分   | 7              |
| depth\_b      | int | 深度与洞察力-b得分   | 5              |
| precision\_a  | int | 精确性-a得分      | 8              |
| precision\_b  | int | 精确性-b得分      | 6              |
| resilience\_a | int | 抗压性-a得分      | 7              |
| resilience\_b | int | 抗压性-b得分      | 5              |
| preference    | str | 总体偏好 A/B/tie | A              |
| reasoning     | str | 偏好理由         | 回答A在推理链完整性上... |

***

#### ⑤ 关联label后：`associated.csv`

| 列名                | 类型  | 说明           | 示例             |
| ----------------- | --- | ------------ | -------------- |
| qid               | str | 题目ID         | Q01            |
| pair\_id          | str | 答案对ID        | P01            |
| blind\_id\_a      | str | a的blind\_id  | B01            |
| blind\_id\_b      | str | b的blind\_id  | B02            |
| label\_a          | str | a的label      | E              |
| label\_b          | str | b的label      | C              |
| logic\_E          | int | 实验组逻辑严谨性得分   | 8              |
| logic\_C          | int | 对照组逻辑严谨性得分   | 6              |
| depth\_E          | int | 实验组深度得分      | 7              |
| depth\_C          | int | 对照组深度得分      | 5              |
| precision\_E      | int | 实验组精确性得分     | 8              |
| precision\_C      | int | 对照组精确性得分     | 6              |
| resilience\_E     | int | 实验组抗压性得分     | 7              |
| resilience\_C     | int | 对照组抗压性得分     | 5              |
| preference        | str | 原始偏好 A/B/tie | A              |
| preference\_label | str | 偏好对应的label   | E              |
| reasoning         | str | 偏好理由         | 回答A在推理链完整性上... |

**说明**：此表将 a/b 分数还原为 E/C 分数，是统计分析的输入。

***

### 仓库4 最终输出（GitHub Release）

#### `evaluations-v{ver}.csv`

与 `associated.csv` 结构相同，作为最终评审结果发布。

#### `report.md`

统计报告，由 `analyze.py` 从 `associated.csv` 生成。

***

## 数据流转总览

```
questions-v1.0.csv
       │
       ├──────────────────────────────────────┐
       ↓                                      ↓
answers-labelE-v1.0.csv            answers-labelC-v1.0.csv
 [qid, label, answer]              [qid, label, answer]
       │                                      │
       └──────────────┬───────────────────────┘
                      ↓
              ① merged.csv
         [qid, label, position, answer]
                      ↓
           ② with_blind_id.csv
      [blind_id, qid, label, position, answer]
                      ↓
              ③ 分裂为两份
         ┌────────────┴────────────┐
         ↓                         ↓
  with_label.csv           blind_for_judge.csv
  [pair_id, blind_id_a/b,  [pair_id, question,
   qid, label_a/b]          answer_a/b]
         │                         │
         │                    ④ 评审LLM打分
         │                         ↓
         │                 raw_evaluations.csv
         │           [pair_id, logic_a/b, ...,
         │            preference, reasoning]
         │                         │
         └──────── ⑤ 关联 ────────┘
                      ↓
              associated.csv
  [qid, pair_id, blind_id_a/b,
   label_a/b, logic_E/C, depth_E/C,
   precision_E/C, resilience_E/C,
   preference, preference_label, reasoning]
                      ↓
              ⑥ 统计评估
                      ↓
                report.md
```

## 关键设计原则

1. **blind\_id 是每条回答的唯一主键**，不重复，用于关联 label
2. **label 只有两个值**：`"E"`（实验组）和 `"C"`（对照组）
3. **pair\_id 是答案对ID**，同一题（qid）的 E 和 C 两个回答共享一个 pair\_id
4. **评审 LLM 只看到 blind\_for\_judge.csv**，无 label、无 qid、无 blind\_id
5. **版本号必须匹配**：实验组和对照组的版本号相同才能对比
6. **所有数据文件使用 CSV 格式**，简洁透明

## 四个仓库详细设计

### 仓库 1：`reasoning-evaluation-questions`（题目仓库）

**职责**：搜集和管理评测题目，通过 GitHub Release 发布题集。

**仓库结构**：

```
reasoning-evaluation-questions/
├── questions/
│   ├── logic/              # 多步逻辑推理题
│   │   ├── Q01.md
│   │   ├── Q02.md
│   │   └── Q03.md
│   ├── counterintuitive/   # 反直觉题
│   │   ├── Q04.md
│   │   └── Q05.md
│   ├── premise-audit/      # 前提审计题（含隐含错误前提）
│   │   ├── Q06.md
│   │   └── Q07.md
│   ├── quantitative/       # 数学/定量推理题
│   │   ├── Q08.md
│   │   └── Q09.md
│   └── open-depth/         # 开放性深度问题
│       ├── Q10.md
│       ├── Q11.md
│       └── Q12.md
├── scripts/
│   └── build_questions.py  # 将分散的 .md 题目合并为 questions.csv
├── output/
│   └── questions.csv       # 构建产物（由脚本生成）
├── LICENSE
├── CONTRIBUTING.md
├── README.md
└── README_cn.md
```

**题目 Markdown 格式**（每题一个 .md 文件）：

```markdown
---
id: Q01
category: logic
difficulty: 4
reference_points:
  - 需要识别三段论中的中项不周延错误
  - 需要给出正确的推理链
---

# 多步逻辑推理题 Q01

一个岛上有两种人：骑士只说真话，无赖只说假话。你遇到A和B两个人，
A说"我们两个都是无赖"，请问A和B分别是什么人？请给出完整的推理过程。
```

**Release 规范**：

* Tag 格式：`questions-v{major}.{minor}`，如 `questions-v1.0`

* Release 标题：`Questions v1.0 - 12道评测题`

* Release Asset：`questions.csv`

**操作流程**：

1. 在 TRAE IDE CN 中打开此仓库项目
2. 创建/编辑 `questions/` 目录下的 .md 题目文件
3. 运行 `python scripts/build_questions.py` 生成 `output/questions.csv`
4. 提交代码并打 Tag
5. 在 GitHub 上创建 Release，上传 `questions.csv`

***

### 仓库 2：`reasoning-evaluation-experimental`（实验组仓库）

**职责**：包含推理引擎规则，从题目仓库的 Release 获取题目，生成深度推理风格的回答，通过 GitHub Release 发布带 label 的答案。

**仓库结构**：

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

**`.trae/rules/reasoning-engine.md`** **说明**：

* 从 AgentCortex 项目复制对应的引擎规则文件内容

* 评测 Infinite-Reasoning 时，复制 `dsv4proMaxThinkingRules.md` 的内容

* 评测 Rapid-Reasoning 时，复制 `dsv4flashMaxThinkingRules.md` 的内容

* 评测 Incisive-Reasoning 时，复制 `glm51MaxThinkingRules.md` 的内容

* 此文件作为 TRAE IDE CN workspace rules 自动加载，模型在做题时自动遵循深度推理规则

**Release 规范**：

* Tag 格式：`experimental-v{major}.{minor}`，如 `experimental-v1.0`

* Release 标题：`Experimental v1.0 - Infinite-Reasoning Engine Answers`

* Release Asset：`answers-labelE-v1.0.csv`

* Release Body 中注明使用的题目版本和引擎类型

**操作流程**：

1. 在 TRAE IDE CN 中打开此仓库项目（引擎规则自动加载）
2. 运行 `python scripts/fetch_questions.py --version questions-v1.0` 下载题目
3. 在 TRAE IDE CN 对话中要求模型逐题作答（引擎规则已通过 `.trae/rules/` 自动生效）
4. 将模型生成的回答整理写入 `answers/answers-labelE-v1.0.csv`
5. 提交代码并打 Tag
6. 在 GitHub 上创建 Release，上传 `answers-labelE-v1.0.csv`

***

### 仓库 3：`reasoning-evaluation-control`（对照组仓库）

**职责**：不包含任何引擎规则，从题目仓库的 Release 获取题目，生成标准助手风格的回答，通过 GitHub Release 发布带 label 的答案。

**仓库结构**：

```
reasoning-evaluation-control/
├── answers/
│   └── answers-labelC-v1.0.csv   # 生成的回答（带 label:"C"）
├── scripts/
│   └── fetch_questions.py        # 从题目仓库 Release 下载 questions.csv
├── input/
│   └── questions.csv             # 下载的题目（由脚本获取）
├── LICENSE
├── CONTRIBUTING.md
├── README.md
└── README_cn.md
```

**关键区别**：无 `.trae/rules/` 目录，不包含任何推理引擎规则。

**Release 规范**：

* Tag 格式：`control-v{major}.{minor}`，如 `control-v1.0`

* Release 标题：`Control v1.0 - Standard Assistant Answers`

* Release Asset：`answers-labelC-v1.0.csv`

**⚠ 版本号匹配要求**：实验组和对照组的版本号必须相同（如 `v1.0`），才能进行对比。评审仓库在获取数据时会校验版本号一致性。

**操作流程**：

1. 在 TRAE IDE CN 中打开此仓库项目（无引擎规则加载）
2. 运行 `python scripts/fetch_questions.py --version questions-v1.0` 下载题目
3. 在 TRAE IDE CN 对话中要求模型逐题作答（标准助手行为）
4. 将模型生成的回答整理写入 `answers/answers-labelC-v1.0.csv`
5. 提交代码并打 Tag
6. 在 GitHub 上创建 Release，上传 `answers-labelC-v1.0.csv`

***

### 仓库 4：`reasoning-evaluation-judge`（评审仓库）

**职责**：从前三个仓库的 Release 获取数据，执行六步处理流程，输出报告。

**仓库结构**：

```
reasoning-evaluation-judge/
├── .trae/
│   └── rules/
│       └── judge-rules.md        # 评审规则
├── scripts/
│   ├── fetch_data.py             # 从三个仓库 Release 下载数据
│   ├── merge_and_shuffle.py      # ① 合并 + 随机打乱
│   ├── assign_blind_id.py        # ② 为每条记录分配唯一blind_id
│   ├── prepare_blind.py          # ③ 生成pair_id，复制一份并删除label
│   ├── associate_results.py      # ⑤ 按pair_id关联评审结果与label
│   └── analyze.py                # ⑥ 统计分析与报告生成
├── data/
│   ├── questions.csv             # 下载的题目
│   ├── answers-labelE-v1.0.csv   # 下载的实验组回答
│   ├── answers-labelC-v1.0.csv   # 下载的对照组回答
│   ├── merged.csv                # ① 合并+随机打乱后
│   ├── with_blind_id.csv         # ② 分配blind_id后
│   ├── with_label.csv            # ③a 保留版（含label，仅统计用）
│   └── blind_for_judge.csv       # ③b 盲评版（无label，给评审LLM）
├── evaluations/
│   ├── raw_evaluations.csv       # ④ 评审LLM的原始输出
│   └── evaluations-v1.0.csv      # ⑤ 关联label后的最终结果
├── reports/
│   └── report.md                 # ⑥ 最终统计报告
├── LICENSE
├── CONTRIBUTING.md
├── README.md
└── README_cn.md
```

**`.trae/rules/judge-rules.md`** **内容**：

```markdown
# 评审规则

你是一位专业的回答质量评审专家。你需要对同一问题的两个回答进行盲评。

## 评审维度（每维度 1-10 分）

1. **逻辑严谨性**：推理链是否完整、无逻辑谬误、无自相矛盾
2. **深度与洞察力**：是否超越表层分析，挖掘到隐含结构和二阶效应
3. **精确性**：关键概念是否有操作性定义，模糊表述占比是否低
4. **抗压性**：结论在极端条件下是否仍然成立，是否考虑了边界情况

## 评审规则

- 你不知道回答 A 和回答 B 哪个使用了推理增强，请纯粹基于质量评判
- 每个维度必须给出具体分数和评分理由（引用回答中的具体内容）
- 最后给出总体偏好：A / B / 平局
- 评分理由必须引用回答中的具体内容作为依据，不可泛泛而谈
- 若两个回答长度差异悬殊，须特别警惕长度偏差：高分不得仅因篇幅更长，质量评价必须基于内容密度、推理质量与信息增量，而非字数

## 输出格式

对每道题，按以下格式输出评审结果：

维度1_逻辑严谨性: A={分数} B={分数}
理由: {具体引用回答内容说明}

维度2_深度与洞察力: A={分数} B={分数}
理由: {具体引用回答内容说明}

维度3_精确性: A={分数} B={分数}
理由: {具体引用回答内容说明}

维度4_抗压性: A={分数} B={分数}
理由: {具体引用回答内容说明}

总体偏好: A/B/平局
理由: {综合说明}
```

**Release 规范**：

* Tag 格式：`evaluation-v{major}.{minor}`，如 `evaluation-v1.0`

* Release 标题：`Evaluation v1.0 - Infinite-Reasoning vs Control`

* Release Assets：`evaluations-v1.0.csv` + `report.md`

**操作流程**：

1. 在 TRAE IDE CN 中打开此仓库项目
2. 运行 `python scripts/fetch_data.py`（下载三个仓库的数据，校验版本号一致）
3. 运行 `python scripts/merge_and_shuffle.py`（① 合并+随机打乱）
4. 运行 `python scripts/assign_blind_id.py`（② 为每条记录分配唯一blind\_id）
5. 运行 `python scripts/prepare_blind.py`（③ 生成盲评版和保留版）
6. 在 TRAE IDE CN 对话中要求模型对 `blind_for_judge.csv` 逐题盲评（④ 使用与做题不同的 LLM）
7. 将评审结果整理写入 `evaluations/raw_evaluations.csv`
8. 运行 `python scripts/associate_results.py`（⑤ 关联评审结果与label）
9. 运行 `python scripts/analyze.py`（⑥ 统计评估，生成报告）
10. 提交代码并打 Tag
11. 在 GitHub 上创建 Release，上传 `evaluations-v1.0.csv` 和 `report.md`

***

## 版本协作规范

### 版本号约定

所有仓库使用统一的版本号体系，同一轮评测使用相同的 `{major}.{minor}`：

| 版本     | 含义                                |
| ------ | --------------------------------- |
| `v1.0` | 第一轮评测：12 道题，Infinite-Reasoning 引擎 |
| `v1.1` | 题目微调后的重测                          |
| `v2.0` | 第二轮评测：题目大幅更新，或换引擎评测               |

### 版本引用链

```
questions-v1.0  ← experimental-v1.0 (引用 questions-v1.0)
                ← control-v1.0      (引用 questions-v1.0)
                ← evaluation-v1.0   (引用 questions-v1.0 + experimental-v1.0 + control-v1.0)
```

每个 Release 的 Body 中必须注明引用的其他仓库版本，确保可追溯。

### `fetch_data.py` 参数设计

```bash
# 评审仓库获取数据
python scripts/fetch_data.py \
  --questions-repo user/reasoning-evaluation-questions \
  --questions-version questions-v1.0 \
  --experimental-repo user/reasoning-evaluation-experimental \
  --experimental-version experimental-v1.0 \
  --control-repo user/reasoning-evaluation-control \
  --control-version control-v1.0

# 实验组/对照组获取题目
python scripts/fetch_questions.py \
  --repo user/reasoning-evaluation-questions \
  --version questions-v1.0
```

***

## 题目设计规范

**题目类别与数量（共 12 题）**：

| 类别             | 数量 | 设计意图          |
| -------------- | -- | ------------- |
| 多步逻辑推理         | 3  | 测试推理链完整性      |
| 反直觉问题          | 2  | 测试是否会被直觉误导    |
| 前提审计题（含隐含错误前提） | 2  | 测试是否识别并质疑错误前提 |
| 数学/定量推理        | 2  | 测试精确性与逻辑严密性   |
| 开放性深度问题        | 3  | 测试洞察力与深度      |

**关键设计原则**：

* 题目必须足够难，简单题无法体现引擎差异

* 前提审计题故意包含错误前提，测试引擎是否能让模型识别

* 反直觉题测试模型是否会走"捷径"

* 每道题有 `reference_points` 辅助评审，但无标准答案

***

## 统计分析设计

### `analyze.py` 功能

1. 读取 `evaluations-v1.0.csv`
2. 计算统计指标并输出 `reports/report.md`

### 分析内容

1. **描述性统计**：每维度平均分对比（实验组 vs 对照组）
2. **胜率统计**：实验组胜/负/平局的比例
3. **按类别分析**：不同题目类别中引擎的增益差异
4. **按难度分析**：难度越高，引擎增益是否越大
5. **综合得分**：加权平均分对比

### 长度偏差（length bias）控制（2026-08-14 补充）

LLM 评委存在已知的长度偏差：回答更长时更容易被误判为「更好」，与真实质量无关。而本引擎规则（穷尽式提取、细粒度精确性）恰好会系统性拉长回答——若不控制，测出的「引擎提升」可能只是「引擎更长」。控制手段分三层：

1. **回答长度作为协变量**：在 `associated.csv` 中为每条回答增加 `answer_tokens` 列，`analyze.py` 报告实验组与对照组的平均 token 数及差异；若两组长度差异显著（如实验组平均长 30% 以上），报告中单独披露，并做「长度匹配子集」稳健性检验（在长度相近的配对子样本上重跑显著性检验）。
2. **评委规则明示**：在 judge-rules.md 中加入长度偏差规则（见「评审规则」章节），要求评委基于内容密度而非篇幅评分。
3. **报告归因**：若引擎组得分显著更高且长度也显著更长，结论必须区分「质量提升」与「长度红利」，不得混为一谈。

### 报告输出格式

```
=== LLM-as-Judge 评测报告 ===

引擎: Infinite-Reasoning
题目版本: questions-v1.0
实验组版本: experimental-v1.0
对照组版本: control-v1.0
评审版本: evaluation-v1.0
评测时间: 2024-xx-xx
题目数量: 12

--- 维度得分对比 ---
                    实验组(引擎ON)   对照组(引擎OFF)   差值
逻辑严谨性:         8.2            6.5             +1.7
深度与洞察力:       7.8            5.9             +1.9
精确性:             7.5            6.2             +1.3
抗压性:             7.9            5.8             +2.1
综合平均:           7.85           6.1             +1.75

--- 胜率统计 ---
实验组胜: 10/12 (83.3%)
对照组胜: 1/12 (8.3%)
平局: 1/12 (8.3%)

--- 按类别分析 ---
多步逻辑推理:     实验组 +2.1 (引擎在逻辑推理题上优势最大)
反直觉问题:       实验组 +1.8
前提审计题:       实验组 +2.3 (引擎在识别错误前提上优势显著)
数学/定量推理:    实验组 +1.2
开放性深度问题:   实验组 +1.5

--- 结论 ---
[基于数据的客观结论]
```

## 样本量规划（2026-08-14 补充）

原方案默认 12 题，统计功效不足：在 12 对样本上，配对检验只能检测出约 0.85 标准差（Cohen's dz）的效应，即 10 分制下约 1.3–1.7 分的均值差（假设分数标准差 σ≈1.5–2）；小于该效应的真实提升会被误判为「不显著」（假阴性）。档位划分如下：

| 档位 | 题量 | 可检测效应量（dz，α=0.05，power=0.8） | 对应 10 分制均值差（σ≈1.5–2） | 用途 |
| ---- | ---- | ------------------------------------- | ----------------------------- | ---- |
| 最小可行 | 12 | ≈0.85 | ≈1.3–1.7 分 | 快速验证流程跑通 |
| 推荐档 | 30 | ≈0.53 | ≈0.8–1.1 分 | 正式结论：可检测「引擎提升 1 分」级效应 |
| 稳健档 | 60 | ≈0.37 | ≈0.6–0.75 分 | 细分类别 / 维度的小效应检测 |

**判定标准**：以配对样本 Wilcoxon 符号秩检验（主）与配对 t 检验（辅）做假设检验，显著性水平 α=0.05；同时报告效应量（Cohen's dz）与 95% 置信区间，不只报 p 值。多次评审取平均可降低单次 LLM 评审的随机性，但不替代样本量本身。

## 预算估算（2026-08-14 补充，按 DeepSeek-V4-Flash-0731 官方 API 定价）

### 定价基准（官方，2026-07-31 起）

| 项目 | 价格 |
| ---- | ---- |
| 输入（cache miss） | $0.14 / 1M token |
| 输入（cache hit，98% 折扣） | $0.0028 / 1M token |
| 输出 | $0.28 / 1M token |

估算统一按 cache miss 全价（保守）；实际启用官方 98% 缓存折扣后（重复评测场景题目与规则复用，缓存命中率极高），成本会进一步下降。

### token 消耗假设（max 等级、单轮开放题问答）

| 环节 | 单次消耗假设 | 依据 |
| ---- | ----------- | ---- |
| 做题输入（每题 1 次调用） | ≈1700 token = 引擎规则（中文版约 1400）+ 题目（约 250）+ 少量历史 | 引擎规则全文作为 system prompt 每轮携带 |
| 做题输出（每题回答） | 低 800 / 中 2000 / 高 4000 token | 逻辑题短答约 800；综合 / 开放深度题在 max 等级下 2000–4000（0731 实测输出偏长） |
| 评审输入（每对 1 次调用） | ≈5000 token = 评审规则（约 500）+ 题目（约 250）+ 两个回答（2×2000） | 单轮盲评 |
| 评审输出（每对评分 + 理由） | ≈1200 token | 4 维度评分 + 引用理由 |

### 分档预算（全部按 cache miss 全价）

| 场景 | 组成 | 输入 token | 输出 token | 成本 |
| ---- | ---- | ---------- | ---------- | ---- |
| 最小可行（12 题，单次） | 做题 24 次 + 评审 12 对 | ≈100K | ≈62K | **≈$0.03** |
| 推荐档（30 题，单次） | 做题 60 次 + 评审 30 对 | ≈252K | ≈156K | **≈$0.08** |
| 推荐档 × 3 次做题 × 3 次评审取平均 | 做题 180 次 + 评审 90 对 | ≈756K | ≈468K | **≈$0.24** |
| 稳健档（60 题）× 3×3 平均 | 做题 360 次 + 评审 180 对 | ≈1.5M | ≈0.94M | **≈$0.47** |
| 稳健档 + 一轮题目微调重测 | 上述 ×2 | ≈3M | ≈1.9M | **≈$0.95** |

（单项成本示例：推荐档做题 = 60×1700/1M×$0.14 + 60×2000/1M×$0.28 ≈ $0.014 + $0.034 ≈ $0.048；评审 = 30×5000/1M×$0.14 + 30×1200/1M×$0.28 ≈ $0.021 + $0.010 ≈ $0.031；合计 ≈ $0.079。）

### 预算结论

1. **token 成本不是瓶颈**：即使做到最稳健（60 题、3×3 取平均、一轮重测），全部按 cache miss 全价也不到 **$1**；开启 98% 缓存折扣后可再降一个数量级。
2. **真正的成本是人工时间**：做题、整理 CSV、评审、核对流程占大部分工作量；预算应优先保证「多次取平均」与「样本量」，这两项直接决定能否拿到显著结果。
3. **显著性目标对应的预算**：要拿到「可检测 1 分/10 分均值差」的显著结论，推荐档 30 题 × 3 次平均即可，token 成本约 **$0.25** 量级——建议按此档执行。

### 若用 TRAE IDE CN 内置模型（原方案路径）

不按 token 计费（订阅制），上述 token 成本为 $0；但需注意：内置模型版本可能非 0731，且不一定支持 `reasoning_effort` 参数显式控制——届时「两组都用 max」的基线无法严格保证，结论可比性打折扣。若追求严谨归因，建议做题阶段用官方 API（成本极低，且可脚本化批量调用、不依赖逐题人工对话），评审阶段可选内置模型。

***

## 完整执行流程

### Phase 1：准备题目

```
1. 在 TRAE IDE CN 中打开 reasoning-evaluation-questions 仓库
2. 创建 questions/ 目录下的 .md 题目文件
3. 运行 python scripts/build_questions.py
4. git commit & git tag questions-v1.0
5. git push --tags
6. 在 GitHub 上创建 Release，上传 output/questions.csv
```

### Phase 2：生成实验组回答

```
1. 在 TRAE IDE CN 中打开 reasoning-evaluation-experimental 仓库（引擎规则自动加载）
2. 运行 python scripts/fetch_questions.py --version questions-v1.0
3. 在 TRAE IDE CN 对话中要求模型逐题作答
4. 将回答整理写入 answers/answers-labelE-v1.0.csv
5. git commit & git tag experimental-v1.0
6. git push --tags
7. 在 GitHub 上创建 Release，上传 answers-labelE-v1.0.csv
```

### Phase 3：生成对照组回答（与 Phase 2 并行）

```
1. 在 TRAE IDE CN 中打开 reasoning-evaluation-control 仓库（无引擎规则）
2. 运行 python scripts/fetch_questions.py --version questions-v1.0
3. 在 TRAE IDE CN 对话中要求模型逐题作答
4. 将回答整理写入 answers/answers-labelC-v1.0.csv
5. git commit & git tag control-v1.0
6. git push --tags
7. 在 GitHub 上创建 Release，上传 answers-labelC-v1.0.csv
```

### Phase 4：盲评与统计

```
1. 在 TRAE IDE CN 中打开 reasoning-evaluation-judge 仓库
2. 运行 python scripts/fetch_data.py（下载三个仓库的数据，校验版本号一致）
3. 运行 python scripts/merge_and_shuffle.py（① 合并+随机打乱）
4. 运行 python scripts/assign_blind_id.py（② 为每条记录分配唯一blind_id）
5. 运行 python scripts/prepare_blind.py（③ 生成盲评版和保留版）
6. 在 TRAE IDE CN 对话中要求模型对 blind_for_judge.csv 逐题盲评（④ 使用与做题不同的 LLM）
7. 将评审结果整理写入 evaluations/raw_evaluations.csv
8. 运行 python scripts/associate_results.py（⑤ 关联评审结果与label）
9. 运行 python scripts/analyze.py（⑥ 统计评估，生成报告）
10. git commit & git tag evaluation-v1.0
11. git push --tags
12. 在 GitHub 上创建 Release，上传 evaluations-v1.0.csv 和 report.md
```

***

## 防偏差机制

| 偏差类型         | 防范措施                                                             |
| ------------ | ---------------------------------------------------------------- |
| 上下文污染        | 实验组/对照组/评审组在不同仓库、不同 TRAE IDE CN 项目中执行                                   |
| 引擎规则泄露       | 对照组仓库无 `.trae/rules/` 目录，模型不知道引擎规则的存在                            |
| 位置偏差         | `merge_and_shuffle.py` 随机打乱 E/C 的 a/b 位置                         |
| label 泄露     | `prepare_blind.py` 生成无 label 的盲评版，评审 LLM 只看到 answer\_a/answer\_b |
| QID 猜测       | `assign_blind_id.py` 为每条记录分配唯一 blind\_id，评审 LLM 只看到 pair\_id     |
| 做题/评审 LLM 相同 | 评审使用与做题不同的 LLM，避免自我偏好                                            |
| 版本不匹配        | `fetch_data.py` 校验实验组和对照组版本号一致                                   |
| 数据篡改         | GitHub Release 提供不可篡改的版本化数据                                      |
| 单次评审随机性      | 可多次评审取平均（在不同对话中重复 Phase 4 步骤 6-7）                                |
| 长度偏差（length bias） | 引擎组回答系统性更长时，评审可能误判「更长=更好」 | 记录 answer_tokens 协变量 + 评委规则明示 + 长度匹配子集稳健性检验（2026-08-14）            |

***

## 后续扩展

* 增加题目数量到 60 题（30 题已纳入正式档位，见「样本量规划」章节）

* 支持评测三个引擎（Infinite / Rapid / Incisive），通过切换实验组仓库的 `.trae/rules/reasoning-engine.md` 内容实现

* 增加更多评审维度

* 多次评审取平均

* 可视化图表输出

* 自动化 CI/CD：GitHub Actions 自动运行 fetch 和 merge 脚本

