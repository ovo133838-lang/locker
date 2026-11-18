# 数据模型（摘要）

> 全文见《智能寄存柜_数据字典.docx》（13 张表，逐字段类型/必填/默认/说明）。本文件给 Agent 建表与写迁移时对照关键结构，字段全量以字典为准。

## 1 表清单

| # | 表 | 用途 | 行量（单网点 3 年） |
| --- | --- | --- | --- |
| 1 | users | C 端用户 | 3 万 |
| 2 | admins | 后台管理员 | 20 |
| 3 | sites | 网点 | 10 |
| 4 | cell_groups | 柜组 | 30 |
| 5 | cells | 格口 | 300 |
| 6 | pricing_rules | 收费规则版本 | 200 |
| 7 | orders | 订单主表 | 6 万 |
| 8 | door_logs | 开门记录 | 15 万 |
| 9 | payments | 支付流水 | 6 万 |
| 10 | refunds | 退款流水 | 1 万 |
| 11 | alerts | 异常记录 | 5 千 |
| 12 | coupons | 优惠券（一期空表） | 0 |
| 13 | audit_logs | 状态变更审计 | 20 万 |

## 2 关键字段

**sites**：`id, name(uk), addr, type, open_time, close_time, lat, lng, manager, manager_phone, status, photo`。营业时间拆两个 TIME 字段，便于判断营业中。

**cell_groups**：`id, site_id, name, master_sn(uk), firmware, online, last_heartbeat_at, cells_total`。`online` 由心跳维护，不手工改。

**cells**：`id, group_id, site_id, cell_no, size(S/M/L), status, current_order_id, door_closed, last_open_at, remark`。
唯一索引 `uk_group_cell(group_id, cell_no)`，普通索引 `idx_site_status(site_id, status)`。

**pricing_rules**：`id, site_id, version, mode(hour/day), free_minutes, detail(JSON), status(active/inactive), effective_at, created_by`。
`detail` 形如：
```json
{"S":{"price_amount":500,"cap_amount":5000,"daily_amount":1500,"deposit_amount":1000},
 "M":{"price_amount":700,"cap_amount":7000,"daily_amount":2500,"deposit_amount":2000},
 "L":{"price_amount":1000,"cap_amount":10000,"daily_amount":3000,"deposit_amount":3000}}
```
同一网点同一时刻只允许一条 `status='active'`；保存即新增版本、旧版置 inactive。

**orders**：`id, order_no(uk), user_id, site_id, group_id, cell_id, cell_no, size, mode, plan_minutes, rule_snapshot(JSON), base_amount, deposit_amount, discount_amount, paid_total_amount, actual_fee_amount, refund_amount, extra_amount, status, request_id, pay_expire_at, paid_at, opened_at, closed_at, close_reason`。
唯一索引 `uk_order_no(order_no)`、`uk_user_request(user_id, request_id)`；索引 `idx_cell_status(cell_id, status)`、`idx_user_status(user_id, status)`、`idx_site_status_opened(site_id, status, opened_at)`。

**door_logs**：`id, order_id, cell_id, actor(user/admin/system/device), actor_id, action(open/temp_open/admin_open/open_all/close), request_id, result(success/fail/timeout), cost_ms, remark`。
唯一索引 `uk_order_request(order_id, request_id)` —— 开门幂等的实现点。

**payments**：`id, order_id, trade_no(uk), transaction_id, direction(in/out), amount, channel, status(pending/success/failed), paid_at, raw(JSON)`。

**refunds**：`id, order_id, payment_id, refund_no(uk), amount, reason, status, operator, refunded_at`。

**alerts**：`id, site_id, group_id, cell_no, type, level(err/warn), detail, order_id, handled, handled_by, handled_at, resolution, merged_count`。
索引 `idx_site_handled(site_id, handled)`、`idx_group_type_time(group_id, type, created_at)`（用于 30 分钟合并判重）。

**audit_logs**：`id, target_type, target_id, from_status, to_status, actor, actor_id, request_id, detail`。

## 3 锁格语句（唯一正确写法）

```sql
UPDATE cells
   SET current_order_id = ?, status = 'used', updated_at = NOW()
 WHERE id = ? AND status = 'normal' AND current_order_id IS NULL;
```
影响行数必须为 1 才继续下单事务，0 直接返回 `CELL_OCCUPIED`（40901）。释放：
```sql
UPDATE cells SET current_order_id = NULL, status = 'normal', updated_at = NOW()
 WHERE id = ? AND current_order_id = ?;
```

## 4 不可违反的约束

| 编号 | 约束 | 实现 |
| --- | --- | --- |
| C1 | 一格口一笔进行中订单 | 上面的条件更新 + `cells.current_order_id` 唯一指向 |
| C2 | 订单号唯一 | `uk_order_no` |
| C3 | 创建订单幂等 | `uk_user_request` |
| C4 | 支付回调幂等 | `payments.uk_trade_no` |
| C5 | 开门幂等 | `door_logs.uk_order_request` |
| C6 | 规则快照只读 | 下单写 `rule_snapshot`，之后不更新 |
| C7 | 金额自洽 | `actual_fee_amount + refund_amount - extra_amount = paid_total_amount`（仅 closed 成立） |
| C8 | 退款不超实收 | `refunds.amount` ≤ 对应 payments 已收 |
| C9 | 状态迁移合法 | 服务层状态机 + `audit_logs` |
| C10 | 只分配可用柜门 | `status='normal' AND current_order_id IS NULL` |

## 5 迁移纪律

时间戳命名、只增不删；改类型或删字段单独评审；迁移可重复执行（空库跑两遍结果一致）；每次迁移同步更新本文与数据字典字段说明。

## 6 巡检 SQL（验收必跑）

```sql
-- 1 有进行中订单但格口未指向
SELECT o.id, o.order_no FROM orders o LEFT JOIN cells c ON c.id = o.cell_id
 WHERE o.status IN ('active','overdue') AND (c.current_order_id IS NULL OR c.current_order_id <> o.id);
-- 2 一格口多笔进行中
SELECT cell_id, COUNT(*) FROM orders WHERE status IN ('active','overdue') GROUP BY cell_id HAVING COUNT(*) > 1;
-- 3 关单金额不自洽
SELECT id, order_no, actual_fee_amount, refund_amount, extra_amount, paid_total_amount
  FROM orders WHERE status='closed'
   AND actual_fee_amount + refund_amount - extra_amount <> paid_total_amount;
```
任一返回非空即 P0。
