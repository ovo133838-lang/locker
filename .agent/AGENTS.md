# AGENTS.md — 优小宝智能寄存柜 · 项目开发 Agent

> 本文件位于后端仓库最外层 `gin-master/.agent/`，是整个项目对 AI 编码助手的唯一入口。任何 Agent 进入本项目，先读本文件，再按第 2 节的索引读取对应上下文；不要凭记忆写字段、写规则。

## 1 角色与目标

你是本项目的实现 Agent，负责把已经冻结的需求与契约落成可运行代码：一个 Go 单体服务（C 端 + B 端 + 定时任务 + 柜体适配）、一个微信小程序、一个管理后台。你的产出必须同时满足三件事：页面能走通、规则能复算、异常能追溯。

工作方式：先读契约 → 再改代码 → 跑检测 → 出证据。没有契约依据的字段和逻辑，一律视为待确认，不得自行发明。

## 2 上下文索引（按需读取，不要一次全读）

| 你要做什么 | 必读 |
| --- | --- |
| 任何任务开始前 | `context/requirements.md`（功能编号与验收项）、`agent.yaml`（技术栈与门禁） |
| 写接口、改出入参 | `context/api-contract.md` |
| 建表、改字段、写迁移 | `context/data-model.md` |
| 算钱、结算、退款、补交 | `context/billing.md`（含 6 组算例，必须逐条对上） |
| 排期、拆任务、判断做到哪一步 | `context/plan.md` |
| 作为开发 Agent 执行 | `prompts/developer.md` |
| 作为检测 Agent 执行 | `prompts/inspector.md` |
| 提测 / 验收 / 上线前 | `checklists/gate.md` |

外部 Skill（若宿主环境提供，名称固定）：`locker-dev` 提供开发契约与代码骨架，`locker-check` 提供静态扫描、计费回归、接口冒烟与报告生成。两个 Skill 的 `references/` 与本目录 `context/` 口径必须一致，冲突时以本项目文档与 `context/` 为准（它们是本项目的唯一事实来源），并回写修正 Skill。

## 3 文档优先级（冲突时的裁决顺序）

1. 《优小宝智能寄存柜_业务需求说明书》—— 业务规则、状态定义、验收标准。
2. 《智能寄存柜_接口文档》—— 路径、参数、错误码。
3. 《智能寄存柜_数据字典》—— 表结构、字段类型、约束。
4. 《智能寄存柜_开发前准备》—— 技术方案、WBS、风险。
5. `.agent/context/` —— 上述四份的提炼，用于快速定位，不替代原文。
6. 交互原型 `优小宝寄存柜原型.html` —— 页面结构与交互细节；规则冲突时以第 1 项为准。

金额口径的唯一裁决：**存储与传输一律用「分」的整数，展示层转元**。需求说明书正文里的「元、保留两位小数」是展示口径，不是存储口径，不得据此把金额字段改成 DECIMAL 或 float。

## 4 仓库与目录

本文件位于后端仓库最外层：`D:\Desktop\寄存柜\gin-master\.agent\`。以本文件所在目录（即 `gin-master`）为工作根，项目根在其上一层，路径写作 `../`。

```text
gin-master/                     # 后端仓库 = 本 Agent 的工作根，远端 locker，分支 main + dev
├── .agent/                     # 本文件与上下文（Agent 唯一入口）
├── cmd/api/main.go             # 入口：配置、DB、路由、定时任务
├── internal/{config,middleware,model,repository,service,handler,cabinet,job,pkg}/
├── miniprogram/{pages,components,utils,api}/
├── deploy/                     # docker-compose.yml、nginx.conf、迁移脚本
└── go.mod

../                             # 项目根
├── vue-element-admin-master/   # 后台仓库，远端 locker-Backend，分支 main + dev
└── 优小宝寄存柜原型.html 与 4 份 Word 文档
```

约定：后端业务代码写在本仓库（`gin-master`）内，不改动 gin 框架源码；后台业务页面写在 `../vue-element-admin-master` 内，复用其布局、动态路由、权限与 `utils/request`。两个仓库各自提交，不互相引用本地绝对路径。

## 5 七阶段流程与门禁

| 阶段 | 产出 | 通过条件（门禁） |
| --- | --- | --- |
| S1 需求分析 | 需求说明书、FR 编号 | FR-C-01~10、FR-B-01~11 全部有页面与规则，无「待定」项进入开发 |
| S2 原型设计 | 高保真原型 | C 端 10 页、B 端 9 页可点通，无假按钮；金额可复算 |
| S3 任务规划 | WBS、里程碑、人力 | 15 项任务有前置关系，人日合计 32.5，M1–M7 判定标准明确 |
| S4 开发 | 后端 / 小程序 / 后台代码 | 按第 7 节顺序推进，每个里程碑可独立验收 |
| S5 测试 | 单测 + 集成 + 冒烟 | 计费 14 条算例全绿，覆盖率 ≥ 70%，P0 静态扫描为零 |
| S6 验收 | 验收报告 + 证据 | AC-01~16 全部通过，数据不变量 SQL 无非空结果 |
| S7 部署上线 | 镜像、Compose、Nginx | 生产环境完成一次真实开门与一次真实退款验证 |

每跨一个阶段，先在 `checklists/gate.md` 勾完对应项，再进入下一阶段。门禁不通过时不允许「先上线再补」。

## 6 五条硬约束（违反即返工）

1. **金额只用分**：存储、计算、接口出入参都是整数分，字段以 `_amount` 结尾；禁止 float、禁止字符串参与运算；展示层转元。
2. **规则快照不可变**：下单时把单价、封顶、押金、免费时长快照进 `orders.rule_snapshot`，订单生命周期内只读快照，不读最新规则；改价只影响新订单。
3. **一格口一笔进行中订单**：用「格口行条件更新 + 订单唯一约束」保证，不靠应用层先查后写；影响行数为 0 立即返回占格错误。
4. **开门等回执**：未收到柜体成功回执不得把订单置为「进行中」；失败写 `door_logs`，允许重试一次。
5. **外部回调幂等**：支付以 `trade_no`、柜体以 `(order_id, request_id)` 建唯一索引；重复回调返回成功且不重复执行业务。

## 7 开发顺序（自底向上，禁止先做页面）

1. 迁移与模型（按 `context/data-model.md` 建表与索引，空库可重复执行）
2. 计费引擎（纯函数 `CalcFee`，先让算例全绿）
3. 网点 / 柜组 / 格口查询
4. 下单锁格
5. 支付（下单参数、回调验签、幂等置成功、异步开门）
6. 开门（`CabinetAdapter` + mock + 回执落 `door_logs`）
7. 结算（结束、退款、补交、释放格口）
8. 定时任务（超时扫描、支付超时、离线检测、概览汇总）
9. B 端接口（登录、概览、网点、柜组、订单、营收、异常、规则）
10. 前端（小程序 10 页、后台 9 页，按契约取数，不得自造字段）

## 8 技术栈（锁定，不得擅自替换）

| 层 | 选型 | 版本 |
| --- | --- | --- |
| 后端 | Go | 1.26（gin v1.12.0 的 go.mod 要求） |
| HTTP | gin | v1.12.0（本地 `gin-master`） |
| ORM | GORM | v1.25 |
| 数据 | MySQL / Redis | 8.0 / 7.x |
| C 端 | 微信小程序原生 | WXML/WXSS/JS，不用 uni-app、Taro |
| B 端 | vue-element-admin | 4.4.0（Vue 2.6.10 + Element UI 2.13.2 + Router 3 + Vuex 3 + Vue CLI，端口 9527） |
| 部署 | Docker Compose + Nginx | 静态资源 dist + `/api/v1` 反代 |

后台页面用 Vue2 选项式 API（`data` / `computed` / `methods`），表格 `el-table` + 基座 `Pagination`，请求从 `@/utils/request` 引入；不引入 Vue3、Element Plus、Pinia。

## 9 常用命令

```bash
# 后端（在 gin-master 内）
go run ./cmd/api                 # 启动，默认 8080
go build ./... && go test ./...  # 编译与单测
go test ./internal/service/billing -run TestCalcFee -v

# 后台（在 vue-element-admin-master 内）
npm install
npm run dev        # 9527
npm run build:prod # dist

# 检测（locker-check skill 的 scripts 目录，在 gin-master 根执行）
python scripts/scan_project.py --root . --out reports/static.json
python scripts/check_billing.py --out reports/billing.json
python scripts/api_smoke.py --base-url http://127.0.0.1:8080 --phone 18855558888 --password 123456
python scripts/make_report.py --dir reports
```

Windows 环境：命令逐条执行，不串 `&&` 以外的复合语法；复杂逻辑写 `.py` 脚本，读写文件显式 `encoding='utf-8'`。

## 10 分支与提交

- 分支：`main` 为可上线版本，`dev` 为集成分支，特性分支 `feat/<任务号>-<简述>`，修复分支 `fix/<缺陷号>`；合并前先 rebase `dev`。
- 提交信息：`<类型>(<范围>): <做了什么>`，类型取 `feat|fix|refactor|test|docs|chore`，范围取 `order|billing|auth|site|cabinet|admin|mini|deploy`。示例：`feat(billing): 实现按日模式并与算例对齐`。
- 一次提交只做一件事；契约变更必须在提交正文写明影响的页面与接口。
- 不提交 `node_modules`、`dist`、本地配置文件、真实密钥与商户号。

## 11 禁止清单

1. 禁止把金额写成 float / decimal / 元为单位的整数。
2. 禁止在 handler 里直接写 SQL 或开事务，事务只在 repository/service 边界。
3. 禁止先查后写实现锁格、禁止用 Redis 分布式锁替代数据库条件更新。
4. 禁止在支付回调的事务内调用柜体开门。
5. 禁止前端自算金额、自造字段、自造状态值。
6. 禁止把未实现逻辑用空对象或固定值冒充成功（检测会把这类判为 P0）。
7. 禁止修改 `locker-check` 的断言或跳过用例来让报告变绿。
8. 禁止在代码、日志、提交信息中出现真实商户号、证书、用户手机号明文。

## 12 每次交付前必须给出的三样证据

1. 计费自检输出（算例逐条 PASS）。
2. 静态扫描与冒烟结果（`reports/*.json` 摘要数字）。
3. 本次改动的接口 / 字段清单（对应到文档章节），以及未做的部分和原因。

（本文件完）
