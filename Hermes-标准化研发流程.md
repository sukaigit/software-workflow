# Hermes-标准化研发流程

> v1.3 | 7 阶段流水线
> 核心理念：Harness Engineering，不是让 AI 更聪明，而是设计约束环境让 AI 不犯错
> 生命周期：需求分析 → 架构设计 → 开发实施 → 测试保障 → 审查 → 部署验收 → 回顾总结

---

## 通用规则

```
🔴 跨阶段修改：本阶段发现上一阶段产出物有问题 → 修改后**立即提交**，不积压不跳过后补
```

---

## 项目初始化

### Step 0：问项目基本信息

先问项目名称（中文以及英文目录名），用户回答后，再问技术栈，再问数据库类型，最后问 GitHub 仓库可见性。

```
示例：
→ 项目名称是什么？
← 项目工时管理系统 project-time-manage
→ 技术栈是什么？（后端: Spring Boot+MyBatis / Flask / FastAPI 等；前端: Vue3）
← Spring Boot + MyBatis + Vue3
→ 数据库类型是什么？（MySQL / SQLite 等）
← MySQL
→ GitHub 仓库可见性？（public/private）
← private
```

**展示给用户确认：** 汇总项目信息给用户——"项目名称：xxx，技术栈：xxx，数据库类型：xxx，仓库可见性：xxx，确认无误？"
→ 用户确认后进入 Step 1。如有修改，修正后重新展示确认。

### Step 1：创建项目骨架

> 使用 GitHub 时，Step 1 会自动检查 gh CLI 是否已安装并完成认证，未登录则提示用户执行 `gh auth login`

```bash
mkdir -p /d/hermes/workspace/{项目名} && cd /d/hermes/workspace/{项目名}
git init
cp /d/hermes/softwareWorkFlow/CLAUDE_CN.md ./CLAUDE.md   # 中英文行为准则，再补充技术栈特定规则
mkdir -p ops sql \
  docs/adr docs/architecture docs/api docs/database docs/plans \
  docs/test docs/reviews docs/retrospectives docs/reports

cp /d/hermes/softwareWorkFlow/.gitignore-template .gitignore

mkdir -p .github/workflows
echo "# ADR-001: {标题}" > docs/adr/ADR-001.md

# OpenSpec 初始化
openspec init --tools none

# 初始提交
git add . && git commit -m "chore: initial project scaffold"

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

---

### 1. 需求分析

**执行步骤：**
```
1. **逐一澄清需求，累计不生成：** 一问一答逐步澄清所有功能，每个功能确定后写入 OpenSpec spec 文件
   - 每个功能独立一个 `openspec/specs/{domain}/spec.md`
   - 每条 Requirement 正文必须包含 SHALL 或 MUST
2. **字段/边界/非功能一次性问完：** 所有功能的字段定义、边界条件、非功能需求全部确认完毕后再进入下一步
   - 字段定义、边界条件、非功能需求逐功能一次性问完并写入 Scenario
   - 参考示例页的字段模式问需求：\"这个功能需要哪些字段？（类似产品管理的名称/分类/单价/库存模式）\"、\"哪些是必填？\"、\"状态有哪些可选值？\"
3. **批量生成原型（有前端时）：**
   - 默认 Apple 风格，直接使用模板：
     - 复制模板：`cp -r /d/hermes/softwareWorkFlow/prototype ./docs/prototype`
     - 模板已有：Apple 设计规范（`design.css` + `docs/prototype/design/design-guide.md`）+
       系统管理套件（7 模块）+ 产品管理示例页（CRUD/筛选/分页/导入/导出/删除确认/备注）
     - 保留示例页作为参考模板（所有业务页面生成并确认后手动删除）
   - **标准组件清单（验证用）：** 布局/登录/首页/筛选栏/表格/弹窗/分页/确认框/Toast/状态标签/导出(Excel XML .xls)/导入(.xls+.csv)
   - **批量创建所有业务页面：** 按已确认的 spec 字段定义，参考模板中产品管理示例页的代码模式，一次性生成所有业务页面的 CRUD（列表+新增+编辑+删除+筛选+导出+分页+备注）
   - 启动 dev server
4. **一次性展示确认（不是改一个看一个）：**
   - 让用户浏览器打开，翻看**所有页面**
   - 用户列出全部问题（文案/字段/布局/交互）
   - **一次性修完所有问题** → 再展示确认
   - 反复至用户确认通过
5. **原型变更回流：** 将原型中所有页面的字段、交互、规则同步到 OpenSpec spec 文件
   - **逐个页面对照：** 按 `docs/prototype/src/router/index.js` 逐一检查每个路由对应的页面，确保每个页面在 spec 中有对应 Requirement/Scenario
   - **系统管理模块也要写 spec：** 登录/退出/改密/用户/部门/机构/角色/菜单/功能/日志 同样需要在 `openspec/specs/` 下有对应的 spec 文件
   - 字段变化 → 更新 spec 中字段定义
   - 状态值/枚举变化 → 更新 Scenario 中的值
   - 交互/行为变化 → 更新 Scenario 描述
   - 规则约束（密码规则/锁定时长/必填校验等）→ 补充到 Requirement 中
   - 弹出提示/Toast/确认框 → 记录到 Scenario 中
   - **最终检查：** 原型每个页面、每个按钮、每个弹窗都应在 spec 中有描述，prototype 与 spec 完全一致
6. `openspec validate --specs` — 最终规范校验
7. **提交：**
   ```bash
   git add openspec/specs/ docs/prototype/
   git commit -m "feat: {功能名} requirements and prototype"
   git tag prototype-v1
   git push origin main --tags
   ```
```

**产出物：** openspec/specs/（**唯一真相源 + 功能定义 + 业务规则**）+ docs/prototype/（**视觉参考 + 交互流程**，含设计风格说明 docs/prototype/design/design-guide.md + 系统管理套件）

> **specs vs 原型的分工：**
> - **specs** 决定「做什么」— 功能定义、字段、规则、枚举值、业务逻辑。后续阶段以 specs 为准。
> - **原型** 决定「长什么样」— 页面布局、交互方式、UI文案、视觉风格。开发前端时参考。
> - **⚠️ 前提条件：** 进入 Phase 2 前，specs 必须完整覆盖原型中所有页面（含系统管理模块），不能缺页。
> - 检查方式：ls openspec/specs/ 列出模块 vs grep path: router/index.js 列出页面 → 一一对应

---

### 2. 架构设计

**先读取上一阶段产出物：**
**read_file openspec/specs/{domain}/spec.md — 了解需求范围（唯一真相源）
**read_file docs/prototype/src/router/index.js — 了解所有页面（业务+系统管理）路由路径
read_file docs/prototype/src/views/ 下每个页面组件（含系统管理套件）— 从 data 和 template 中提取字段定义、筛选条件、表单字段，作为数据库和 API 设计的输入**
**⚠️ 原型与 specs 不一致时，以 specs 为准（原型变更应在 Phase 1 回流到 specs）**

**🔍 功能覆盖检查：** 对比 router 中的路由列表与 spec 文件列表，确保没有页面遗漏
- `ls openspec/specs/` → 列出已有 spec 模块
- `grep "path:" docs/prototype/src/router/index.js` → 列出所有页面路由
- 缺少 spec 的页面 → 先补充 spec，再继续架构设计

**⚡ 原型 → 架构映射规则：**
- 原型中的数据定义（Mock数据/静态列表）→ 数据库表字段
- 原型的筛选条件栏 → API GET 查询参数
- 原型的表单字段 → API POST/PUT 请求体
- 原型的权限控制逻辑 → RBAC 权限表结构
- 原型的路由/页面划分 → 前后端模块划分依据
- **Vue 对应：** data/mounted 静态数据 → 数据库字段；筛选栏 → API 参数；el-form → 请求体；vue-router → 权限表

**顺序执行以下设计任务（每份完成后自动进入下一份）：**

**① 系统架构设计 → docs/architecture/architecture.md**

1. 根据技术栈描述分层结构
   - **Java/Spring Boot（MyBatis）：** Controller → Service → Mapper → DB
   - **Python/Flask：** Routes → Services → Models → DB
   - **Python/FastAPI：** Routers → Services → Models → DB
   - **前端项目：** 组件树 → 状态管理 → API 层
2. 说明模块划分（**业务模块 + 系统管理模块都要覆盖**）、组件关系、数据流、技术选型依据
3. 关键决策记入 docs/adr/ADR-{序号}.md
4. **创建数据库专用应用用户（仅所选数据库类型需要时执行）：**
   - **MySQL：**
     ```bash
     mysql -u root -p'{密码}' -e "CREATE USER IF NOT EXISTS '{应用名}'@'localhost' IDENTIFIED BY '{密码}';"
     mysql -u root -p'{密码}' -e "GRANT SELECT, INSERT, UPDATE, DELETE ON {数据库名}.* TO '{应用名}'@'localhost';"
     mysql -u root -p'{密码}' -e "FLUSH PRIVILEGES;"
     ```
     - JDBC URL 使用 `localhost` 而非 `127.0.0.1`（MySQL JDBC 驱动将 127.0.0.1 解析为 localhost 导致认证失败）
   - **SQLite / 嵌入式数据库：** 无需创建应用用户，跳过此步骤

**通用规则（仅需要创建数据库用户时适用）：**
- 禁止使用 root 等超级用户连接应用数据库
- 应用用户仅授予所需库的增删改查权限，不授予 DDL 权限
- 凭据记入 `docs/architecture/architecture.md` 的数据库设计章节

5. 每份设计文档末尾同步产出 OpenSpec design.md 中的引用：
   ```markdown
   # 系统架构 / 接口设计 / 数据库设计
   详见 docs/architecture/architecture.md / docs/api/接口定义.md / docs/database/数据库设计.md
   ```

**② API 接口设计 → docs/api/接口定义.md**

1. 读取 `openspec/specs/{domain}/spec.md`（需求字段定义）+ `docs/architecture/architecture.md`（模块划分），列出全部接口清单（**含业务 API 和系统管理 API：用户/部门/机构/角色/菜单/功能/日志/认证**）
2. 每个接口写明：
   - 请求方式（GET/POST/PUT/DELETE）
   - URL 路径（含路径参数，如 `/api/projects/{id}`）
   - 请求参数：
     - **POST/PUT：** JSON 请求体结构（字段名/类型/是否必填/校验规则/示例值）
     - **GET/DELETE：** query string 参数（参数名/类型/是否必填/说明）
   - 响应体 JSON 结构（字段名/类型/说明）
   - 权限要求（哪些角色可以访问）
3. 状态码和错误码统一放在文档末尾

**③ 数据库设计 → docs/database/数据库设计.md + sql/init.sql**

1. 读取上一阶段产出物（`openspec/specs/{domain}/spec.md` + `docs/architecture/architecture.md` + `docs/api/接口定义.md`），设计全部表结构（**含系统管理表：用户/部门/机构/角色/菜单/功能/日志**）
2. 产出数据库设计说明书 → docs/database/数据库设计.md：
   - 表名、字段名、类型、约束、默认值、备注
   - 主键/外键/唯一索引/普通索引
   - 表间关系说明
   - 枚举值定义
   - **命名规范：** 表名以 `tb_` 开头（如 `tb_user`），字段用下划线命名（如 `user_name`、`create_time`），主键统一 `id`，时间字段统一 `create_time`/`update_time`
3. 产出可执行 DDL → sql/init.sql（含所有表结构 + 预置数据 + 索引）

**④ 编写 Plan → docs/plans/{slug}.md**

1. 画依赖图：数据库 → API → 前端
2. 垂直切片：每条功能完整路径一次做完
3. 每个任务粒度合理（含目标/文件路径/预计耗时/代码片段/验证方式），一个功能 3~8 个任务
4. **对照设计文档编写任务：** `read_file docs/architecture/architecture.md` + `docs/api/接口定义.md` + `docs/database/数据库设计.md` + `docs/plans/{slug}.md`，确保 Plan 中的每个任务能在设计文档中找到依据
   - 每个接口对应一个 API 任务
   - 每个数据库表对应一个数据层任务
   - 每个页面组件对应一个前端任务
   - **系统管理模块（用户/部门/机构/角色/菜单/功能/日志/认证）的 API 也要纳入 Plan**
   - 有前端时：原型项目已存在于 `docs/prototype/`，Plan 中的每个前端任务应明确是在原型组件上新增还是修改
5. 有前端：每个 Task 注明以下信息
   - **页面路由**（对应页面原型中的 URL 路径）
   - **对接 API**（标注调用 docs/api/接口定义.md 中的哪个接口）
6. 产出 Plan → docs/plans/{slug}.md
7. 同步产出 OpenSpec tasks.md（内容为指向 Plan 的标准引用）
   ```
   # 任务分解
   任务分解详见 [docs/plans/{slug}.md](../../docs/plans/{slug}.md)
   ```
8. openspec validate → 检查 tasks.md 结构完整性
9. **创建 OpenSpec change（用于追踪需求实现状态）：** `openspec new change {功能名} --description "feat: {功能名}"`

**展示全部设计文档给用户一次性确认：** 展示 docs/architecture/architecture.md + docs/api/接口定义.md + docs/database/数据库设计.md + docs/plans/{slug}.md → 确认后继续，有修改则修正后重新展示。

10. **提交（main 分支）：**
    ```bash
    git add docs/architecture/ docs/api/ docs/database/ docs/adr/ docs/plans/ sql/init.sql
    git commit -m "feat: {功能名} architecture, design and plan"
    git push origin main
    ```

---

### 3. 开发实施

**分支策略：** 开发前 `git checkout -b feat/{功能名}`，每任务提交到该分支。

### 先加载上一阶段产出物：
```
read_file openspec/specs/{domain}/spec.md — 了解需求范围
read_file docs/architecture/architecture.md — 了解架构设计
read_file docs/api/接口定义.md — 了解 API 设计
read_file docs/database/数据库设计.md — 了解数据库设计
read_file docs/plans/{slug}.md — 加载执行 Plan
```

**🔍 功能覆盖检查：** 对比 `docs/api/接口定义.md` 中的接口列表与 Plan 中的 Task 列表，确保没有接口遗漏实现
- Plan 中每个 Task 应对应至少一个 API 接口
- 接口定义中的每个 endpoint 都应在 Plan 中有对应 Task

### 按 Plan 逐任务自动执行（LLM 全过程自主执行，不追问用户）：

**第一步：设置 todo 列表展示全部 Task**
- 读取 Plan 中的所有 Task，写入 todo（id/status=pending）
- 用户可实时看到 todo 列表知晓全部进度（pending/in_progress/completed）
- 每开始一个 Task → `todo` status=in_progress
- 每完成一个 Task → `todo` status=completed
- 遇阻塞 → `todo` status=cancelled + 新 task 接替

**第二步：使用子代理执行（粘贴以下上下文模板到 delegate_task）：**

```markdown
TDD（RED→GREEN→REFACTOR），Plan：docs/plans/{slug}.md

Step 0：执行 sql/init.sql 建库建表（已存在则跳过）
⚠️ 禁止修改数据库连接配置
【数据库连接检查】按技术栈选：
  - MySQL：mysql -u {应用用户} -p'{应用密码}' -e "SELECT 1;"
  - SQLite：文件存在即 OK，跳过
【实体字段映射检查】按技术栈选：
  - Java/MyBatis：Entity 类须包含 resultMap 全部字段（createTime/updateTime 等）
  - Python/SQLAlchemy：Model 类字段须与表字段一致
  - 其他 ORM：确保实体模型与 DDL 字段对齐

按 Plan 切片逐任务：RED→写单元测试+集成测试 → GREEN→实现 → REFACTOR→重构

【有前端时】原型 docs/prototype/，规则：
1. 静态数据展示 → API 动态渲染，保留布局和交互
2. Mock 数据定义 → 映射 API 响应字段（字段名一致）
3. 筛选/表单/表格列 → 对应 API 参数/请求体/响应字段
4. 路由路径和模块划分保留不变
5. 权限规则保留不变
6. 视觉风格保留，禁止自创样式

数据替换：
- Vue：mounted() 调 API → data 替换静态数据 → 表单 axios.post/put
- 模式：page2Data.staticData → fetch('/api/xxx').then(r=>r.json()).then(data=>data.list)
```

**第三步：子代理执行结束后，逐任务验证**
- 后端：`mvn compile` 编译通过
- 前端：`vite build` 构建通过
- 出问题：自动修复 → 重新编译 → 直到通过

**第四步：全栈联调**
- 启动后端（spring-boot:run）
- 启动前端（vite --port）
- 浏览器验证关键页面（列表/新增/编辑/删除 各模块走一遍）
- 发现 bug → 自动修复 → 重新验证

**第五步：提交**
- `git add . && git commit -m "feat: {功能描述}"`

---

### 4. 测试保障

**先加载上一阶段产出物：**
```
read_file openspec/specs/{domain}/spec.md — 了解需求范围
read_file docs/plans/{slug}.md — 了解任务分解和验证方式
read_file docs/api/接口定义.md — 了解 API 接口用于集成测试
```

**🔍 功能覆盖检查：** 对比 Plan 中的 Task 列表与测试案例，确保每个功能点都有测试覆盖
- 每个 API endpoint 至少有一个正向测试案例
- 每个业务规则（如仅待审核可编辑）至少有一个反向测试案例
- 每个字段的边界条件至少有一个边界值测试案例

**📋 问题跟踪：** 所有测试发现的问题，无论发现阶段，统一登记到 `docs/test/issues.md`
- 格式：`| 模块 | 问题描述 | 发现阶段 | 修复方式 | 状态 |`
- 修复后更新状态为 ✅ 已修复，并补充修复方式
- 遗留问题保持状态为 ⏳ 待修复

---

#### 步骤 1：端到端集成测试（API → DB）

**编写测试案例：** 对照 `openspec/specs/{domain}/spec.md` 逐条编写
- 每个 Requirement → 至少一个**正向案例**（正常流程）
- 每个约束条件 → 至少一个**反向案例**（异常/非法输入）
- 每个数值字段 → 至少一个**边界值案例**（0/负数/最大值/最小值）
- 每案例格式：`### TC-{模块}-{P/N/B}-{序号}: 描述` → 保存到 `docs/test/test-cases.md`

**执行测试：**
- 逐条执行，**一条不落**
- 每个案例记录实际结果 ✅ 或 ❌（含失败原因）
- 发现问题 → 立即登记到 `docs/test/issues.md`
- 修复 bug → 回归测试 → 更新 issues.md 状态

**产出物：** `docs/test/test-cases.md`（含全部执行结果标记）

---

#### 步骤 2：页面结构检查（有前端时）
- 路由检查：查看 `router/index.js`，确认所有页面路由已注册
- 组件检查：确认每个路由对应的视图组件存在
- API 对接检查：确认每个页面覆盖了全部后端接口
- 发现问题 → 登记到 `docs/test/issues.md`

---

#### 步骤 3：用户测试（Playwright 有头模式）

**编写用户测试案例：** 对照 `openspec/specs/{domain}/spec.md` + 原型逐功能编写
- 每个 Scenario → 至少一条用户测试案例
- 每案例格式：`### UT-{模块}-{序号}: 描述` → 保存到 `docs/test/user-test-cases.md`
- 描述清楚：操作步骤（点击/输入/选择）和预期结果

**编写 Playwright 脚本：** 保存到 `docs/test/e2e.cjs`
- 脚本头设一个 `DEMO_MODE` 开关，默认 `false`（正常速度）
- DEMO_MODE = false：`headless: false`，`fill()` 快速填表，正常速度跑完
- DEMO_MODE = true：`slowMo: 300`，`keyboard.type()` 逐字输入，加操作间延时
- 用户说「看看效果/演示一下」时，LLM 自动改为 `true` 再运行，不用用户手动改
- 覆盖全部用户测试案例，逐条验证

**执行测试：**
- 逐条执行，**一条不落**
- 每个案例记录实际结果 ✅ 或 ❌
- 发现问题 → 立即登记到 `docs/test/issues.md`
- 修复 bug → 回归测试 → 更新 issues.md 状态

**产出物：** `docs/test/user-test-cases.md`（含全部执行结果标记）+ `docs/prototype/e2e.cjs`

---

#### 步骤 4：清理测试数据
- 删除测试过程中新增的垃圾数据（如 Playwright 新增的测试记录）
- 清理测试截图（可选）
- 恢复环境到可验收状态

---

#### 步骤 5：汇总测试报告
- 写入 `docs/test/reports/{功能名}-test-report.md`
- 内容：汇总数据（正向/反向/边界值分类）、执行记录、关键验证点、以及测试过程中发现的**所有问题清单**（含严重度和状态）

---

**🔴 纪律（必须遵守）：**
- 每个步骤**不能跳过**、不能简化
- 发现 bug → 先登记到 issues.md → 再修复 → 回归 → 更新状态
- 测试案例必须**全部执行完毕**，不能只跑一部分
- 测试数据清理完成前，不能提交给用户验收
- 诚实对待每个失败案例，不编造通过结果

**提交（feat 分支）：**
```bash
git add docs/test/ src/test/
git commit -m "test: {功能名} integration + user tests, {命中数}/{总数} pass"
git push origin feat/{功能名}
```

---

### 5. 审查与归档

**先加载上一阶段产出物：**
```
read_file openspec/specs/{domain}/spec.md — 了解需求范围
read_file docs/plans/{slug}.md — 了解任务分解
read_file docs/test/reports/{功能名}-test-report.md — 了解测试结论和遗留问题
```

自动扫描 → 质量审查 → 安全审查，结论写入审查报告后归档合并。Code Review 在本阶段完成，PR 仅用于 Merge。

> ⚠️ 确保已提交待审查内容（`git log` 有记录），否则审查会失败。

自动扫描：
  - `semgrep --config=auto .`
  - `gitleaks detect --source . -v`

质量审查（六维：正确性/可读性/架构/安全/性能/品味）：
  - 抽查 `openspec/specs/{domain}/spec.md` 中 20% 的需求条目，确认实现与需求一致
  - 产出审查报告 `docs/reviews/{功能名}-{日期}.md`

安全审查（在质量审查之后执行），逐项检查并将结论追加到审查报告：

```text
# 认证
□ 密码使用 bcrypt/scrypt/argon2 加密（salt rounds ≥ 12）
□ Session token 设置 httpOnly、secure、sameSite
□ 登录接口有频率限制
□ 密码重置 token 有时效

# 鉴权
□ 每个接口校验用户权限
□ 用户只能操作自己的数据
□ 管理员操作需校验角色

# 输入
□ 所有用户输入在边界层校验
□ SQL 查询参数化
□ HTML 输出编码转义

# 数据
□ 代码和版本控制中无密钥硬编码
□ API 响应中排除敏感字段

# 基础设施
□ 安全响应头已配置（CSP、HSTS 等）
□ CORS 限制为已知域名
□ 依赖已审计漏洞
□ 错误信息不暴露内部细节
```

**归档（审查通过后执行）：**
```bash
git push origin feat/{功能名}
gh pr create --title "feat: {功能名}" --body "详见 openspec/specs/{domain}/spec.md" --base main
git checkout main && git pull origin main && git branch -d feat/{功能名}
```

---

### 6. 部署上线与验收

1. **构建项目：** 按 Step 0 确定的技术栈执行构建命令
   - **Spring Boot：** `mvn clean package -DskipTests`
   - **Flask / FastAPI：** 无打包步骤（直接启动源码）
   - **前端（有前端时）：** `npm run build`

2. **启动服务：** 按对应技术栈启动，验证服务可访问
   - **Spring Boot：** `java -jar target/*.jar`
   - **Flask：** `flask run` 或 `python app.py`
   - **FastAPI：** `uvicorn main:app --reload`
   - **前端（有前端时）：** 启动静态服务或 dev server

3. **用户验收：** 通知用户浏览器打开 `http://localhost:{端口}` 逐功能验收。问题登记到 `docs/test/issues.md`，修复后重新构建。

4. **打版本 tag：** `git tag v1.0.0 && git push origin --tags`（版本号按实际迭代递增）

---

### 7. 回顾总结

**先加载上一阶段产出物：**
```
read_file openspec/specs/{domain}/spec.md — 了解需求范围
read_file docs/test/reports/{功能名}-test-report.md — 了解测试结论和问题
read_file docs/reviews/{功能名}-{日期}.md — 了解审查报告
```

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
- 模板/规范缺失 → 更新原型模板或流程文档

**提交（main 分支）：**
```bash
git add docs/retrospectives/
git commit -m "feat: {功能名} retrospective"
git push origin main
```

---