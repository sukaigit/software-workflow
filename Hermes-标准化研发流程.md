# Hermes-标准化研发流程

> v1.2 | 7 阶段流水线 × 逐阶段门禁检查
> 核心理念：Harness Engineering，不是让 AI 更聪明，而是设计约束环境让 AI 不犯错
> 三层分离：OpenSpec 管"契约"（说清楚要什么）→ Superpowers 管"流程"（教会怎么做）→ agent-skill 管"质量"（定义什么算好）
> 门禁机制：每阶段完成逐项检查门禁 → 全部通过才问用户确认 → 不等确认不前进
> 生命周期：需求分析 → 架构设计 → 开发实施 → 测试保障 → 审查 → 提交归档 → 回顾总结
| 支柱 | 流程落地 | 解决的问题 |
|:-----|:---------|:-----------|
| **Context Engineering**（上下文工程） | 每阶段 promote 前注入完整 context（技能路径 + 执行步骤 + 上一阶段产出物）；每阶段产出物传递到下一个阶段 | Agent 不知道自己在哪个阶段、该干什么时的胡猜乱写 |
| **Architecture Constraints**（架构约束） | OpenSpec 契约锁死需求边界；.coding-taste.yaml 编码品味检查；组件库锁定样式选择 | Agent 自创 API/样式/架构的自由裁量权 |
| **Entropy Management**（熵管理） | 每周五 cronjob 自动执行清理脚本 | Agent 留下的死代码/重复 import/废弃分支的累积债务 |

---

## 执行方式

项目经理（当前会话）逐阶段协调执行（直接执行或通过 delegate_task 子代理），每阶段完成后检查门禁，问用户确认后进入下一阶段。

---

## 通用规则（所有阶段遵守）

```
🔴 每阶段开始前 review 当前会话已加载的 CLAUDE.md 规则（重点：Workflow Compliance 第5条）
🔴 先 read_file 加载本阶段 skill 的 SKILL.md 全文后执行，不能凭记忆跳过
🔴 能力调用优先级：① OpenSpec CLI → ② Superpowers → ③ agent-skill → ④ 内建
🔴 每阶段完成：检查产出物 → openspec validate → 逐项检查门禁，全部通过 → 问用户是否继续，不等确认不前进
🔴 跨阶段修改：本阶段发现上一阶段产出物有问题 → 修改后**立即提交**，不积压不跳过后补（main 分支的修改直接提交 main；feat 分支的修改直接提交 feat）

**每阶段执行流程：**
1. 确认上一阶段产出物通过门禁
2. 加载本阶段 skill，按步骤执行
3. 产出本阶段交付物
4. 逐项检查门禁，全部通过后问用户是否继续
5. 继续 → 进入下一阶段
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

先问项目名称（中文以及英文目录名），用户回答后，再问技术栈，再问数据库类型，最后问 GitHub 仓库可见性。

```
示例：
→ 项目名称是什么？
← 项目工时管理系统 project-time-manage
→ 技术栈是什么？（后端: Spring Boot+MyBatis / Flask / FastAPI 等；前端: 原生HTML/JS / React / Vue 等）
← Spring Boot + MyBatis + 原生HTML/JS
→ 数据库类型是什么？（MySQL / SQLite 等）
← MySQL
→ GitHub 仓库可见性？（public/private）
← private
```

**展示给用户确认：** 汇总项目信息给用户——"项目名称：xxx，技术栈：xxx，数据库类型：xxx，仓库可见性：xxx，确认无误？"
→ 用户确认后进入 Step 1。如有修改，修正后重新展示确认。

**🚪 门禁：**
```
□ 项目名称和技术栈已确定
□ 数据库类型已确定
□ 仓库可见性已确定（public/private）
□ 用户已确认项目信息
```

### Step 1：创建项目骨架

> 使用 GitHub 时，Step 1 会自动检查 gh CLI 是否已安装并完成认证，未登录则提示用户执行 `gh auth login`

```bash
mkdir -p /d/hermes/workspace/{项目名} && cd /d/hermes/workspace/{项目名}
git init
cp /d/hermes/softwareWorkFlow/CLAUDE_CN.md ./CLAUDE.md   # 中英文行为准则，再补充技术栈特定规则
cp /d/hermes/softwareWorkFlow/scripts/dot-coding-taste.yaml ./.coding-taste.yaml
mkdir -p docs/{reviews,retrospectives,plans,reports,adr,design,prototype,architecture,api,database} sql scripts
cp /d/hermes/softwareWorkFlow/scripts/taste-check.sh scripts/taste-check.sh
cp /d/hermes/softwareWorkFlow/scripts/entropy-cleanup.sh scripts/entropy-cleanup.sh
# 写入 .gitignore（通用条目 + 按 Step 0 技术栈追加）

# 通用条目（所有项目用）
cat > .gitignore << 'GITIGNORE'
.hermes/
*.zip
*.log
*.tmp
.vscode/
.DS_Store
Thumbs.db
.env
GITIGNORE

# Spring Boot+MyBatis 追加：
cat >> .gitignore << 'GITIGNORE'
target/
build/
*.class
*.jar
*.war
.idea/
*.iml
GITIGNORE

# Flask / FastAPI 追加：
cat >> .gitignore << 'GITIGNORE'
*.pyc
__pycache__/
GITIGNORE

# React / Vue 追加：
cat >> .gitignore << 'GITIGNORE'
node_modules/
GITIGNORE

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
which gh > /dev/null 2>&1 || { echo "⚠️ gh CLI 未安装，请执行: winget install GitHub.cli"; }

# 创建 GitHub 仓库并关联远程（需先登录 gh CLI）
gh repo create {项目名} --{可见性} --source=. --remote=origin --push 2>/dev/null || {
  echo "⚠️ gh repo create 失败（可能是网络问题或仓库已存在）"
  echo "  手动处理："
  echo "  1. 浏览器打开 https://github.com/new 创建仓库 {项目名}"
  echo "  2. 执行: git remote add origin <仓库地址>"
  echo "  3. 执行: git push -u origin main"
  echo "  如果网络不通，可在网络恢复后手动 push"
}
```

**🚪 门禁：**
```
□ 项目骨架已创建（git/CLAUDE.md/.coding-taste.yaml）
□ OpenSpec 已初始化（`ls openspec/specs/` 目录存在）
□ 脚本文件已复制（scripts/taste-check.sh + scripts/entropy-cleanup.sh）
□ 项目目录结构完整（docs/ 全部子目录 + sql/ + scripts/ 已创建）
□ 熵管理 cronjob 已创建
□ GitHub 仓库已创建并 push（gh repo create 成功或手动配置 remote）
```

---

### 1. 需求分析

**加载 Superpowers：** `brainstorming` — `read_file D:\hermes\superpowers\skills\brainstorming/SKILL.md → 按步骤执行`
**加载 agent-skill：** `interview-me` — `read_file D:\hermes\agent-skills\skills\interview-me/SKILL.md → 按步骤执行`
**加载 agent-skill：** `idea-refine` — `read_file D:\hermes\agent-skills\skills\idea-refine/SKILL.md → 按步骤执行`

**执行步骤：**
```
1. 直接一问一答逐步澄清需求，每次只问一个问题
2. 确定功能名称，立即写入 OpenSpec 主规范：
   - 在 `openspec/specs/{domain}/spec.md` 中编写 **Requirement + Scenario**
   - ⚠️ **每个功能独立一个 spec 文件，禁止将所有需求合并写入单个文件**
   - 主规范格式见 OpenSpec 校验要求（`## Purpose` + `## Requirements` + `### Requirement:` + `#### Scenario:`）
   - ⚠️ **每条 Requirement 的正文必须包含 SHALL 或 MUST 关键词**，否则 validate 报错
3. **字段级细化、边界条件与非功能需求：** 按功能每确认一个就一次性问完以下内容并写入 Scenario
   - **字段定义：** 字段名/类型/约束/校验规则/默认值/可空性（有数据库时逐表逐字段，有接口时逐接口确认参数和响应字段，枚举值确认可选值列表，业务规则确认字段间约束关系）
   - **边界条件：** 角色权限边界（每个角色能做什么/不能做什么）、数据边界（无数据展示方式/超限违规提示）、异常流程（超时/500/并发冲突）、状态流转（触发条件/谁可触发/不可逆状态）
   - **非功能需求：** 性能（页面加载时间/API响应时间/并发/数据量级）、安全（密码策略/会话超时/鉴权/验证码/防注入）、浏览器兼容性（支持范围）、可用性（校验时机/确认弹窗/Toast规范）
   - 无可写"无特殊要求"
4. **设计风格与页面原型（有前端时）：** 定义风格 → 基于实际技术栈生成原型 → 一次展示确认
   - `ls D:\\\\hermes\\\\awesome-design-md\\\\design-md\\\\` 选风格
   - 提取 Token（色板/字体/间距/圆角）→ 生成 `docs/design/design.css` + `docs/design/design-guide.md`
   - 有组件时写入 `docs/design/component-library.md`
   - **将设计文件复制到原型目录**：`cp docs/design/design.css docs/prototype/css/design.css`
   - 在 `docs/prototype/` 下 scaffold 项目（Vite + framework 或原生 HTML/JS）
   - 安装依赖，配置路由（每个功能页面一条路由）
   - 按页面编写组件，应用设计风格 Token
   - 覆盖：登录页 + 所有功能页面 + 弹窗/对话框 + 图表占位
   - 启动 dev server：`npm run dev`
   - **让用户浏览器打开翻看，确认风格+页面布局+交互**（一次性确认）
   - 发现问题 → 调整 openspec/specs/{domain}/spec.md → 反复至用户确认
   - 提交 `docs/prototype/` 到 git 并 `git tag prototype-v1`
5. `openspec validate --specs` — 最终规范校验，确保全部通过
```

**产出物：** openspec/specs/（主规范）+ docs/design/（设计风格，有前端时）+ docs/prototype/（页面原型，有前端时）

**🚪 门禁：**
```
□ **每个功能独立 spec 文件：** openspec/specs/ 下每个域各自一个 spec.md，无合并式大文件
□ 页面原型已产出并确认（有前端时，含设计风格与交互）
□ **OpenSpec 验证：** 执行以下命令，全部通过才算通过本门禁
  ```bash
  ls openspec/specs/*/spec.md        # 每个域有独立 spec 文件
  openspec validate --specs          # 全部 specs 通过校验
  ```
□ **Spec 与原型同步：** 原型阶段的需求调整必须同步更新 OpenSpec spec 文件，不可只改原型不改 spec
```

**⚡ Phase 1 原型阶段常见调整（经验汇总）：**

| 调整类型 | 实例 | 影响 |
|:---------|:-----|:-----|
| **模块名变更** | 工时填报→工时管理 | 同步更新 spec + prototype + 权限树 |
| **功能拆分** | 工时统计→拆为员工/项目两个子菜单 | 新增 route + render + permission |
| **字段增删** | 项目加「所属部门」、任务去「分配人员」 | 同步更新 spec + prototype + page2Data |
| **业务逻辑** | 工时仅已驳回可编辑、审批按角色过滤 | 原型条件渲染 + spec Scenario 更新 |
| **数据驱动** | 角色/用户下拉从 page2Data 动态加载 | 不要硬编码 option 列表 |
| **权限细化** | 审批链：部门经理→项目经理，项目经理→普通员工 | 影响原型数据过滤逻辑 |
**门禁通过后提交：**
```bash
git add openspec/specs/ docs/design/ docs/prototype/
git commit -m "feat: {功能名} requirements"
git push origin main
```

---

### 2. 架构设计

**先读取上一阶段产出物：**
read_file openspec/specs/{domain}/spec.md — 了解需求范围
read_file docs/design/design-guide.md — 了解设计风格（有前端时）
read_file docs/design/component-library.md — 了解组件库（有组件时，有前端时）
**read_file docs/prototype/index.html（有前端时，有原型时）— 提取页面结构、数据模型（page2Data）、路由（routes）、筛选条件、表单字段，作为架构设计的输入**

**⚡ 原型 → 架构映射规则（经验）：**
```
原型中的 page2Data 字段定义 → 数据库表字段
原型 renderXxx() 的筛选条件栏 → API GET 查询参数
原型 showXxxForm() 的表单字段 → API POST/PUT 请求体
原型 routes 对象 → Controller 模块划分
原型 rolePermissions 对象 → RBAC 权限表结构
原型分页逻辑 (pageState + pagination()) → 后端分页 API
```

**顺序执行以下设计任务（每份完成后自动进入下一份）：**

**① 系统架构设计 → docs/architecture/architecture.md**

1. 根据技术栈描述分层结构
   - **Java/Spring Boot（MyBatis）：** Controller → Service → Mapper → DB
   - **Python/Flask：** Routes → Services → Models → DB
   - **Python/FastAPI：** Routers → Services → Models → DB
   - **前端项目：** 组件树 → 状态管理 → API 层
2. 说明模块划分、组件关系、数据流、技术选型依据
3. 关键决策记入 docs/adr/ADR-{序号}.md
4. **创建数据库专用应用用户（仅所选数据库类型需要时执行）：**
   - **MySQL：**
     ```bash
     mysql -u root -p -e "CREATE USER IF NOT EXISTS '{应用名}'@'localhost' IDENTIFIED BY '{密码}';"
     mysql -u root -p -e "GRANT SELECT, INSERT, UPDATE, DELETE ON {数据库名}.* TO '{应用名}'@'localhost';"
     mysql -u root -p -e "FLUSH PRIVILEGES;"
     ```
     - JDBC URL 使用 `localhost` 而非 `127.0.0.1`（MySQL JDBC 驱动将 127.0.0.1 解析为 localhost 导致认证失败）
   - **SQLite / 嵌入式数据库：** 无需创建应用用户，跳过此步骤

**通用规则（仅需要创建数据库用户时适用）：**
- 禁止使用 root 等超级用户连接应用数据库
- 应用用户仅授予所需库的增删改查权限，不授予 DDL 权限
- 凭据记入 `docs/architecture/architecture.md` 的数据库设计章节

5. 同步产出 OpenSpec design.md 中的架构设计引用：
   ```markdown
   # 系统架构
   系统架构详见 [docs/architecture/architecture.md](../../docs/architecture/architecture.md)
   ```

**② API 接口设计 → docs/api/接口定义.md**

1. 读取 `openspec/specs/{domain}/spec.md`（需求字段定义）+ `docs/architecture/architecture.md`（模块划分），列出全部接口清单
2. 每个接口写明：
   - 请求方式（GET/POST/PUT/DELETE）
   - URL 路径（含路径参数，如 `/api/projects/{id}`）
   - 请求参数：
     - **POST/PUT：** JSON 请求体结构（字段名/类型/是否必填/校验规则/示例值）
     - **GET/DELETE：** query string 参数（参数名/类型/是否必填/说明）
   - 响应体 JSON 结构（字段名/类型/说明）
   - 权限要求（哪些角色可以访问）
3. 状态码和错误码统一放在文档末尾
4. 同步产出 OpenSpec design.md 中的接口设计引用：
   ```markdown
   # 接口设计
   API 接口定义详见 [docs/api/接口定义.md](../../docs/api/接口定义.md)
   ```
5. openspec validate → 检查 proposal + specs + design.md 结构完整性

**③ 数据库设计 → docs/database/数据库设计.md + sql/init.sql**

1. 读取上一阶段产出物（specs + 架构文档 + API 接口定义），设计全部表结构
2. 产出数据库设计说明书 → docs/database/数据库设计.md：
   - 表名、字段名、类型、约束、默认值、备注
   - 主键/外键/唯一索引/普通索引
   - 表间关系说明
   - 枚举值定义
3. 产出可执行 DDL → sql/init.sql（含所有表结构 + 预置数据 + 索引）
4. 同步产出 OpenSpec design.md 中的数据库设计引用：
   ```markdown
   # 数据库设计
   数据库设计详见 [docs/database/数据库设计.md](../../docs/database/数据库设计.md)
   ```
5. openspec validate → 检查 proposal + specs + design.md 结构完整性

**④ 编写 Plan → docs/plans/{slug}.md**

**加载 Superpowers：** `writing-plans` — `read_file D:\\hermes\\superpowers\\skills\\writing-plans/SKILL.md → 按步骤执行`
**加载 agent-skill：** `planning-and-task-breakdown` — `read_file D:\\hermes\\agent-skills\\skills\\planning-and-task-breakdown/SKILL.md → 按步骤执行`

1. 画依赖图：数据库 → API → 前端
2. 垂直切片：每条功能完整路径一次做完
3. 每个任务粒度合理（含目标/文件路径/预计耗时/代码片段/验证方式），一个功能 3~8 个任务
4. **加载之前写的设计文档** — read_file docs/architecture/architecture.md + docs/api/接口定义.md + docs/database/数据库设计.md，Plan 中的任务结构应与设计文档一致
   - 有前端时：原型项目已存在于 `docs/prototype/`，Plan 中的每个前端任务应明确是在原型组件上新增还是修改
5. 有前端：每个 Task 注明以下信息
   - **组件类名**（加载 docs/design/design.css + docs/design/design-guide.md + docs/design/component-library.md）
   - **页面路由**（对应页面原型中的 URL 路径）
   - **对接 API**（标注调用 docs/api/接口定义.md 中的哪个接口）
6. 产出 Plan → docs/plans/{slug}.md
7. 同步产出 OpenSpec tasks.md（内容为指向 Plan 的标准引用）
   ```
   # 任务分解
   任务分解详见 [docs/plans/{slug}.md](../../docs/plans/{slug}.md)
   ```
8. openspec validate → 检查 tasks.md 结构完整性

**展示全部设计文档给用户一次性确认：** 展示 docs/architecture/architecture.md + docs/api/接口定义.md + docs/database/数据库设计.md + docs/plans/{slug}.md → 确认后继续，有修改则修正后重新展示。

**🚪 门禁：**
```
□ 设计文档间无冲突（API 字段与数据库字段对齐、架构分层与 API 路径一致）
□ **原型与架构一致：** page2Data 字段 ↔ 数据库字段；筛选条件 ↔ API 参数；表单字段 ↔ 请求体（有原型时）
□ 设计文档已全部产出并通过用户确认
□ OpenSpec design.md + tasks.md 已通过 validate
```
**门禁通过后提交（main 分支）：**
```bash
git add docs/architecture/ docs/adr/ docs/plans/ sql/init.sql
git commit -m "feat: {功能名} architecture, design and plan"
git push origin main
```

---

### 3. 开发实施

**分支策略：** 开发前 `git checkout -b feat/{功能名}`，每个任务完成后 commit 到该分支。开发完成并本地测试通过后 push 到远程，后续通过 GitHub Pull Request 合并。

使用 `delegate_task` 按 Plan 切片逐任务 TDD 执行子代理。

**子代理 context 模板：**
```
使用 Superpowers TDD（RED→GREEN→REFACTOR）——全栈工程师，Plan：docs/plans/{slug}.md

先读取：docs/plans/{slug}.md（Plan）+ docs/architecture/architecture.md（架构）+ docs/api/接口定义.md（API 接口）+ sql/init.sql（DDL）
有前端时额外读取：docs/design/design.css + docs/design/design-guide.md（设计风格），原型项目已存在于 docs/prototype/，**以前端原型为开发基础**
  - **原型参照规则：**
    1. 原型 `index.html` 中的 `renderXxx()` 函数 → 改造为 AJAX 调用后保留，**不重写前端页面**
    2. 原型 `page2Data` 中的字段定义 → 映射到 API 响应 JSON，确保字段名一致
    3. 原型的筛选条件栏、表单弹窗、表格列 → 对应 API 的查询参数、请求体、响应字段
    4. 原型的 `routes + rolePermissions` → 前端路由和权限控制逻辑，**保留不变**
    5. 原型的 CSS 样式 → 保留 `design.css`，**禁止自创样式**
  - **数据替换模式：** `page2Data.staticData.map(...)` → `fetch('/api/xxx').then(r=>r.json()).then(data=>data.list.map(...))`
加载：read_file D:\hermes\superpowers\skills\subagent-driven-development/SKILL.md → 按步骤执行
加载：read_file D:\hermes\agent-skills\skills\incremental-implementation/SKILL.md → 按步骤执行

Step 0: 执行 sql/init.sql 建库建表（未初始化时执行；如数据库已存在则跳过，避免重复建表报错）
⚠️ **禁止修改数据库连接配置**（已在架构设计阶段配置好）。如需确认连接，按数据库类型测试：
⚠️ **MyBatis 实体字段检查：** 确保 Entity 类包含 resultMap 中映射的全部字段（如 createTime、updateTime 等），缺少字段会导致 MyBatis 映射异常
- **MySQL：** `mysql -u {应用用户} -p -e "SELECT 1;"`
- **SQLite：** 文件存在即连接成功，无需额外测试
然后按 Plan 切片逐任务 TDD（RED→写单元测试 + 集成测试 → GREEN→实现代码 → REFACTOR→重构），测试覆盖边界/中文/安全/分页等 8 类盲区
有前端时加载 docs/design/design.css + docs/design/design-guide.md + docs/design/component-library.md，禁止自创样式
门禁：openspec validate 通过 + CLAUDE.md 合规 + 数据库已初始化
```

**全部任务完成后，执行完整测试检查：**
1. 运行全部单元测试，确保 100% 通过
   - **Java/Maven：** `mvn test`
   - **Python：** `pytest` 或 `python -m pytest`
2. **需求-测试追溯：** 逐条对照 `openspec/specs/{domain}/spec.md`，确保每行需求至少对应一条测试用例
   - 产出测试追溯矩阵 → `test/traceability.md`（多模块汇总到一个文件，按模块分 section）
   - 缺失测试覆盖的需求条目，补写测试用例后再继续（格式：需求条目 | 测试方法 | 测试类 | 状态）
3. 覆盖率检查（分层标准）：
   - **核心业务模块**（预算校验/审批流/权限等纯逻辑）：**≥95%**
   - **Service/业务层：** **≥90%**
   - **Controller/API 层：** **≥85%**
   - **Mapper/DAO/Repository：** 集成测试覆盖，不设行覆盖率硬线
   - **UI/前端：** E2E 测试覆盖，不设行覆盖率
   - **整体：** **≥85%**（实体类 POJO 可排除，不纳入覆盖率计算）
覆盖率命令按技术栈执行：
- **Java/Maven（JaCoCo）：** `mvn jacoco:report` → 查看 `target/site/jacoco/index.html`
  - ⚠️ **首次使用需先配置 JaCoCo 插件到 pom.xml**，参考：
    ```xml
    <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <version>0.8.11</version>
        <configuration>
            <excludes>
                <exclude>**/entity/**</exclude>  <!-- 实体类 POJO 排除 -->
            </excludes>
        </configuration>
        <executions>
            <execution>
                <id>prepare-agent</id>
                <goals><goal>prepare-agent</goal></goals>
            </execution>
            <execution>
                <id>report</id>
                <phase>test</phase>
                <goals><goal>report</goal></goals>
            </execution>
        </executions>
    </plugin>
    ```
- **Python：** `pytest --cov=src --cov-report=term`

**🚪 门禁（完成时检查）：**
```
□ openspec validate 通过
□ CLAUDE.md 合规自检通过
□ 数据库已初始化
□ 全部单元测试通过
□ 集成测试用例已编写（test/ 目录下存在集成测试文件）
□ 需求-测试追溯矩阵已产出且覆盖完整
□ 覆盖率达标（核心≥95% / Service≥90% / Controller≥85% / 整体≥85%）
□ **MyBatis 实体字段映射检查（Java/MyBatis 项目）：** resultMap 中的字段与 Entity 类字段一致（如 createTime/updateTime 等常见遗漏字段）
□ **前端与原型一致（有原型时）：** 页面布局、字段名、筛选条件、表单交互与原型一致，无新增的自创样式
```

---

### 4. 测试保障

1. **端到端集成测试（验证业务流程，不看页面长相）：** 运行自动化测试，确保 API → 数据库 → 业务逻辑的通路正确
   - 用户创建 → 项目创建 → 工时报单 → 审核通过 → 统计看板数据正确
   - 权限校验：越权操作被正确拦截
2. **有前端时：自动视觉验收 — 页面完整性检查**
   - 启动本地服务，遍历所有路由，逐页执行以下自动化检查：
     - `browser_navigate(url)` 打开页面
     - `browser_console("document.querySelector('link[href*=design]') ? 'OK' : 'design.css 未加载'")` — 检查样式文件已加载
     - `browser_console("document.querySelectorAll('[src],link[href]').length > 0")` — 确认页面有资源加载
     - `browser_console("getComputedStyle(document.body).getPropertyValue('--color-primary')")` — 验证 CSS 自定义属性已定义
     - `browser_snapshot()` — 检查关键 UI 元素存在（侧边栏、顶部栏、内容区、表格、表单等）
     - `browser_console(clear=true)` — 确认无 JS 报错
   - 所有检查通过 → 视觉验收通过
   - 任一项失败 → 修复后重新验收
   - **SPA（单页应用）特殊处理：**
     - 如果前端是 SPA（所有路由在同一个 HTML 文件通过 JS 切换），无需逐个 URL 打开
     - 改用源码分析验证完整性：`grep "function render" index.html`（检查所有 render 函数存在）
     - 检查路由定义：`grep "const routes" index.html`（确保所有模块在 routes 对象中注册）
     - 检查弹窗/表单函数：`grep "function show" index.html`（确保 CRUD 弹窗全部实现）
   - **登录墙处理：**
     - SPA 页面可能需要登录才能渲染数据，此时直接 browser_navigate 无法看到完整页面
     - 改用 read_file 读取源码验证页面结构和所有 render 函数
     - 对于非认证页面（如 login.html 等公开页面），直接 browser_navigate 验证

   **📋 页面完整性检查清单（逐项确认）：**

   **登录页检查（公开页面）：**
   ```
   □ 页面标题 — browser_console("document.title") → 正确
   □ 样式文件加载 — browser_console("querySelector('link[href*=design]')") → OK
   □ CSS 自定义属性 — browser_console("getComputedStyle(body).getPropertyValue('--color-primary')") → 有值
   □ 用户名输入框 — browser_snapshot 可见 #username
   □ 密码输入框 — browser_snapshot 可见 #password
   □ 验证码输入框 — browser_snapshot 可见 captcha 字段
   □ 验证码图片刷新 — 可点击刷新（onclick="refreshCaptcha()"）
   □ 登录按钮 — 点击触发登录 API 调用
   □ 错误提示 — #loginError 存在，登录失败后显示
   □ 登录 API 对接 — 调用了 /api/auth/login
   □ 验证码 API 对接 — 调用了 /api/auth/captcha
   □ JS 报错 — browser_console(clear=true) → 无影响性报错
   ```

   **首页结构检查（SPA 主页面）：**
   ```
   □ 页面标题 — "项目名称" 正确
   □ 导航栏 navbar — DOM 存在，显示用户信息
   □ 侧边栏 sidebar — DOM 元素存在（ASIDE）
   □ 内容区 mainContent — DOM 元素存在（MAIN）
   □ 样式文件 — design.css 已加载
   □ CSS 自定义属性 — --color-primary 已定义
   ```

   **路由模块源码检查（SPA 源码分析）：**
   ```
   □ 路由定义 — grep "const routes" index.html → 检查所有模块是否注册
   □ 渲染函数 — grep "function render" index.html → 每个模块有对应 renderXxx()
   □ 弹窗/表单函数 — grep "function show" index.html → 每个 CRUD 有对应的 showXxxForm()
   □ 核心函数 — loadAllData() / navigate() / checkLogin() / showConfirm() / showToast()
   □ 权限控制 — rolePermissions 每个角色有模块级权限定义
   ```

   **CSS 组件完整性检查（design.css）：**
   ```
   □ CSS 变量 — --color-primary / --color-bg / --radius 等语义变量
   □ 按钮 — .btn / .btn-primary / .btn-secondary / .btn-danger 全部定义
   □ 表单 — .form-input / .form-select / .form-group / .form-label
   □ 状态标签 — .badge / .badge-pending / .badge-approved / .badge-rejected
   □ 弹窗 — .modal / .modal-header / .modal-body / .modal-footer
   □ 侧边栏 — .sidebar / .sidebar-menu / .sidebar-menu li.active
   □ 表格 — table / .table / th / td 样式
   □ 分页 — .pagination / .page-btn / .page-btn.active / .page-info
   □ 导航栏 — .navbar / .navbar-brand / .navbar-user / .navbar-right
   □ Toast — .toast / .toast-success / .toast-error
   ```

   **检查结果记录格式：**
   ```
   登录页: □/□ 项通过
   首页结构: □/□ 项通过
   路由模块: □/□ 项通过
   CSS 组件: □/□ 项通过
   结论: ✅ 视觉验收通过 / ❌ 修复后重新验收
   ```
3. **有前端时：E2E 浏览器测试（验证用户操作链路，不看数据对错）**
   - 加载 `browser-testing-with-devtools` skill：`read_file D:\\hermes\\agent-skills\\skills\\browser-testing-with-devtools/SKILL.md → 按步骤执行`
   - 产出 E2E 测试结论 → `docs/reports/{功能名}-e2e-report.md`
   - 覆盖盲区：
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
4. **变异测试：** 根据需求复杂度自动判断
   - 读取 `openspec/specs/{domain}/spec.md`，检测是否有审批流/权限矩阵/状态机/复杂校验等模式
   - 有复杂业务逻辑 → 自动执行变异测试
   - 标准 CRUD（无复杂逻辑）→ 自动跳过
5. **性能测试：** 根据 Phase 1 确认的非功能需求执行
   - 有性能指标（页面加载 ≤3s、API 响应 ≤500ms、并发数等）→ 执行
   - 无性能指标或嵌入式数据库（SQLite/H2）→ 跳过
   - 页面加载时间 ≤ 3s（浏览器 DevTools 或 curl -w 测）
   - API 响应时间 ≤ 500ms（单接口，curl -w "%{time_total}"）
   - 并发 10 用户无错误（ab -n 100 -c 10 或 k6）
6. 将测试结果写入 `docs/reports/{功能名}-test-report.md`

**🚪 门禁：**
```
□ 集成测试 PASS
□ 页面完整性检查通过（有前端时）
□ E2E 测试已编写并全部通过（有前端时）
□ 变异测试 PASS 或自动跳过（根据需求复杂度判断）
□ 性能测试 PASS 或符合条件跳过（根据非功能需求判断）
```
**门禁通过后提交（feat 分支）：**
```bash
git add docs/reports/ test/
git commit -m "feat: {功能名} tests and reports"
git push origin feat/{功能名}
```

---

### 5. 审查与归档

**阶段间顺序：** 先质量审查 → 再安全审查 → 门禁通过后自动归档

> 如使用 GitHub，审查已在本阶段全部完成（含自动扫描+质量+安全），PR 仅用于 Merge，无需重复 Review。

加载 Superpowers：`requesting-code-review` — `read_file D:\hermes\superpowers\skills\requesting-code-review/SKILL.md → 按步骤执行`
加载 agent-skill：`code-review-and-quality` — `read_file D:\hermes\agent-skills\skills\code-review-and-quality/SKILL.md → 按步骤执行`

**质量审查内部顺序不可换：** 自动扫描 → 质量审查

自动扫描：运行以下三条安全工具 + 品味检查（阶段性快照，审查时发现问题当场修；熵管理 cronjob 已由 Phase 0 配置，每周五自动运行不变式检查）
  - trivy fs --severity CRITICAL,HIGH .
  - semgrep --config=auto .
  - gitleaks detect --source . -v
  - bash scripts/taste-check.sh

质量审查（六维）：正确性/可读性/架构/安全/性能/品味 + 设计一致性
产出：docs/reviews/{功能名}-{日期}.md

**安全审查（在质量审查之后执行）：**

加载 agent-skill：`security-and-hardening` — `read_file D:\hermes\agent-skills\skills\security-and-hardening/SKILL.md → 按步骤执行`

1. SQL注入/XSS/JWT密钥/密码BCrypt/鉴权/密钥硬编码
2. 依赖安全审计：按技术栈检查已知漏洞
   - **Java/Maven：** `mvn org.owasp:dependency-check-maven:check`
   - **Python：** `pip-audit`
3. 无 CRITICAL 级别漏洞才能通过
4. 结论追加到 docs/reviews/{功能名}-{日期}.md

**🚪 门禁：**
```
□ 自动扫描：无 CRITICAL/HIGH 漏洞（trivy + semgrep + gitleaks + taste-check）
□ 质量审查：PASS（六维：正确性/可读性/架构/安全/性能/品味）
□ 安全审查：PASS（SQL注入/XSS/JWT/密码/鉴权/依赖审计无高危漏洞）
□ 审查报告已写入 docs/reviews/{功能名}-{日期}.md（含质量审查 + 安全审查结论）
```

**门禁通过后执行归档（门禁不过不执行）：**

先确认远程仓库已配置：
```bash
git remote -v || { echo "❌ 未配置远程仓库，请先执行: git remote add origin <仓库地址>"; exit 1; }
```

```bash
# 1. 推送功能分支到远程
git push origin feat/{功能名}

# 2. 创建 Pull Request
gh pr create --title "feat: {功能名}" --body \
  "详见 openspec/specs/{domain}/spec.md" \
  --base main || echo "PR 已存在或手动创建"

# 3. Squash Merge 并删除远程分支
git checkout main && git pull origin main
gh pr merge --squash --delete-branch --subject "feat: {功能名}"
```

**自检：**
```
□ 每行改动对应需求
□ 无"顺手改"的相邻代码
□ 无孤儿代码
```

---

### 6. 回顾总结

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

