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
🔴 Skill 引用以文件路径为准：`read_file` 加载前先确认 `.md` 文件存在，不因 `skill_view` 找不到而删除或跳过

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
| 含多层转义的文件修改（HTML/JS 嵌套引号） | 优先用 `execute_code` 做字节级替换，避免 `patch` 引入多余转义 | `patch` 失败则改用 `execute_code` 重试 | — |
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
| **字段增删** | 项目加「所属部门」、任务去「分配人员」 | 同步更新 spec + prototype + 前端数据模型 |
| **业务逻辑** | 工时仅已驳回可编辑、审批按角色过滤 | 原型条件渲染 + spec Scenario 更新 |
| **数据驱动** | 角色/用户下拉从 API 动态加载 | 不要硬编码 option/list 数据 |
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
**read_file docs/prototype/index.html（有前端时，有原型时）— 提取页面结构、数据模型、路由、筛选条件、表单字段，作为架构设计的输入**

**⚡ 原型 → 架构映射规则（通用，适配所有前端技术栈）：**
- 原型中的数据定义（Mock数据/静态列表）→ 数据库表字段
- 原型的筛选条件栏 → API GET 查询参数
- 原型的表单字段 → API POST/PUT 请求体
- 原型的权限控制逻辑 → RBAC 权限表结构
- 原型的路由/页面划分 → 前后端模块划分依据

**对应技术栈参考：**
- **原生 HTML/JS SPA：** page2Data → 数据库字段；renderXxx() 筛选栏 → API 参数；showXxxForm() 表单 → 请求体；rolePermissions → 权限表
- **Vue：** 组件 data/mounted 中的静态数据 → 数据库字段；模板中的筛选栏 → API 参数；el-form 字段 → 请求体；vue-router + 路由守卫 → 权限表
- **React：** 组件 state/useState 静态数据 → 数据库字段；JSX 筛选栏 → API 参数；Form 组件字段 → 请求体；react-router + 鉴权组件 → 权限表
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

**分支策略：** 开发前 `git checkout -b feat/{功能名}`，每任务提交到该分支，完成后 push → PR 合并 main。

使用 `delegate_task` 按 Plan 切片逐任务 TDD 执行子代理。

---

#### 子代理 context 模板

粘贴到 `delegate_task(context=...)` 中，按需微调 `{slug}` 和 `{domain}`，**并根据实际技术栈调整 Step 0 的连接检查和字段映射检查**：

```
Superpowers TDD（RED→GREEN→REFACTOR），Plan：docs/plans/{slug}.md

Step 0：执行 sql/init.sql 建库建表（已存在则跳过）
⚠️ 禁止修改数据库连接配置
【数据库连接检查】按技术栈选：
  - MySQL：mysql -u {应用用户} -p -e "SELECT 1;"
  - SQLite：文件存在即 OK，跳过
【实体字段映射检查】按技术栈选：
  - Java/MyBatis：Entity 类须包含 resultMap 全部字段（createTime/updateTime 等）
  - Python/SQLAlchemy：Model 类字段须与表字段一致
  - 其他 ORM：确保实体模型与 DDL 字段对齐

按 Plan 切片逐任务：RED→写单元测试+集成测试 → GREEN→实现 → REFACTOR→重构
测试覆盖边界/中文/安全/分页等 8 类盲区
加载：read_file D:\hermes\superpowers\skills\subagent-driven-development/SKILL.md → 按步骤执行
加载：read_file D:\hermes\agent-skills\skills\incremental-implementation/SKILL.md → 按步骤执行

【有前端时】额外读取：docs/design/design.css + design-guide.md + component-library.md
原型 docs/prototype/，规则：
1. 静态数据展示 → API 动态渲染，保留布局和交互
2. Mock 数据定义 → 映射 API 响应字段（字段名一致）
3. 筛选/表单/表格列 → 对应 API 参数/请求体/响应字段
4. 路由路径和模块划分保留不变
5. 权限规则保留不变
6. 视觉风格保留，禁止自创样式

数据替换：
- 原生JS：dataCache.xxx 替换 page2Data → renderXxx() API 渲染 → showXxxForm() 调真实 API
- Vue：mounted() 调 API → data 替换静态数据 → 表单 axios.post/put
- React：useEffect 调 API → useState 替换静态数据 → 表单 api.post/put
- 模式：page2Data.staticData → fetch('/api/xxx').then(r=>r.json()).then(data=>data.list)
```

---

#### 前端 Mock 改造（无前端时跳过整段）

原型 `docs/prototype/` 是静态 Mock 数据，**开发阶段必须逐页改完，不允许遗留到测试阶段**。

**6步改造流程：**
1. **API 封装层：** 添加 `api.get/post/put/del`（原生JS）/ axios 实例（Vue）/ fetch 封装（React）
2. **对照需求规格：** 打开 `openspec/specs/{domain}/spec.md`，逐条检查页面/组件完整 → 缺失即补充
3. **数据层：** 静态数据定义 → API 动态加载；页面初始化调 API 缓存到全局/状态管理
4. **渲染层：** 列表/表格/卡片 → API 数据渲染；查询条件栏 → 绑定参数调 API 筛选
5. **CRUD 表单：** 保存/提交 → 调 POST/PUT；删除 → 调 DELETE，操作成功后刷新数据
6. **样式：** 视觉风格保留原型，只替换数据来源和 API 调用

**自检清单：**
```
□ 所有页面从 API 渲染，无静态硬编码数据
□ 保存/提交/删除按钮全部调真实 API（无模拟 Toast/确认）
□ 查询筛选功能已实现
□ 页面渲染无报错
□ 无技术栈常见问题（原生JS引号嵌套、Vue this指向、React hooks 依赖等）
```

---

#### 全部任务完成后验证

**① 前端改造验收（有前端时）：**
1. 打开浏览器逐页检查 → 数据是否来自 API（对比数据库）→ 新建/删除是否真实读写 → 遗留 Mock 登记🔴到问题清单
2. **对照原型 `docs/prototype/` 检查页面布局、字段名、筛选条件、表单交互是否一致** → 发现自创样式/布局走样登记🔴

**② 测试检查：**
1. 运行全部单元测试：`mvn test` / `pytest`，确保 100% 通过
2. **需求-测试追溯：** 逐条对照 `openspec/specs/{domain}/spec.md`，产出 `docs/test/traceability.md`（需求条目 | 测试方法 | 测试类 | 状态），缺失补写
3. **覆盖率检查：**
   - 核心业务（预算/审批/权限）≥95% | Service ≥90% | Controller ≥85% | 整体 ≥85%
   - Mapper/DAO/前端：集成测试覆盖，不设行覆盖率硬线
   - 实体类 POJO 排除
   - 命令：`mvn jacoco:report`（配 JaCoCo 插件后使用） / `pytest --cov=src --cov-report=term`

---

#### 🚪 门禁（完成时检查）

```
□ openspec validate 通过
□ CLAUDE.md 合规自检通过
□ 数据库已初始化
□ 全部单元测试通过
□ 需求-测试追溯矩阵已产出且覆盖完整
□ 覆盖率达标（核心≥95% / Service≥90% / Controller≥85% / 整体≥85%）
□ 实体-表字段映射检查通过（Java/MyBatis 的 resultMap ↔ Entity 字段一致；Python/SQLAlchemy 的 Model ↔ 表字段一致）
□ 前端 Mock 数据已全部改造为真实 API（有前端时，无前端跳过）
□ 前端与原型一致（有原型时），无自创样式
```

---

### 4. 测试保障

**统一问题清单：** 所有测试环节发现的问题统一登记到 `docs/test/issues.md`，格式如下：

```markdown
## Bug #{编号}
- **模块：** 所属功能模块
- **问题描述：** 清晰描述问题和复现步骤
- **截图/日志：** 如有
- **发现阶段：** 集成测试 / 页面结构检查 / 用户测试
- **严重度：** 🔴 高（功能不可用）/ 🟡 中（功能异常但可绕过）/ 🔵 低（UI/文案问题）
- **状态：** ⏳ 待修复 / 🔧 修复中 / ✅ 已修复 / 📌 暂缓
- **发现时间：** {日期}
- **修复时间：** {日期}
- **修复方式：** 简要说明如何修复
```

**统一修复流程：**

1. 按严重度排序修复（🔴 → 🟡 → 🔵），每修复一个更新问题清单状态为 ✅
2. 回归验证：修复后在对应环节重新执行测试，确认问题已解决且不引入新问题
3. **最终检查：** 所有 🔴 🟡 问题均标记为 ✅ 或 📌（经用户确认暂缓），方可进入门禁

1. **端到端集成测试（验证业务流程，不看页面长相）：**
   - **① 编写测试案例：** 对照 `openspec/specs/{domain}/spec.md` 逐条编写测试案例，每个核心流程一个案例（创建→流转/审批→统计、越权拦截等），写明测试场景、步骤、预期结果 → 保存到 `docs/test/test-cases.md`
   - **② 执行自动化测试：** 运行测试，确保 API → 数据库 → 业务逻辑的通路正确
   - 测试发现的问题 → 登记到 `docs/test/issues.md`，修复后更新状态

2. **有前端时：页面结构检查 — 对照需求规格，确认所有页面模块和组件已注册**

   根据前端技术栈选择对应的检查方式：

   **原生 HTML/JS 单页应用（SPA）：**
   - **源码分析：** 对照 `openspec/specs/{domain}/spec.md` 中的功能模块，逐条检查：
     - 路由定义 — `grep "const routes" index.html` → 检查每个需求对应的模块是否已注册路由
     - 渲染函数 — `grep "function render" index.html` → 每个模块有对应 renderXxx()
     - 弹窗/表单函数 — `grep "function show" index.html` → 每个 CRUD 有对应的 showXxxForm()
     - 核心函数 — loadAllData() / navigate() / checkLogin() / showConfirm() / showToast()
     - 权限控制 — rolePermissions 每个角色有模块级权限定义
   - **CSS 组件完整性检查（design.css）：**
     ```text
     □ CSS 变量 — --color-primary / --color-bg / --radius 等语义变量
     □ 按钮 — .btn / .btn-primary / .btn-secondary / .btn-danger / .btn-sm
     □ 表单 — .form-input / .form-select / .form-group / .form-label
     □ 卡片 — .card / .card-title
     □ 统计卡片 — .stat-card / .stat-grid / .stat-value / .stat-label
     □ 筛选栏 — .filter-bar / .filter-bar .form-input / .filter-bar .form-select
     □ 状态标签 — .badge / .badge-pending / .badge-approved / .badge-rejected
     □ 弹窗 — .modal / .modal-title / .modal-footer
     □ 侧边栏 — .sidebar / .sidebar-menu / .sidebar-menu li.active / .menu-icon
     □ 表格 — .table-wrapper / table / th / td
     □ 分页 — .pagination / .page-btn / .page-btn.active / .page-info
     □ 导航栏 — .navbar / .navbar-brand / .navbar-user / .navbar-right
     □ Toast — .toast / .toast-success / .toast-error / .toast-warning
     ```
   - **登录墙处理：** SPA 页面可能需要登录才能渲染，此时直接 `read_file index.html` 读源码验证

   **Vue / React 等组件化框架：**
   - **路由检查：** 查看路由配置文件（`src/router/` 或 `src/App.vue`），确认所有页面路由已注册
   - **组件检查：** 确认每个路由对应的视图组件存在，且组件文件结构与需求一致
   - **组件库/样式检查：** 确认使用的 UI 组件库（Element Plus / Ant Design 等）中，页面所需的组件类型（表格/弹窗/表单/分页等）已全部引入，无自创样式
   - **API 对接检查：** 确认每个页面的 API 调用文件（如 `src/api/`）覆盖了全部后端接口

   - **通用注意：** 页面渲染完整性、JS报错、按钮点击等动态验证 → 由用户测试（Playwright）覆盖，此处不做重复检查
   - 发现的问题 → 登记到 `docs/test/issues.md`，修复后更新状态

3. **有前端时：用户测试（Playwright 浏览器测试）** — 像真实用户一样逐功能验证

   **前置条件：** 启动本地开发服务器，确保服务可访问

   **测试流程：**

   1. **编写 Playwright 测试脚本：** 使用 `playwright.sync_api`，有头模式（`headless=False`），模拟真实用户操作
      - 登录 → 依次点击每个菜单 → 测试每个 CRUD 按钮
      - 每个操作后断言页面正常渲染（Toast / 弹窗 / 数据刷新）
      - 捕获 JS 控制台报错

   2. **对照需求规格/原型，逐项验证：** 打开 `openspec/specs/{domain}/spec.md` 和原型 `docs/prototype/`，逐条确认每个需求对应的功能在浏览器中可用
      - 记录已测试数量 vs 总数（如：功能模块 8/8、CRUD 操作 24/24、筛选栏 6/6）
      - 逐项标记完成，全部打勾才算覆盖完整
      - 如果发现需求中写了但页面上找不到的功能 → 登记到问题清单 🔴

      ```text
      □ 登录/退出功能完整（验证码 → 登录 → 首页 → 退出）
      □ 首页仪表盘/看板数据正确
      □ 每个管理模块 CRUD 完整（列表 → 新建 → 编辑 → 删除）
      □ 带业务状态的模块（审批/流程等）状态流转正确（通过 → 驳回 → 重新提交）
      □ 关联数据操作正常（如选择关联记录、级联删除等）
      □ 搜索/筛选/查询功能正常（输入 → 查询 → 重置）
      □ 修改密码功能
      □ 菜单/导航切换正确
      □ 分页组件正常翻页
      □ 0 JS 报错
      ```

      **覆盖完整性自检：**
      ```text
      □ 页面结构检查输出的 N 个路由已全部点击验证
      □ 页面结构检查输出的 M 个 CRUD 弹窗已全部打开并保存
      □ 页面结构检查输出的 K 个权限角色已全部切换验证
      □ 页面结构检查输出的 CSS 组件在用户测试中均正常渲染
      ```

   3. **发现问题 → 登记到问题清单：** 按上方统一格式登记到 `docs/test/issues.md`，修复后更新状态

4. **汇总测试报告：** 将集成测试、页面结构检查、用户测试的结论和问题清单状态写入 `docs/test/reports/{功能名}-test-report.md`

**🚪 门禁：**
```
□ 集成测试 PASS
□ 页面结构检查通过（有前端时）
□ **用户测试已执行且全部通过或有记录的 🔴 🟡 问题已全部修复/确认暂缓（有前端时）**
```
**门禁通过后提交（feat 分支）：**
```bash
git add docs/test/ src/test/
git commit -m "feat: {功能名} tests and reports"
git push origin feat/{功能名}
```

---

### 5. 审查与归档

**阶段间顺序：** 先质量审查 → 再安全审查 → 门禁通过后自动归档

> 如使用 GitHub，审查已在本阶段全部完成（含自动扫描+质量+安全），PR 仅用于 Merge，无需重复 Review。

加载 Superpowers：`requesting-code-review` — `read_file D:\hermes\superpowers\skills\requesting-code-review/SKILL.md` → 按步骤执行
加载 agent-skill：`code-review-and-quality` — `read_file D:\hermes\agent-skills\skills\code-review-and-quality/SKILL.md` → 按步骤执行

**质量审查内部顺序不可换：** 自动扫描 → 质量审查

> ⚠️ 前置条件：先 `git add` 待审查文件，确保 `git diff --cached` 有内容，否则 `requesting-code-review` 的子代理审查会因空 diff 失败。

自动扫描：
  - `semgrep --config=auto .`
  - `gitleaks detect --source . -v`
  - `bash scripts/taste-check.sh`

质量审查（六维）：正确性/可读性/架构/安全/性能/品味
   - 抽查 `openspec/specs/{domain}/spec.md` 中 20% 的需求条目，确认实现与需求一致
产出：`docs/reviews/{功能名}-{日期}.md`

**安全审查（在质量审查之后执行）：**

加载：`read_file D:\hermes\agent-skills\skills\security-and-hardening/SKILL.md` → 按步骤执行

1. SQL注入/XSS/密码BCrypt/鉴权/密钥硬编码检查
2. 结论追加到审查报告

**🚪 门禁：**
```
□ 自动扫描：无 CRITICAL/HIGH 漏洞（semgrep + gitleaks + taste-check）
□ 质量审查：PASS（六维：正确性/可读性/架构/安全/性能/品味）
□ 安全审查：PASS（SQL注入/XSS/密码/鉴权无高危漏洞）
□ 审查报告已写入 docs/reviews/{功能名}-{日期}.md（含质量审查 + 安全审查结论）
```

**门禁通过后执行归档（门禁不过不执行）：**

```bash
# 有 feat 分支时
git push origin feat/{功能名}
gh pr create --title "feat: {功能名}" --body "详见 openspec/specs/{domain}/spec.md" --base main
# 合并后清理本地分支
git checkout main && git pull origin main && git branch -d feat/{功能名}

# 无 feat 分支（直接提交 main）时
git push origin main
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

**流程回馈检查：** 产出回顾后，逐条列出踩坑记录让用户选择是否回流：

```markdown
## 踩坑记录

| # | 问题 | 方案 | 是否回流 |
|:-|:-----|:-----|:--------|
| 1 | {问题描述} | {解决方案} | ☐ 回流到流程文档 / ☐ 不留 |
| 2 | {问题描述} | {解决方案} | ☐ 回流到流程文档 / ☐ 不留 |
```

逐条确认后，对标记"回流"的条目执行对应操作：
- 流程文档遗漏 → 更新 `Hermes-标准化研发流程.md`
- skill 未覆盖的陷阱 → 更新对应 skill
- 工具使用痛点 → 更新文档示例
- 通用规则缺失 → 补充到通用规则章节

**提交（main 分支）：**
```bash
git add docs/retrospectives/
git commit -m "feat: {功能名} retrospective"
git push origin main
```

---

