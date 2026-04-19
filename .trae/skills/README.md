# Trae Skills

这个目录存放当前项目在 Trae 中使用的核心 skill。

## Design Principles

- 单一职责：每个 skill 只负责一个清晰流程，不互相越界
- 输入明确：每个 skill 都要明确自己依赖哪些输入
- 输出可验证：每个 skill 都要有稳定的输出契约
- 边界清晰：明确什么能做、什么不能做
- 交互稳定：用户选择优先可点击，且保持单轮单问
- 易于维护：规则变更时优先修改对应 `SKILL.md`

## Skill List

### `help`

- 作用：介绍项目怎么用，适合新用户快速上手
- 触发：`/help`，或用户询问“怎么用”“如何开始”“有哪些功能”
- 入口文件：`help/SKILL.md`

### `mock-interview`

- 作用：执行模拟面试，支持技术面试和综合面试
- 触发：`/mock-interview`，或用户要求模拟面试
- 关键边界：
  - 出题前先确认参数
  - 先参考联网高频题信号
  - 不直接使用本地 `题库/*.md` 作为题源
  - 每次只问 1 题
  - 用户回答后再给可点击选项
- 入口文件：`mock-interview/SKILL.md`

### `review-quiz`

- 作用：从本地题库抽题复习
- 触发：`/review-quiz`，或用户要求从已保存题库中抽题
- 关键边界：
  - 只使用本地 `题库/*.md`
  - 不联网生成新题
  - 错题优先，兼顾目录均衡
- 入口文件：`review-quiz/SKILL.md`

### `save-interview`

- 作用：将题目和答案保存到本地题库
- 触发：`/save-interview`，或用户要求保存当前题目 / 详细讲解
- 关键边界：
  - 保存到 `题库/*.md`
  - 先判断归档目录
  - 先检查相似题，再决定融合更新、追加新题或保持不变
- 入口文件：`save-interview/SKILL.md`

### `weak-analysis`

- 作用：分析题库覆盖情况和简历匹配度，输出薄弱点和补题优先级
- 触发：`/weak-analysis`，或用户要求分析薄弱点
- 关键边界：
  - 这是分析流程，不是出题流程
  - 可以联网看高频信号，但只用于分析
  - 不直接替代 `mock-interview` 或 `review-quiz`
- 入口文件：`weak-analysis/SKILL.md`

## Maintenance Notes

- 当前以 `.trae/skills` 为唯一主版本
- 后续新增或调整 skill 时，优先修改对应目录下的 `SKILL.md`
- 若某个流程的交互规则发生变化，先更新 skill，再测试对应命令行为
- 建议优先保持统一结构：`Mission -> Inputs -> Output Contract -> Workflow -> Guardrails -> Error Handling`
- 若新增 skill，优先补这几类信息：触发条件、输入依赖、输出格式、质量门槛、异常处理

## Governance Suggestions

- 先改契约，再改流程：优先更新 `Output Contract` 或 `Quality Bar`
- 交互问题优先查看 `Interaction Rules`、`Workflow`、`Post-Answer Rule`
- 能力越界问题优先查看 `Scope Boundaries`、`Core Principles`、`Do / Don't`
- 保存和题库问题优先查看 `save-interview` 的格式规范与回执约束
