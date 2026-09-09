# 仓库4：Judge（评审仓库）实现计划

## 仓库名称

`reasoning-evaluation-judge`

## 职责

从前三个仓库（Questions、Experimental、Control）的 GitHub Release 获取数据，执行六步处理流程（合并打乱 → 分配盲评ID → 生成盲评版 → 评审LLM打分 → 关联label → 统计评估），最终输出评审结果和统计报告。

---

## 仓库结构

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

---

## `.trae/rules/judge-rules.md` 内容

此文件作为 TRAE IDE CN workspace rules 自动加载，指导评审 LLM 进行盲评。

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

---

## 六步处理流程

### 步骤①：合并 + 随机打乱

**脚本**：`merge_and_shuffle.py`

**输入**：
- `answers-labelE-v{ver}.csv`（实验组回答）
- `answers-labelC-v{ver}.csv`（对照组回答）

**输出**：`data/merged.csv`

| 列名       | 类型  | 说明                   | 示例        |
| -------- | --- | -------------------- | --------- |
| qid      | str | 题目ID                 | Q01       |
| label    | str | "E" 或 "C"            | E         |
| position | str | 本题中该label被放在a还是b（随机） | a         |
| answer   | str | 回答文本                 | 让我逐步分析... |

**逻辑**：
1. 读取实验组和对照组的回答 CSV
2. 对每个 qid，随机决定 E 放在 position=a 还是 position=b
3. 同一 qid 生成两行（一行 E 一行 C），position 互补
4. 整体随机打乱顺序

---

### 步骤②：分配唯一 blind_id

**脚本**：`assign_blind_id.py`

**输入**：`data/merged.csv`

**输出**：`data/with_blind_id.csv`

| 列名        | 类型  | 说明        | 示例        |
| --------- | --- | --------- | --------- |
| blind_id  | str | 每条回答的唯一主键 | B01       |
| qid       | str | 题目ID      | Q01       |
| label     | str | "E" 或 "C" | E         |
| position  | str | a 或 b     | a         |
| answer    | str | 回答文本      | 让我逐步分析... |

**逻辑**：
1. 为 merged.csv 的每一行分配唯一的 blind_id（如 B01, B02, ..., B24）
2. blind_id 是唯一主键，不重复

---

### 步骤③：生成盲评版和保留版

**脚本**：`prepare_blind.py`

**输入**：`data/with_blind_id.csv` + `questions.csv`

**输出 ③a**：`data/with_label.csv`（仅统计用，评审不可见）

| 列名           | 类型  | 说明                        | 示例  |
| ------------ | --- | ------------------------- | --- |
| pair_id      | str | 答案对ID，同一题的E和C共享           | P01 |
| blind_id_a   | str | position=a 的回答的 blind_id | B01 |
| blind_id_b   | str | position=b 的回答的 blind_id | B02 |
| qid          | str | 题目ID                      | Q01 |
| label_a      | str | position=a 的 label        | E   |
| label_b      | str | position=b 的 label        | C   |

**输出 ③b**：`data/blind_for_judge.csv`（给评审LLM，无任何来源信息）

| 列名        | 类型  | 说明             | 示例          |
| --------- | --- | -------------- | ----------- |
| pair_id   | str | 答案对ID          | P01         |
| question  | str | 题目文本           | 一个岛上有两种人... |
| answer_a  | str | position=a 的回答 | 让我逐步分析...   |
| answer_b  | str | position=b 的回答 | A是无赖，B是骑士。  |

**逻辑**：
1. 按 qid 分组，将同一 qid 的 E 和 C 两个 blind_id 配对
2. 分配 pair_id（P01, P02, ..., P12）
3. 生成 with_label.csv（保留所有 label 信息）
4. 生成 blind_for_judge.csv（删除 label、qid、blind_id，仅保留 pair_id、question、answer_a、answer_b）

---

### 步骤④：评审 LLM 打分

**操作方式**：人工在 TRAE IDE CN 对话中完成

1. 确保 `.trae/rules/judge-rules.md` 已正确配置
2. 在 TRAE IDE CN 对话中，逐题要求模型对 `blind_for_judge.csv` 进行盲评
3. **评审必须使用与做题不同的 LLM**，避免自我偏好

**输出**：`evaluations/raw_evaluations.csv`

| 列名            | 类型  | 说明           | 示例             |
| ------------- | --- | ------------ | -------------- |
| pair_id       | str | 答案对ID        | P01            |
| logic_a       | int | 逻辑严谨性-a得分    | 8              |
| logic_b       | int | 逻辑严谨性-b得分    | 6              |
| depth_a       | int | 深度与洞察力-a得分   | 7              |
| depth_b       | int | 深度与洞察力-b得分   | 5              |
| precision_a   | int | 精确性-a得分      | 8              |
| precision_b   | int | 精确性-b得分      | 6              |
| resilience_a  | int | 抗压性-a得分      | 7              |
| resilience_b  | int | 抗压性-b得分      | 5              |
| preference    | str | 总体偏好 A/B/tie | A              |
| reasoning     | str | 偏好理由         | 回答A在推理链完整性上... |

**注意**：此文件需人工从评审 LLM 的输出中整理填入。

---

### 步骤⑤：关联评审结果与 label

**脚本**：`associate_results.py`

**输入**：
- `evaluations/raw_evaluations.csv`（评审原始输出）
- `data/with_label.csv`（label 映射表）

**输出**：`evaluations/evaluations-v{ver}.csv`

| 列名                | 类型  | 说明           | 示例             |
| ----------------- | --- | ------------ | -------------- |
| qid               | str | 题目ID         | Q01            |
| pair_id           | str | 答案对ID        | P01            |
| blind_id_a        | str | a的blind_id  | B01            |
| blind_id_b        | str | b的blind_id  | B02            |
| label_a           | str | a的label      | E              |
| label_b           | str | b的label      | C              |
| logic_E           | int | 实验组逻辑严谨性得分   | 8              |
| logic_C           | int | 对照组逻辑严谨性得分   | 6              |
| depth_E           | int | 实验组深度得分      | 7              |
| depth_C           | int | 对照组深度得分      | 5              |
| precision_E       | int | 实验组精确性得分     | 8              |
| precision_C       | int | 对照组精确性得分     | 6              |
| resilience_E      | int | 实验组抗压性得分     | 7              |
| resilience_C      | int | 对照组抗压性得分     | 5              |
| preference        | str | 原始偏好 A/B/tie | A              |
| preference_label  | str | 偏好对应的label   | E              |
| reasoning         | str | 偏好理由         | 回答A在推理链完整性上... |

**逻辑**：
1. 根据 with_label.csv 将 a/b 分数还原为 E/C 分数
2. 如果 preference 是 "A"，则 preference_label = label_a
3. 如果 preference 是 "B"，则 preference_label = label_b
4. 如果 preference 是 "tie"，则 preference_label = "tie"
5. 关联 qid 信息

---

### 步骤⑥：统计评估与报告生成

**脚本**：`analyze.py`

**输入**：`evaluations/evaluations-v{ver}.csv`

**输出**：`reports/report.md`

**分析内容**：
1. **描述性统计**：每维度平均分对比（实验组 vs 对照组）
2. **胜率统计**：实验组胜/负/平局的比例
3. **按类别分析**：不同题目类别中引擎的增益差异
4. **按难度分析**：难度越高，引擎增益是否越大
5. **综合得分**：加权平均分对比

**报告输出格式**：
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

---

## `fetch_data.py` 参数设计

```bash
# 评审仓库获取数据
python scripts/fetch_data.py \
  --questions-repo user/reasoning-evaluation-questions \
  --questions-version questions-v1.0 \
  --experimental-repo user/reasoning-evaluation-experimental \
  --experimental-version experimental-v1.0 \
  --control-repo user/reasoning-evaluation-control \
  --control-version control-v1.0
```

**脚本功能**：
1. 通过 GitHub API 从三个仓库的指定 Release 下载数据
2. 校验实验组和对照组的版本号 `{major}.{minor}` 必须一致
3. 下载文件保存到 `data/` 目录

---

## Release 规范

- **Tag 格式**：`evaluation-v{major}.{minor}`，如 `evaluation-v1.0`
- **Release 标题**：`Evaluation v1.0 - Infinite-Reasoning vs Control`
- **Release Assets**：`evaluations-v{ver}.csv` + `report.md`
- **Release Body 需注明**：
  - 引用的题目版本、实验组版本、对照组版本
  - 使用的评审 LLM 模型名称
  - 评审日期

---

## 完整操作流程

1. 在 TRAE IDE CN 中打开此仓库项目
2. 运行 `python scripts/fetch_data.py`（下载三个仓库的数据，校验版本号一致）
3. 运行 `python scripts/merge_and_shuffle.py`（① 合并+随机打乱）
4. 运行 `python scripts/assign_blind_id.py`（② 为每条记录分配唯一blind_id）
5. 运行 `python scripts/prepare_blind.py`（③ 生成盲评版和保留版）
6. 在 TRAE IDE CN 对话中要求模型对 `data/blind_for_judge.csv` 逐题盲评（④ 使用与做题不同的 LLM）
7. 将评审结果整理写入 `evaluations/raw_evaluations.csv`
8. 运行 `python scripts/associate_results.py`（⑤ 关联评审结果与label）
9. 运行 `python scripts/analyze.py`（⑥ 统计评估，生成报告）
10. `git add .trae/rules/judge-rules.md`
11. `git commit -m "Add evaluation v1.0"`
12. `git tag evaluation-v1.0`
13. `git push && git push --tags`
14. 在 GitHub 上创建 Release，上传 `evaluations/evaluations-v1.0.csv` 和 `reports/report.md` 作为 Assets

---

## 防偏差机制

| 偏差类型         | 防范措施                                                             |
| ------------ | ---------------------------------------------------------------- |
| 上下文污染        | 评审组在独立仓库、独立 TRAE IDE CN 项目中执行                                      |
| 位置偏差         | `merge_and_shuffle.py` 随机打乱 E/C 的 a/b 位置                         |
| label 泄露     | `prepare_blind.py` 生成无 label 的盲评版，评审 LLM 只看到 answer_a/answer_b |
| QID 猜测       | `assign_blind_id.py` 为每条记录分配唯一 blind_id，评审 LLM 只看到 pair_id     |
| 做题/评审 LLM 相同 | 评审使用与做题不同的 LLM，避免自我偏好                                            |
| 版本不匹配        | `fetch_data.py` 校验实验组和对照组版本号一致                                   |
| 数据篡改         | GitHub Release 提供不可篡改的版本化数据                                      |
| 单次评审随机性      | 可多次评审取平均（在不同对话中重复步骤 6-7）                                      |

---

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
              associated.csv / evaluations-v1.0.csv
  [qid, pair_id, blind_id_a/b,
   label_a/b, logic_E/C, depth_E/C,
   precision_E/C, resilience_E/C,
   preference, preference_label, reasoning]
                      ↓
              ⑥ 统计评估
                      ↓
                report.md
```

---

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

---

## 后续扩展

- 增加题目数量到 30 题
- 支持评测三个引擎（Infinite / Rapid / Incisive）
- 增加更多评审维度
- 多次评审取平均
- 可视化图表输出（使用 matplotlib / plotly 生成图表）
- 自动化 CI/CD：GitHub Actions 自动运行 fetch 和 merge 脚本
