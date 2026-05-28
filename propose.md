# HyperSpec:propose — 规格阶段

## 目标

把用户的需求从模糊想法变成可执行的任务清单。通过项目感知 + 调用原生 skill 产出完整的规格文档和实现计划。

## 阶段流程

### 1. 项目分析（仅首次运行）

如果 `.hyperspec-state.yaml` 不存在或 `project_profile` 为空，执行项目分析器：

**如果存在AGENTS.md文件并且文件内容不是空白：**
读取AGENTS.md作为输入，确认技术栈（语言、框架、构建工具）,编译/测试命令,项目结构, 检测CI配置 等

**如果AGENTS.md文件不存在或者AGENTS.md文件内容是空白：**
1. **探测语言和框架**：扫描源码目录的文件扩展名 + 读取依赖配置文件
2. **识别构建工具**：检测根目录的 `pom.xml`、`build.gradle`、`package.json`、`go.mod`、`Cargo.toml`、`pyproject.toml` 等
3. **推导编译/测试命令**：根据构建工具映射（见 SKILL.md "构建命令映射"表）
4. **判断项目结构**：单模块 vs 多模块/monorepo
5. **检测 CI 配置**：是否有 `.github/workflows/`、`.gitlab-ci.yml`、`Jenkinsfile` 等

分析完成后将 `project_profile` 写入 `.hyperspec-state.yaml`，并更新 `phase: propose`、`checkpoint: profiler-done`。

**命令运行时验证**：推导出 compile_command 后，必须实际运行一次验证其可用性。具体规则见 SKILL.md "命令运行时验证"段落。如果验证失败且为环境问题，立即阻塞并报告，等待用户提供正确的命令后再继续。

**空白项目（greenfield）处理：**
如果是空项目（无源码、无配置），跳过分析，project_profile 保持默认空值。在需求确认步骤中一并确定技术栈，分析完成后补充 project_profile。

### 2. 需求确认

**项目中没有实际代码**

  - 如果项目中没有实际的代码，调用原生 `grill-me` skill，与用户交互确认需求。

  **做法：**
  1. 宣布："调用 grill-me 确认需求"
  2. **空白项目（greenfield）额外确认：** 如果 Step 1 中检测到是空项目（project_profile 为空），在此步骤中一并确认技术栈（语言、框架、构建工具）。确认后补充 `project_profile` 并更新 `.hyperspec-state.yaml`。
  3. **如果用户提供了需求链接（如 JIRA 任务、GitHub Issue）：**先用 web reader 或其他方式获取需求详情，将完整需求作为本次需求的输入。
  4. 使用 Skill 工具调用 `grill-me`，args 格式：
  ```
    /grill-me <需求描述>
  ```
  **用途说明**：grill-me 的官方定位是「围绕已有 plan/design 追问」，但在 propose Step 2 中，我们将其用于**需求发现**——从模糊想法中提取技术选型、约束和验收标准。此时还没有 plan，grill-me 的追问机制正好帮助我们逐步明确需求。
  5. 从用户描述中提取变更名（英文短横线格式，如 `add-user-auth`）

**项目有实际代码**

  - 如果项目中有实际的代码，或者有CONTEXT.md文件存在。调用 `grill-with-docs` skill，与用户交互确认需求。

  **做法：**
  1. 宣布："调用 grill-with-docs 确认需求"
  2. **空白项目（greenfield）额外确认：** 如果 Step 1 中检测到是空项目（project_profile 为空），在此步骤中一并确认技术栈（语言、框架、构建工具）。确认后补充 `project_profile` 并更新 `.hyperspec-state.yaml`。
  3. **如果用户提供了需求链接（如 JIRA 任务、GitHub Issue）：**先用 web reader 或其他方式获取需求详情，将完整需求作为本次需求的输入。
  4. 使用 Skill 工具调用 `grill-with-docs`，args 格式：
  ```
    /grill-with-docs <需求描述>
  ```
  **用途说明**：grill-with-docs 在 grill-me 基础上增加了 domain awareness（读取 CONTEXT.md、挑战术语、创建 ADR）。用于需求发现时，这些功能同样有价值——已有项目通常有领域术语和文档，需求确认时应尊重已有定义。
  5. 从用户描述中提取变更名（英文短横线格式，如 `add-user-auth`）

**降级方案：** 如果 `grill-me` 或 `grill-with-docs` skill 不可用（Skill 工具返回 "Unknown skill"），手动执行等效操作：

1. 与用户交互确认需求，规则：一次只问一个问题、优先使用选择题、关注目的/约束/成功标准
2. 从用户描述中提取变更名（英文短横线格式，如 `add-user-auth`）
3. **空白项目（greenfield）额外确认：** 如果 Step 1 中检测到是空项目（project_profile 为空），在此步骤中一并确认技术栈（语言、框架、构建工具）。确认后补充 `project_profile` 并更新 `.hyperspec-state.yaml`。
4. **如果用户提供了需求链接（如 JIRA 任务、GitHub Issue）：** 先用 web reader 或其他方式获取需求详情，将完整需求作为本次需求的输入。
5. **自主模式：** 如果用户声明了自主执行意图（如「完全自主」「你决定」「不用确认」），跳过交互式需求确认，直接根据用户最初的需求描述进入 Step 3。空白项目自主模式下根据需求描述推断技术栈。

需求确认完成后，执行以下持久化操作：

1. **写入需求摘要文件**：将 Step 2 中确认的所有需求决策（技术选型、约束、验收标准等）整理为结构化摘要，写入项目根目录的 `.hyperspec-requirements.md`。格式示例：

```markdown
# 需求摘要：<变更名>

## 目标
<一句话描述本次变更要做什么>

## 决策记录
- <决策项1>: <选择> — <原因>
- <决策项2>: <选择> — <原因>
...

## 约束
- <约束1>
- <约束2>
...

## 验收标准
- <标准1>
- <标准2>
...
```

2. 更新 `.hyperspec-state.yaml`：`active_change: <变更名>`、`checkpoint: requirements-persisted`。

**为什么持久化**：grill-me 的对话上下文在后续步骤中可能丢失（特别是断点恢复或上下文压缩后）。写入文件确保 Step 3 和 Step 4 能获取完整需求，避免重复询问用户相同问题。

### 3. 调用 openspec-propose 生成规格文档

调用原生 `openspec-propose` skill，通过 CLI 创建变更并生成所有 artifacts。

**做法：**

1. 宣布："调用 openspec-propose 生成规格文档"
2. **读取需求摘要**：读取 `.hyperspec-requirements.md`，获取完整的需求决策信息
3. 使用 Skill 工具调用 `openspec-propose`，args 格式：
   ```
   Change name: <变更名>. Description: [项目: {project_profile.languages} + {project_profile.frameworks}, 构建: {project_profile.build_tool}] <用户原始需求>. 需求决策摘要: <从 .hyperspec-requirements.md 提取的完整需求决策内容，包括所有技术选型、约束、验收标准>
   ```
   **关键**：`<需求决策摘要>` 必须包含 `.hyperspec-requirements.md` 的**全部内容**，不是简略总结。这是避免 openspec-propose 重复询问已确认问题的核心机制。
4. 变更目录创建后，将 `.hyperspec-requirements.md` **移入** `openspec/changes/<变更名>/requirements.md`，使其成为正式 artifact。移动后删除项目根目录的 `.hyperspec-requirements.md`。
5. `openspec-propose` 会自动执行：
   - `openspec new change "<name>"` — 创建变更目录
   - `openspec status --change "<name>" --json` — 获取 artifact build order
   - 循环 `openspec instructions <artifact>` — 按依赖顺序生成每个 artifact
   - 产出：proposal.md、design.md、specs/、tasks.md
6. 等待 skill 完成后，验证 `openspec/changes/<变更名>/` 下所有 artifacts 已生成且**非空**（包括 `requirements.md`）。如果某个 artifact 文件为空或不存在，重新调用 openspec-propose 补充缺失的 artifact。
7. **覆盖 openspec-propose 的执行建议**：openspec-propose 完成后会建议 "Run /opsx:apply"。在 HyperSpec 上下文中忽略此建议，宣布 "回到 HyperSpec 流程，继续生成实现计划"

**降级方案：** 如果 `openspec-propose` skill 不可用（Skill 工具返回 "Unknown skill"），手动执行等效操作：
1. 读取 `.hyperspec-requirements.md` 获取完整需求摘要
2. 运行 `bunx @fission-ai/openspec new change "<变更名>"` 创建变更目录（如果 bunx 失败，改用 `npx @fission-ai/openspec new change "<变更名>"`）
3. 将 `.hyperspec-requirements.md` 移入 `openspec/changes/<变更名>/requirements.md`
4. 运行 `bunx @fission-ai/openspec status --change "<变更名>" --json` 获取 artifact build order（如果 bunx 失败，改用 `npx @fission-ai/openspec status --change "<变更名>" --json`）
5. 对每个 artifact，运行 `bunx @fission-ai/openspec instructions <artifact>` 获取生成指令（如果 bunx 失败，改用 `npx @fission-ai/openspec instructions <artifact>`）
6. 按照指令内容，使用 Write 工具直接创建对应的 artifact 文件（proposal.md、design.md、specs/ 下的 .md 文件、tasks.md）——**生成时必须参考 requirements.md 中的需求决策，确保 artifacts 反映已确认的需求**
7. 确保每个 artifact 文件非空且内容完整

完成后更新 `.hyperspec-state.yaml`：`checkpoint: openspec-generated`。

### 4. 调用 writing-plans 生成实现计划

调用 Superpowers 的 `writing-plans` skill，传入 openspec artifacts + project_profile 作为上下文。

**做法：**

1. 宣布："调用 writing-plans 生成实现计划"
2. **准备上下文**：读取以下文件并构造上下文字符串：
   - `openspec/changes/<变更名>/requirements.md` — **需求决策摘要（来自 Step 2 grill-me 确认）**
   - `openspec/changes/<变更名>/proposal.md` — 需求背景和目标
   - `openspec/changes/<变更名>/design.md` — 技术方案
   - `openspec/changes/<变更名>/specs/` — 规格增量（所有 .md 文件）
   - `openspec/changes/<变更名>/tasks.md` — 任务清单（**作为权威任务分解，writing-plans 应基于此展开**）
3. **准备 API 验证上下文**：扫描项目中已有的同类代码，提取关键框架 API 的实际签名，作为 writing-plans 的额外约束。方法：在项目中找到与本次变更同层的已有文件（如同模块的 Controller、Service、Handler 等），读取其中对框架工具类、第三方库的实际调用方式，重点关注：
   - 框架提供的工具方法是否存在多个重载（参数个数/类型不同），确认计划中应使用哪个签名
   - 第三方库的 import / require 路径是否与直觉不同（如包名重组后的新路径）
   - 构建工具在项目实际环境中的可用命令（如私有仓库可能限制增量编译，必须全量编译）
   - 将验证结果整理为"框架 API 注意事项"列表，附加到 writing-plans 的 args 中
4. 使用 Skill 工具调用 `superpowers:writing-plans`，args 传入上下文：
   ```
   变更名: <name>
   项目技术栈: {project_profile.languages} + {project_profile.frameworks}
   构建工具: {project_profile.build_tool}
   测试框架: {project_profile.test_command}
   项目结构: {project_profile.structure}
   编译命令: {project_profile.compile_command}
   计划保存路径: superpowers/plans/YYYY-MM-DD-<变更名>.md
   编译约束:
   - 本项目的编译命令为 {compile_command}，计划中每个 Task 完成后必须使用此命令验证编译
   - 接口层和实现层分开定义在不同的 Task 中会导致单独编译失败，必须在同一 Task 中同时修改接口和实现
   checkbox 唯一性约束:
   - 计划中每个 Step 的 checkbox 描述必须全局唯一，不能仅靠步骤编号区分（如"Step 2: 编译验证"会在多个 Task 中重复）
   - 推荐格式：`- [ ] **Step N: <动作描述>（Task <编号>）**` 或在步骤描述中包含 Task 特有的上下文（如文件名、类名）
   - 这确保 apply 阶段使用 Edit 工具更新 checkbox 时，old_string 在文件中唯一匹配
   框架 API 注意事项:
   <Step 3 中验证的实际 API 签名列表>
   openspec artifacts:
   <将上述文件内容拼接>
   ```
   将完整 project_profile 信息、编译约束和 API 验证结果传入，让 writing-plans 从一开始就生成正确的任务分解，避免事后合并。
5. writing-plans 会读取上下文，生成实现计划（File Structure 表 + 带 checkbox 的 TDD 微步骤），保存到 `superpowers/plans/YYYY-MM-DD-<变更名>.md`
6. **跳过执行移交**：writing-plans 完成后会 offer 执行选择（Subagent-Driven / Inline）。在 HyperSpec 上下文中，跳过此 offer，直接回到本阶段 Step 5。
7. 确认 `superpowers/plans/` 下有计划文件且包含至少 1 个 checkbox。如果没有 checkbox，说明 writing-plans 未能基于 tasks.md 展开步骤，需要重新执行并更明确地指定 "按 tasks.md 中的每个 Task 展开为 TDD 微步骤"。
8. **绑定变更名**：在计划文件开头添加一行注释 `<!-- hyperspec change: <变更名> -->`，用于 SKILL.md 状态检测时确认计划文件与活跃变更的对应关系。
9. **计划质量审查（必须完成）**：逐项检查 writing-plans 生成的计划，确认以下维度全部通过。此步骤不可跳过，即使自主模式也必须执行：
   - **编译约束遵守**：接口+实现是否在同一 Task 中。如果 writing-plans 仍将接口和实现拆分为不同 Task，手动合并并标注原因
   - **import 语句正确性**：检查计划中代码块的 import 语句是否与项目实际路径一致（对比 Step 3 中扫描的已有代码的 import 路径）
   - **无效代码检查**：检查代码块中是否有调试残留、空行占位、无意义的方法调用（如 `.getClass()` 仅用于触发泛型推断）
   - **类型一致性**：计划中后出现的 Task 引用的类型、方法签名是否与前面 Task 定义的一致
   - 如果发现问题，直接在计划文件中修复，修复后重新确认编译依赖
10. **立即更新 checkpoint**：writing-plans 完成且 Step 7-9 验证通过后，**立即**更新 `.hyperspec-state.yaml`：`checkpoint: plan-generated`。这一步**不可跳过**，即使用户声明自主模式也必须先写 `plan-generated`，再在 Step 5 中更新为 `plan-generated-and-confirmed`。确保每个 checkpoint 都是原子性推进，避免中断时状态文件与实际文件不一致。

### 5. 用户确认

展示产出摘要，请用户确认「规格 OK，可以开始实现」：

- 变更名和位置
- 项目 profile 信息（语言、框架、构建工具）
- 生成的 artifacts 列表
- 实现计划中的 Task 数量和关键步骤
- 预计涉及修改的文件列表

## 出口条件

- `openspec/changes/<变更名>/` 下有完整的 artifacts（由 openspec-propose 生成）
- `superpowers/plans/` 下有对应的实现计划文件（由 writing-plans 生成）
- 计划文件中至少有 1 个 checkbox
- 用户明确确认可以进入构建阶段，或用户已声明自主执行模式

出口时更新 `.hyperspec-state.yaml`：`phase: apply`、`checkpoint: plan-generated-and-confirmed`。

## 断点恢复

重新运行时，先读取 `.hyperspec-state.yaml` 的 checkpoint，然后检查实际文件状态：

| checkpoint | 实际文件状态 | 恢复到 |
|-----------|-------------|--------|
| `profiler-done` | 无活跃变更目录 | Step 2（需求确认） |
| `requirements-confirmed` | `.hyperspec-requirements.md` 不存在 | Step 2（补充持久化需求摘要，然后进入 Step 3） |
| `requirements-persisted` | 变更目录不存在 | Step 3（调用 openspec-propose） |
| `requirements-persisted` | 变更目录存在但 artifacts 不完整或有空文件 | Step 3（openspec-propose 会自动补充） |
| `openspec-generated` | artifacts 完整但 `superpowers/plans/` 无计划文件 | Step 4（调用 writing-plans） |
| `plan-generated` | 计划文件存在但无 checkbox | Step 4（重新调用 writing-plans） |
| `plan-generated` | 计划文件存在且有 checkbox | Step 5（用户确认） |
| `plan-generated-and-confirmed` | 计划文件存在且有 checkbox | 出口到 apply 阶段（`phase: apply, checkpoint: plan-generated-and-confirmed`） |
| `plan-generated-and-confirmed` | 计划文件不存在或无 checkbox | 回退到 Step 4（重新生成计划） |

**旧版兼容**：如果状态文件 checkpoint 为 `requirements-confirmed` 且 `.hyperspec-requirements.md` 已存在，视为等价于 `requirements-persisted`。

**状态文件缺失时的降级**：如果没有 `.hyperspec-state.yaml`，回退到文件扫描方式（按上述表格右列的实际文件状态判断）。

## 硬门

本阶段**禁止写任何代码或创建分支**。任何实现行为必须在 apply 阶段进行。
