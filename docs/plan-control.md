# 仓库3：Control（对照组仓库）实现计划

## 仓库名称

`reasoning-evaluation-control`

## 职责

**不包含任何引擎规则**，从题目仓库的 GitHub Release 获取题目，让模型在标准助手行为下逐题作答，生成标准风格的回答，通过 GitHub Release 发布带 label 的答案（label 固定为 `"C"`）。

---

## 仓库结构

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

---

## 输出 CSV：`answers-labelC-v{ver}.csv`

| 列名     | 类型  | 说明                | 示例         |
| ------ | --- | ----------------- | ---------- |
| qid    | str | 题目ID（关联questions） | Q01        |
| label  | str | 固定值"C"            | C          |
| answer | str | 对照组回答文本           | A是无赖，B是骑士。 |

---

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

**注意**：此脚本与实验组的 `fetch_questions.py` 功能相同，可复用代码。

---

## Release 规范

- **Tag 格式**：`control-v{major}.{minor}`，如 `control-v1.0`
- **Release 标题**：`Control v1.0 - Standard Assistant Answers`
- **Release Asset**：`answers-labelC-v{ver}.csv`
- **Release Body 需注明**：
  - 使用的题目版本（如 `questions-v1.0`）
  - 使用的 LLM 模型名称（需与实验组相同）
  - 生成回答的日期

---

## ⚠ 版本号匹配要求

**实验组和对照组的版本号必须相同**（如 `v1.0`），才能进行对比。评审仓库在获取数据时会校验版本号一致性。

| 实验组 Tag                   | 对照组 Tag              | 是否可对比 |
| ---------------------------- | ---------------------- | -------- |
| `experimental-v1.0`          | `control-v1.0`         | ✅        |
| `experimental-v1.0`          | `control-v1.1`         | ❌        |
| `experimental-v2.0`          | `control-v2.0`         | ✅        |

---

## 操作流程

1. 在 TRAE IDE CN 中打开此仓库项目（**无引擎规则加载**）
2. 运行 `python scripts/fetch_questions.py --version questions-v1.0` 下载题目
3. 验证 `input/questions.csv` 已正确下载（12行）
4. 在 TRAE IDE CN 对话中要求模型逐题作答（标准助手行为，无任何额外规则）
5. 将模型生成的回答整理写入 `answers/answers-labelC-v1.0.csv`
   - 每行包含 `qid`, `label`（固定为 `C`）, `answer`（模型回答文本）
   - 确保 12 道题全部作答
6. `git add answers/answers-labelC-v1.0.csv`
7. `git commit -m "Add control answers v1.0"`
8. `git tag control-v1.0`
9. `git push && git push --tags`
10. 在 GitHub 上创建 Release，上传 `answers/answers-labelC-v1.0.csv` 作为 Asset

---

## 版本号约定

与题目仓库和实验组版本号对齐，同一轮评测使用相同的 `{major}.{minor}`：

| 版本     | 含义                                |
| ------ | --------------------------------- |
| `v1.0` | 第一轮评测：12 道题                        |
| `v1.1` | 题目微调后的重测                          |
| `v2.0` | 第二轮评测：题目大幅更新                      |

---

## 防偏差要点

- **绝对不要**在此仓库中创建 `.trae/rules/` 目录或任何形式的规则文件
- 模型在做题时应处于标准助手模式，不受任何额外规则影响
- 使用的 LLM 模型必须与实验组**完全相同**（仅区别在于有无引擎规则）
- 确保在独立的 TRAE IDE CN 工作区中打开此仓库，避免与实验组或评审组的上下文互相污染

---

## 后续扩展

- 增加自动化回答生成脚本（通过 API 调用模型作答）
- 增加回答质量预检脚本（检查是否所有题目都已回答、回答长度是否合理等）
