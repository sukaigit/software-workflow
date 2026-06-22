# Hermes-标准化研发流程

> v1.0 | 顺序执行 × 分阶段交付
> 理论基础: Harness Engineering,不是让 AI 更聪明，而是设计约束环境让 AI 不犯错
> 三层分离：OpenSpec 管契约-说清楚你要什么 × Superpowers 管流程-教会 AI 怎么做 × agent-skill 管质量-定义什么算做好,跳过任意一层 → 翻车
> 三大支柱：
| 支柱 | 流程落地 | 解决的问题 |
|:-----|:---------|:-----------|
| **Context Engineering**（上下文工程） | 每阶段 promote 前注入完整 context（技能路径 + 执行步骤 + 上一阶段产出物）；每阶段产出物传递到下一个阶段 | Agent 不知道自己在哪个阶段、该干什么时的胡猜乱写 |
| **Architecture Constraints**（架构约束） | OpenSpec 契约锁死需求边界；.coding-taste.yaml 编码品味检查；组件库锁定样式选择 | Agent 自创 API/样式/架构的自由裁量权 |
| **Entropy Management**（熵管理） | 每周五 cronjob 自动清理；CI/CD 集成清理脚本 | Agent 留下的死代码/重复 import/废弃分支的累积债务 |

---

## 执行方式

项目经理（当前会话）逐阶段协调执行（直接执行或通过 delegate_task 子代理），每阶段完成后检查门禁，问用户确认后进入下一阶段。

---

## 通用规则（所有阶段遵守）

```
🔴 每阶段开始前 review 当前会话已加载的 CLAUDE.md 规则（重点：Workflow Compliance 第5条）
🔴 先 read_file 加载本阶段 skill 的 SKILL.md 全文后执行，不能凭记忆跳过
🔴 能力调用优先级：① OpenSpec CLI → ② Superpowers → ③ agent-skill → ④ 内建
🔴 每阶段完成：检查产出物 → openspec validate → 门禁打勾 → 问用户是否继续，不等确认不前进
🔴 门禁逐项检查，全部通过方可进入下一阶段
🔴 跨阶段修改：本阶段发现上一阶段产出物有问题 → 修改后**立即提交**，不积压不跳过后补（main 分支的修改直接提交 main；feat 分支的修改直接提交 feat）
```

---

## 异常处理

| 问题类型 | 第一步 | 第二步 | 第三步 |
|:---------|:------|:------|:------|
| 需求问题 | 直接问用户确认，修正 specs 后继续 | — | — |
| 架构/Plan问题 | 展示修改方案让用户确认 | 确认后直接修正继续 | — |
| 代码 Bug | 修 Bug，补回归测试 | 再次失败则回退到上一提交重试 | 中止，问用户方案 |
| 审查发现问题 | 修复后重新审查 | — | — |
| 门禁未通过 | 修复后重新检查 | 问用户是否放宽此项门禁 | 中止 |
| delegate_task 子代理卡住/崩溃 | 增加 context 清晰度后重试 | 改为在当前会话直接执行 | 中止，问用户方案 |
| 依赖安装失败 | 换源重试 | 问用户手动安装方式 | — |
| Skill 加载失败 | 检查路径是否存在 | 改用 read_file 直接加载 | 跳过该 skill，改内建能力执行 |
| 用户确认等待超时 | 再次提示 + 给出进展摘要 | — | — |

同一个问题修了 3 次还没过 → 中止，问用户"是否继续还是重新启动？"

---

## 项目初始化

### Step 0：问项目基本信息

先问项目名称（英文目录名），用户回答后，再问技术栈。

> 使用 GitHub：Step 1 会自动安装 gh CLI，需提前准备好 GitHub token 并执行 `gh auth login`

```
示例：
→ 项目名称是什么？
← project-work-log
→ 技术栈是什么？（如 后端 Spring Boot+MyBatis+MySQL / Flask+SQLite / Node.js+MongoDB，前端 原生HTML/JS / React / Vue）
← Spring Boot + MyBatis + MySQL
```

### Step 1：创建项目骨架

```bash
mkdir -p ~/workspace/{项目名} && cd ~/workspace/{项目名}
git init
cp ~/softwareWorkFlow/CLAUDE.md ./CLAUDE.md   # 再补充技术栈特定规则
cp ~/softwareWorkFlow/scripts/dot-coding-taste.yaml ./.coding-taste.yaml
mkdir -p docs/{requirements,reviews,retrospectives,plans,reports,project_docs,adr,templates} scripts
cp ~/softwareWorkFlow/scripts/taste-check.sh scripts/taste-check.sh && chmod +x scripts/taste-check.sh
cp ~/softwareWorkFlow/scripts/entropy-cleanup.sh scripts/entropy-cleanup.sh && chmod +x scripts/entropy-cleanup.sh
cp ~/softwareWorkFlow/docs-templates/标准项目文档模板合集.md docs/templates/
# 写入优化的 .gitignore（根据技术栈追加，避免构建产物/IDE配置/系统文件被提交）
cat > .gitignore << 'GITIGNORE'
.hermes/
target/
build/
*.class
*.jar
*.war
*.zip
*.log
*.tmp
.idea/
*.iml
.vscode/
.DS_Store
Thumbs.db
*.pyc
__pycache__/
node_modules/
.env
GITIGNORE
# 注意：根据实际技术栈（Java/Python/Node等）在初始化阶段已配好，后续不用再补
mkdir -p .github/workflows
echo "# ADR-001: {标题}" > docs/adr/ADR-001.md

# 熵管理 cronjob（每周五 17:00，脚本直跑，不经 Agent）
hermes cronjob create --name "$(basename $(pwd))-entropy-cleanup" \
  --schedule "0 17 * * 5" \
  --script scripts/entropy-cleanup.sh \
  --workdir "$(pwd)" \
  --no_agent > /dev/null 2>&1

# OpenSpec 初始化
openspec init --tools none

# 初始提交
git add . && git commit -m "chore: initial project scaffold"

# 检查并安装 gh CLI
which gh > /dev/null 2>&1 || { echo "正在安装 gh CLI..."; sudo apt install gh -y -qq 2>/dev/null; }

# 创建 GitHub 仓库并关联远程（需先登录 gh CLI）
gh repo create {项目名} --public --source=. --remote=origin --push 2>/dev/null || \
  echo "→ 请手动创建 GitHub 仓库后执行: git remote add origin <仓库地址> && git push -u origin main"
```

**🚪 门禁：**
```
□ 项目骨架已创建（git/CLAUDE.md/.coding-taste.yaml/OpenSpec）
□ 脚本文件已复制（scripts/taste-check.sh + scripts/entropy-cleanup.sh）
□ 熵管理 cronjob 已创建
□ GitHub 仓库已创建并 push（gh repo create 成功或手动配置 remote）
```

---

## 11 阶段执行流程（10 阶段 + 1 子阶段）

> 项目初始化仅首次执行。后续功能开发从阶段1开始，直接进入以下流程。

每个阶段由当前会话协调执行（阶段3通过 delegate_task 子代理执行开发，其余阶段直接执行），按顺序完成。每阶段开始前先 read_file 读取上一阶段的产出物作为输入。

### 阶段间交接物一览

| 交接 | 传入下阶段的产出物 |
|:----|:-------------------|
| 1→2 | `docs/requirements/`（需求简述）+ `openspec/changes/`（specs） |
| 2→3 | `docs/architecture/`（架构设计）+ `docs/adr/`（决策记录）+ `docs/plans/`（Plan）+ `sql/init.sql`（DDL）+ `docs/design/`（设计风格，有前端时）+ `openspec/changes/`（design.md 接口设计） |
| 3→4 | `src/`（业务代码）+ `test/`（测试代码） |
| 4→5 | `src/` + `test/` + `docs/reports/`（测试报告）+ `test/traceability-{功能名}.md`（追溯矩阵） |
| 4.5→5 | `docs/reviews/{功能名}-acceptance-{日期}.md`（验收报告） |
| 5→6 | `src/` + `test/` + `docs/`（含审查报告 `docs/reviews/`） |
| 6→7 | `.github/workflows/ci.yml`（CI/CD 配置就绪的完整代码） |
| 7→8 | 已归档 + 已 push 的代码 |
| 8→9 | `docs/retrospectives/`（回顾总结） |
| 9→10 | `docs/project_docs/`（全部必选文档 → .docx，可选按用户选择） |

> 注：设计风格相关产出物（`docs/design/`）仅在有前端时产出。

### 执行流程

1. 确认上一阶段产出物通过门禁
2. 加载本阶段 skill，按步骤执行
3. 产出本阶段交付物
4. 检查门禁 → 问用户是否继续
5. 继续 → 进入下一阶段

---

### 1. 需求分析

**加载 Superpowers：** `brainstorming` — `read_file /home/sukai/superpowers/skills/brainstorming/SKILL.md → 按步骤执行`
**加载 agent-skill：** `interview-me` — `read_file /home/sukai/agent-skills/agent-skills/skills/interview-me/SKILL.md → 按步骤执行`
**加载 agent-skill：** `idea-refine` — `read_file /home/sukai/agent-skills/agent-skills/skills/idea-refine/SKILL.md → 按步骤执行`

**执行步骤：**
```
1. 直接一问一答逐步澄清需求，每次只问一个问题
2. 确定功能名称，产出需求简述 → docs/requirements/{功能名}.md
3. openspec new change {功能名} → 写 proposal.md + specs/*/spec.md
4. 字段级细化：逐模块问用户确认每个字段（字段名/类型/约束/校验规则/默认值/可空性）
   - 有数据库的模块：逐表逐字段确认
   - 有接口的模块：逐接口确认请求参数和响应字段
   - 状态/角色等枚举值：确认可选值列表
   - 业务规则：确认字段间的约束关系（如：累计工时≥预算上限时禁止再报）
5. 边界条件清单：字段确认后，逐条问用户确认非正常场景的处理方式
   - 角色权限边界：每个角色能做什么、不能做什么（如：Admin 是否有审批权限？）
   - 数据边界：无数据时页面如何展示、超限/违规时如何提示用户
   - 异常流程：关键操作失败时的用户体验（网络超时、后端500、并发冲突）
   - 状态流转：每个状态变化的触发条件、谁可以触发、哪些状态不可逆
6. 产出字段级需求规格 → docs/requirements/{功能名}.md（含全部字段定义、权限矩阵和边界条件）
7. openspec validate --specs
```

**产出物：** docs/requirements/ + openspec/changes/{功能名}/

**🚪 门禁：**
```
□ 需求简述已产出（目标+范围+验收标准）
□ 字段级细化完成：每个模块的字段定义、校验规则、枚举值已逐项确认
□ 边界条件清单已逐项确认（角色权限/数据边界/异常流程/状态流转）
□ OpenSpec proposal + specs 已写入并通过 validate
□ 用户已确认需求
```
**门禁通过后提交：**
```bash
git add docs/requirements/ openspec/changes/
git commit -m "feat: {功能名} requirements (field-level)"
git push origin main
```

---

### 2. 架构设计

架构设计和写 Plan 无条件执行（所有项目必选）。设计风格仅在有前端时执行。

**先读取上一阶段产出物：**
read_file docs/requirements/{功能名}.md — 了解需求范围
read_file openspec/changes/{功能名}/proposal.md — 了解提案

**先产出系统架构设计：**
1. 产出 docs/architecture/architecture.md — 根据技术栈描述分层结构
   - Java/Spring Boot: Controller → Service → Mapper → DB
   - Python/Flask: Routes → Services → Models → DB
   - Node.js: Routes → Middleware → Services → DB
   - 前端项目: 组件树 → 状态管理 → API 层
2. 说明模块划分、组件关系、数据流、技术选型依据
3. 关键决策记入 docs/adr/ADR-{序号}.md
4. **有数据库：产出 DDL** → sql/init.sql（含所有表结构 + 预置数据 + 索引）
5. **创建数据库专用应用用户：**
   ```bash
   mysql -u root -p -e "CREATE USER IF NOT EXISTS '{应用名}'@'localhost' IDENTIFIED BY '{密码}';"
   mysql -u root -p -e "GRANT SELECT, INSERT, UPDATE, DELETE ON {数据库名}.* TO '{应用名}'@'localhost';"
   mysql -u root -p -e "FLUSH PRIVILEGES;"
   ```
   - 禁止使用 root 用户连接数据库（安全风险）
   - 应用用户仅授予所需库的增删改查权限，不授予 DDL 权限
   - 凭据记入 `docs/architecture/architecture.md` 的数据库设计章节
   - JDBC URL 使用 `localhost` 而非 `127.0.0.1`（MySQL JDBC 驱动将 127.0.0.1 解析为 localhost 导致认证失败）

**展示架构设计给用户确认：**
先展示 docs/architecture/architecture.md 给用户，说"架构设计已完成，请确认是否 OK？"
→ 用户确认后再继续。如有修改，修正后重新展示。

**然后执行设计风格（有前端时）：**
先问用户：这次功能有前端页面吗？→ 无则跳过
1. ls ~/awesome-design-md/design-md/ 选风格
2. cat 选中站点的 DESIGN.md 提取 Token（色板/字体/间距/圆角）→ 生成 docs/design/design.css
3. 写入 docs/design/design-guide.md
4. 有组件时写入 docs/design/component-library.md
5. 所有页面只能从组件库取样式

**展示设计风格给用户确认（有前端时）：**
展示 docs/design/design.css + docs/design/design-guide.md 给用户，确认风格是否满意。
→ 用户确认后再继续。如有修改，修正后重新展示。

完成上述架构设计（及设计风格，如有前端时）后，继续写 Plan。

**加载 Superpowers：** `writing-plans` — `read_file /home/sukai/superpowers/skills/writing-plans/SKILL.md → 按步骤执行`
**加载 agent-skill：** `planning-and-task-breakdown` — `read_file /home/sukai/agent-skills/agent-skills/skills/planning-and-task-breakdown/SKILL.md → 按步骤执行`

1. 画依赖图：数据库 → API → 前端
2. 垂直切片：每条功能完整路径一次做完
3. 每个任务 2-5 分钟，含目标/文件路径/代码片段/验证方式
4. **加载之前写的系统架构文档** — read_file docs/architecture/architecture.md，Plan 中的任务结构应与架构设计一致
5. **有数据库：产出 DDL** → sql/init.sql
6. **有 API：design.md 含完整接口设计** — 字段级定义（路径/方法/请求参数/响应结构/权限）
   - 每个接口写明完整请求体 JSON 结构（字段名/类型/是否必填/校验规则/示例值）
   - 每个接口写明完整响应 JSON 结构（字段名/类型/说明）
   - 状态码和错误码统一放在文档末尾
   - 参考模板：docs/requirements/{功能名}.md 中的字段定义
7. 有前端：每个Task注明组件类名（加载 docs/design/design-guide.md + docs/design/component-library.md）
8. 同步产出 OpenSpec design.md + tasks.md → validate
9. Plan → docs/plans/{slug}.md

**展示 Plan 给用户确认：**
展示 docs/plans/{slug}.md 给用户，确认任务切分和时间估算是否合理。
→ 用户确认后再进入门禁检查。如有修改，修正后重新展示。

**合并门禁：**
```
□ 系统架构文档已产出 docs/architecture/architecture.md
□ 架构文档中涉及的全部表已有字段级定义（字段名/类型/约束/关系）
□ 关键架构决策已记入 docs/adr/
□ 数据库 DDL 已产出 sql/init.sql（有数据库时）
□ 数据库专用应用用户已创建，凭据记入架构文档（有数据库时）
□ design.md 每个接口已包含字段级请求/响应结构（有 API 时）
□ docs/design/design.css 已生成（有前端时）
□ docs/design/design-guide.md 已写入（有前端时）
□ docs/design/component-library.md 已写入（有组件时，有前端时）
□ Plan 已产出 docs/plans/{slug}.md
□ OpenSpec design.md + tasks.md 已通过 validate
□ 用户已确认架构设计
□ 用户已确认风格（有前端时）
□ 用户已确认 Plan
```
**门禁通过后提交（main 分支）：**
```bash
git add docs/architecture/ docs/adr/ docs/design/ docs/plans/ openspec/changes/ sql/init.sql
git commit -m "feat: {功能名} architecture, design and plan"
git push origin main
```

阶段1阶段2期间如发现需求问题需修改 specs → **同步修改 docs/requirements/ 和 openspec/changes/ 后直接提交**，不积压改动。

---

### 3. 开发实施

**分支策略：** 开发前 `git checkout -b feat/{功能名}`，每个任务完成后 commit 到该分支。开发完成并本地测试通过后 push 到远程，后续通过 GitHub Pull Request 合并。

使用 `delegate_task` 按 Plan 切片逐任务 TDD 执行子代理。

**子代理 context 模板：**
```
使用 Superpowers TDD（RED→GREEN→REFACTOR）——全栈工程师，Plan：docs/plans/{slug}.md

先读取：docs/plans/{slug}.md（Plan）+ docs/architecture/architecture.md（架构）+ sql/init.sql（DDL）
加载：read_file /home/sukai/superpowers/skills/subagent-driven-development/SKILL.md → 按步骤执行
加载：read_file /home/sukai/agent-skills/agent-skills/skills/incremental-implementation/SKILL.md → 按步骤执行

Step 0: 执行 sql/init.sql 建库建表（未初始化时执行；如数据库已存在则跳过，避免重复建表报错）
⚠️ **禁止修改 `application.yml` 中的数据库用户名、密码、JDBC URL**（已在架构设计阶段配置好）。如需确认连接，用 `mysql -u {应用用户} -p -e "SELECT 1;"` 测试。不得以 root 身份连接数据库。
然后按 Plan 切片 TDD（RED→GREEN→REFACTOR），测试覆盖边界/中文/安全/分页等 8 类盲区
有前端时加载 docs/design/design-guide.md + docs/design/component-library.md，禁止自创样式
门禁：openspec validate 通过 + CLAUDE.md 合规 + 数据库已初始化（测试类门禁统一在阶段4检查）
```

**🚪 门禁（完成时检查）：**
```
□ openspec validate 通过
□ CLAUDE.md 合规自检通过
□ 数据库已初始化
```

---

### 4. 测试保障

**问用户：** 是否需要跑可选测试（变异测试/性能测试）。

加载 Superpowers：`test-driven-development` — `read_file /home/sukai/superpowers/skills/test-driven-development/SKILL.md → 按步骤执行`

1. **需求-测试追溯：** 逐条对照 `docs/requirements/{功能名}.md`，确保每行需求至少对应一条测试用例
   - 产出测试追溯矩阵 → `test/traceability-{功能名}.md`
   - 缺失测试覆盖的需求条目，补写测试用例后再继续（格式：需求条目 | 测试方法 | 测试类 | 状态）
2. 运行集成测试（端到端流程验证）
3. 覆盖率检查 ≥80%
4. 按用户要求执行变异测试/性能测试（可选）

有前端时，额外加载 `browser-testing-with-devtools` 并将 E2E 测试结论写入 `docs/reports/{功能名}-e2e-report.md`：
加载：read_file /home/sukai/agent-skills/agent-skills/skills/browser-testing-with-devtools/SKILL.md → 按步骤执行

```
E2E 盲区清单（有前端时）：
□ 入口路径: 根路由跳转逻辑（未登录→登录，已登录→首页）
□ 所有注册路由：遍历 ALL 路由，每条至少一个测试
□ UI 可见性：不同角色看到的按钮/链接是否正确
□ 设计规范类名：所有 <button> 含 btn 基类
□ 组件库类名：使用组件库类名，非内联 style
□ 弹窗表单：表单 fragment 不含 DOCTYPE
先写 E2E → 跑通 → 再标记完成
```

```
需要跑以下可选测试吗？
- 变异测试：复杂业务逻辑（审批流/权限矩阵）→ 推荐，标准 CRUD → 跳过
- 性能测试：用户操作手册需要性能数据时启用，测试工程师执行：
  - 页面加载时间 ≤ 3s（浏览器 DevTools 或 curl -w 测）
  - API 响应时间 ≤ 500ms（单接口，curl -w "%{time_total}"）
  - 并发 10 用户无错误（ab -n 100 -c 10 或 k6）
```

用户回答后，将可选测试的选择结论和测试结果写入 `docs/reports/{功能名}-test-report.md`。如果用户选了性能测试，测试工程师额外执行性能测试并记录结果供阶段9 生成报告。

**🚪 门禁：**
```
□ 需求-测试追溯清单已确认（每行需求至少对应一条测试用例）
□ 集成测试 PASS
□ 覆盖率 ≥ 80%
□ 变异测试（可选）PASS 或已记录
□ 性能测试（可选）PASS 或已记录
□ E2E 测试已编写并全部通过（有前端时）
```
**门禁通过后提交（feat 分支）：**
```bash
git add docs/reports/ test/
git commit -m "feat: {功能名} tests and reports"
git push origin feat/{功能名}
```

### 4.5. 功能验收

在进入审查前，先对照需求文档逐条验证功能可正常使用。功能验收由项目经理在当前会话直接执行，不可通过 delegate_task 子代理执行（AI 无法感知真实用户体验）。

**执行步骤：**
```
1. 读取 docs/requirements/{功能名}.md（需求规格含全部边界条件）
2. 读取 test/traceability-{功能名}.md（测试追溯矩阵）
3. 选择一条尚未验证的功能路径：
   a) API 测试：用 curl 调用接口，验证请求参数是否按需求响应
   b) 页面走查（有前端时）：打开浏览器，验证页面功能是否正常
   c) 边界场景：验证边界条件清单中的每个场景（权限/无数据/超限/异常/状态流转）
4. 发现问题 → 修复后重新验证 → 直至本功能路径通过
5. 选择下一条功能路径，重复步骤3-4，直至全部功能路径验证通过
```

**功能验收产出物：**
- 发现问题 → 修复后重走测试+提交 → 再进入功能验收重新验证
- 全部通过 → 产出 `docs/reviews/{功能名}-acceptance-{日期}.md`（验收清单，逐条标记 ✅/❌ 及修复记录）

**⚠️ 功能验收发现的 bug 必须在进入阶段5 之前修复完毕。禁止携带已知 bug 进入审查阶段。**

**🚪 门禁（功能验收阶段独立门禁，不通过不得进入阶段5）：**
```
□ 全部功能路径已逐条验收（API/页面/边界场景）
□ 验收报告已产出 docs/reviews/{功能名}-acceptance-{日期}.md
□ 0 个已知未修复 bug
```
**门禁通过后继续进入阶段5。无需提交（功能验收不产生代码变更，修复已在迭代中提交）。**

---

### 5. 审查

**阶段间顺序：** 先质量审查（5a）→ 再安全审查（5b）

> 如使用 GitHub，审查已在本阶段全部完成（含自动扫描+规范+质量+安全），PR 仅用于 Merge，无需重复 Review。

加载 Superpowers：`requesting-code-review` — `read_file /home/sukai/superpowers/skills/requesting-code-review/SKILL.md → 按步骤执行`
加载 agent-skill：`code-review-and-quality` — `read_file /home/sukai/agent-skills/agent-skills/skills/code-review-and-quality/SKILL.md → 按步骤执行`

**质量审查内部顺序不可换：** 自动扫描 → 规范审查 → 质量审查

自动扫描：运行以下三条安全工具 + 品味检查（阶段性快照，审查时发现问题当场修；阶段10做全量不变式检查，验证编码约束在迭代中未被侵蚀）
  - trivy fs --severity CRITICAL,HIGH .
  - semgrep --config=auto .
  - gitleaks detect --source . -v
  - bash scripts/taste-check.sh

规范审查：需求覆盖？Spec场景通过？超范围功能？边界情况？

质量审查（六维）：正确性/可读性/架构/安全/性能/品味 + 设计一致性

额外检查：E2E 测试是否存在并全部通过（有前端时，由测试工程师在阶段4完成）

产出：docs/reviews/{功能名}-{日期}.md

**阶段5a 完成后，进行阶段5b 安全审查：**

加载 agent-skill：`security-and-hardening` — `read_file /home/sukai/agent-skills/agent-skills/skills/security-and-hardening/SKILL.md → 按步骤执行`

1. SQL注入/XSS/JWT密钥/密码BCrypt/鉴权/密钥硬编码
2. 依赖版本检查：按技术栈执行（mvn versions:display-dependency-updates / npm outdated / pip list --outdated）
3. 无 CRITICAL 级别漏洞才能通过
4. 结论追加到 docs/reviews/{功能名}-{日期}.md

**🚪 门禁：**
```
□ 自动扫描：无 CRITICAL/HIGH 漏洞
□ 规范审查：PASS
□ 质量审查：PASS（六维）
□ 安全审查：PASS
□ E2E 测试已通过（有前端时）
□ 审查报告已写入 docs/reviews/
```

---

### 6. CI/CD

加载 agent-skill：`ci-cd-and-automation` — `read_file /home/sukai/agent-skills/agent-skills/skills/ci-cd-and-automation/SKILL.md → 按步骤执行`

创建 .github/workflows/ci.yml，包含编译→测试→扫描→打包→部署，配置 PR 触发事件：

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "添加编译→测试→扫描→打包步骤"
      - run: bash scripts/entropy-cleanup.sh
```

集成熵管理脚本

**🚪 门禁：**
```
□ CI/CD 配置已创建（编译→测试→扫描→打包）
□ 熵管理脚本已加入 CI 流程（.github/workflows/ci.yml 中包含 entropy-cleanup 步骤）
□ CI 已在 GitHub Actions 上至少触发一次（push/PR 后检查 Actions 运行日志；如因网络等问题无法触发，在审查报告中注明原因）
```

> ⚠️ 如果项目一直直接在 main 分支开发（不经 feat 分支 + PR 流程），CI 配置虽已创建但从未在 GitHub Actions 上实际运行过，无法确认流水线是否可正常工作。

---

### 7. 提交归档

阶段5 审查通过（已修复所有问题）且阶段6 CI/CD 配置就绪后执行：

先确认远程仓库已配置：
```bash
git remote -v || { echo "❌ 未配置远程仓库，请先执行: git remote add origin <仓库地址>"; exit 1; }
```

```bash
# 1. 推送功能分支到远程
#    （已在阶段3推过则跳过，如有新提交重新推送）
# 先用 openspec validate 检查，如果因 delta 格式问题失败（如 "no deltas" 错误），
# 直接用 openspec archive 强制归档（非阻塞警告）
openspec validate && openspec archive {功能名} -y || { openspec archive {功能名} -y; echo "⚠️ validate 跳过（delta 格式检查非阻塞）"; }
git push origin feat/{功能名}

# 2. 创建 Pull Request（触发 CI）
#    在 PR body 中注明期间对需求/架构的回溯修改：
#    - 需求变更（修改了哪些 docs/requirements/ 或 specs）
#    - 架构变更（修改了哪些 docs/architecture/ 或 design）
#    - 其他跨阶段修改
#    跨阶段修改的 commit 链接贴在 PR 评论区
gh pr create --title "feat: {功能名}" --body \
  "## 功能说明
详见 docs/requirements/{功能名}.md

## 回溯修改记录
- [需求变更] 无 / 见 commit {hash}
- [架构变更] 无 / 见 commit {hash}
- [其他] 无 / 见 comment" \
  --base main || echo "PR 已存在或手动创建"

# 3. 等待 CI 通过（阶段6配置的 workflow 自动执行）
#    审查已在阶段5完成，PR 无需等待 Approve

# 4. CI 通过后直接 Squash Merge 并删除远程分支
git checkout main && git pull origin main
gh pr merge --squash --delete-branch --subject "feat: {功能名}"
```

**自检：**
```
□ 每行改动对应需求
□ 无"顺手改"的相邻代码
□ 无孤儿代码
□ openspec validate + archive 已完成
```

---

### 8. 回顾总结

产出 → `docs/retrospectives/{功能名}.md`

```
# {功能名} 开发回顾
## 做了什么
## 踩过的坑 → [问题] → [方案]
## 下次改进
## 耗时： 计划 X 分钟 / 实际 Y 分钟
```

**流程回馈检查：** 逐条检查"踩过的坑"中是否涉及：
- 流程文档遗漏的门禁/步骤 → 更新 `Hermes-标准化研发流程.md`
- skill 未覆盖的常见陷阱 → 更新对应 skill 内容
- 工具使用痛点（路径错误、参数混淆等）→ 更新文档示例
- 通用规则缺失 → 补充到通用规则章节

产出后问用户：踩坑记录有需要回流到流程文档的内容吗？

**提交（main 分支）：**
```bash
git add docs/retrospectives/
git commit -m "feat: {功能名} retrospective"
git push origin main
```

---

### 9. 生成文档

全部文档（标 ◐ 为可选项，执行前问用户是否需要）：

| # | 文档 | 必选 | 数据来源 |
|:-:|:-----|:----:|:---------|
| 1 | 需求规格说明书 | ✅ | docs/requirements/ |
| 2 | 系统设计说明书 | ✅ | docs/architecture/ + docs/design/ + ADR |
| 3 | 接口设计说明书 | ✅ | `openspec/changes/`（接口定义 + design.md） |
| 4 | 数据库设计说明书 | ✅ | `sql/init.sql` |
| 5 | 单元测试报告 | ✅ | `docs/reports/{功能名}-test-report.md` |
| 6 | 集成测试报告 | ✅ | `docs/reports/` |
| 7  | **性能测试报告** | ◐ | `docs/reports/{功能名}-test-report.md` |
| 8 | 上线操作手册 | ✅ | `.github/workflows/ci.yml` + 部署配置 |
| 9 | 日常运维文档 | ✅ | 运维配置 |
| 10 | 用户操作手册 | ✅ | `docs/requirements/` |
| 11 | 架构决策记录 | ✅ | `docs/adr/` |

数据来源：各阶段已有产出物。模板见 `docs/templates/标准项目文档模板合集.md`。

**执行步骤：**

每份文档按以下流程生成：
1. 从 `docs/templates/标准项目文档模板合集.md` 中对应当前文档的模板结构
2. 从对应阶段产出物中提取数据填充
3. 写入 `docs/project_docs/{文档名}.md`
4. 全部写完后转 docx

```bash
# 转 docx
for f in docs/project_docs/*.md; do
  pandoc "$f" -o "${f%.md}.docx" --toc
done
```

**🚪 门禁：**
```
□ 所有必选文档已生成到 docs/project_docs/
□ 可选文档已按用户选择生成或跳过
□ 已转 .docx 格式
```
**提交（main 分支）：**
```bash
git add docs/project_docs/
git commit -m "feat: {功能名} project docs"
git push origin main
```

---

### 10. 熵管理维护

熵管理 cronjob 已在项目初始化时创建（每周五 17:00 自动执行 `scripts/entropy-cleanup.sh`）。

本阶段检查最近的熵管理报告，处理遗留问题：

```bash
# 查看最新报告（无报告则提示尚未执行）
ls -t docs/reports/entropy-*.md 2>/dev/null | head -1 || echo "熵管理报告尚未生成（cronjob 每周五执行）"
```

如有 CRITICAL 安全问题 → 通知用户处理。无问题则通过。

**品味不变式检查：** 运行品味检查脚本，验证 `.coding-taste.yaml` 中定义的编码约束未被破坏（命名规范、代码组织、模式一致性等）。这些约束在项目初始化时定义，熵管理阶段确保它们在项目演进中没有漂移。

```bash
bash scripts/taste-check.sh
```

**🚪 门禁：**
```
□ 最新熵管理报告已检查
□ 安全扫描无 CRITICAL 漏洞（或已创建修复任务）
□ 品味不变式已检查
```

---

### 总耗时

```
0.  项目初始化:         5-10 分钟
1.  需求分析:           8-15 分钟（含边界条件清单 3-5 分钟）
2.  架构设计:           5-10 分钟
3.  开发实施:           5-10 分钟/任务 × N
4.  测试保障:          10-15 分钟（含需求-测试追溯 3-5 分钟）
4.5 功能验收:          10-20 分钟
5.  审查:              10-15 分钟
6.  CI/CD:              5-10 分钟
7.  提交归档:           1-2 分钟
8.  回顾总结:           2-3 分钟
9.  生成文档:          10-15 分钟
10. 熵管理维护:         2-3 分钟
```
