# 仓库1：Questions（题目仓库）实现计划

## 仓库名称

`reasoning-evaluation-questions`

## 职责

搜集和管理评测题目，按类别组织，通过脚本将分散的 .md 题目文件合并为 `questions.csv`，通过 GitHub Release 发布题集供实验组和对照组使用。

---

## 仓库结构

```
reasoning-evaluation-questions/
├── questions/
│   ├── logic/              # 多步逻辑推理题（3题）
│   │   ├── Q01.md
│   │   ├── Q02.md
│   │   └── Q03.md
│   ├── counterintuitive/   # 反直觉题（2题）
│   │   ├── Q04.md
│   │   └── Q05.md
│   ├── premise-audit/      # 前提审计题（含隐含错误前提）（2题）
│   │   ├── Q06.md
│   │   └── Q07.md
│   ├── quantitative/       # 数学/定量推理题（2题）
│   │   ├── Q08.md
│   │   └── Q09.md
│   └── open-depth/         # 开放性深度问题（3题）
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

---

## 题目 Markdown 格式

每题一个 .md 文件，使用 YAML front matter 定义元数据：

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

**字段说明**：
- `id`: 题目唯一ID，格式 `QXX`
- `category`: 类别，取值 `logic` / `counterintuitive` / `premise-audit` / `quantitative` / `open-depth`
- `difficulty`: 难度等级，1-5 整数
- `reference_points`: 参考答案要点，列表格式

---

## 输出 CSV：`questions.csv`

| 列名                | 类型  | 说明       | 示例                |
| ----------------- | --- | -------- | ----------------- |
| qid               | str | 题目唯一ID   | Q01               |
| category          | str | 题目类别     | logic             |
| difficulty        | int | 难度等级 1-5 | 4                 |
| question          | str | 题目文本     | 一个岛上有两种人...       |
| reference_points  | str | 参考答案要点   | 需要识别中项不周延;需要给出推理链 |

**注意**：`reference_points` 多个要点用分号 `;` 连接。

---

## `build_questions.py` 脚本功能

1. 扫描 `questions/` 目录下所有子目录中的 `.md` 文件
2. 解析每个文件的 YAML front matter（`id`, `category`, `difficulty`, `reference_points`）
3. 提取 front matter 之后的正文作为题目文本（`question`）
4. 将所有题目信息汇总，按 `qid` 排序
5. 输出到 `output/questions.csv`

**依赖**：`pyyaml` 或手动解析 front matter

---

## Release 规范

- **Tag 格式**：`questions-v{major}.{minor}`，如 `questions-v1.0`
- **Release 标题**：`Questions v1.0 - 12道评测题`
- **Release Asset**：`questions.csv`
- **Release Body**：注明题目数量、类别分布、难度范围

---

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
- 题目必须足够难，简单题无法体现引擎差异
- 前提审计题故意包含错误前提，测试引擎是否能让模型识别
- 反直觉题测试模型是否会走"捷径"
- 每道题有 `reference_points` 辅助评审，但无标准答案

---

## 操作流程

1. 在 TRAE IDE CN 中打开此仓库项目
2. 创建 `questions/` 目录下对应子目录的 .md 题目文件
3. 运行 `python scripts/build_questions.py` 生成 `output/questions.csv`
4. 验证 CSV 输出正确性（12行、字段完整）
5. `git add . && git commit -m "Add questions v1.0"`
6. `git tag questions-v1.0`
7. `git push && git push --tags`
8. 在 GitHub 上创建 Release，上传 `output/questions.csv` 作为 Asset

---

## 版本号约定

| 版本     | 含义                                |
| ------ | --------------------------------- |
| `v1.0` | 第一轮评测：12 道题                        |
| `v1.1` | 题目微调后的重测                          |
| `v2.0` | 第二轮评测：题目大幅更新                      |

每个 Release 的 Body 中需注明题目变更说明。

---

## 后续扩展

- 增加题目数量到 30 题
- 增加新的题目类别（如代码推理、多模态推理等）
- 增加难度分布的均衡性校验脚本
