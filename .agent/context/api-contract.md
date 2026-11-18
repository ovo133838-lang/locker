# 接口契约（摘要）

> 全文见《智能寄存柜_接口文档.docx》（59 个接口，C 24 / B 30 / 柜体 5）。本文件给 Agent 快速比对路径与错误码；字段级定义以全文第 4 章为准。

## 1 统一约定

- 基础路径 `/api/v1`，JSON + UTF-8，HTTPS。
- 响应三段：`{code, msg, data, trace_id}`，`code=0` 成功。
- 金额一律整数分，字段以 `_amount` 结尾；时长整数分钟；时间 RFC3339。
- 分页 `page`（从 1）、`size`（默认 20，上限 100），返回 `page/size/total/list`。
- 鉴权 `Authorization: Bearer <token>`；写操作带 `X-Request-Id`。
- HTTP 语义：200 成功、400 参数、401 未登录、403 越权、404 不存在、409 状态冲突、429 过频、500 异常。

## 2 C 端（24）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | /c/auth/login | 微信 code 换令牌 |
| POST | /c/auth/phone | 绑定手机号 |
| POST | /c/auth/refresh | 刷新令牌 |
| POST | /c/auth/logout | 退出 |
| GET | /c/spots/nearby | 附近网点（lat/lng/radius/keyword） |
| GET | /c/spots/{id} | 网点详情（含三档价格与余量） |
| GET | /c/spots/{id}/avail | 余量轮询 |
| GET | /c/pricing | 收费规则展示 |
| POST | /c/orders/estimate | 预估费用（不锁格） |
| POST | /c/orders | 创建订单并锁格（幂等 request_id） |
| GET | /c/orders/{id}/pay-params | 微信支付参数 |
| POST | /c/pay/notify | 支付回调 |
| GET | /c/orders/{id} | 订单详情（with_logs） |
| GET | /c/orders | 订单列表（status=active/closed） |
| POST | /c/orders/{id}/open | 首次开门 |
| POST | /c/orders/{id}/temp-open | 临时开门 |
| POST | /c/orders/{id}/close | 结束并结算 |
| POST | /c/orders/{id}/extra-pay | 补交差额 |
| GET | /c/orders/{id}/logs | 开门记录 |
| GET | /c/me | 个人中心 |
| PUT | /c/me/profile | 昵称头像 |
| GET | /c/coupons | 优惠券（一期空列表） |
| GET | /c/guide | 寄存指南 |
| GET | /c/config | 客服电话、协议版本 |

## 3 B 端（30）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | /b/auth/login | 管理员登录（phone/password） |
| GET | /b/auth/me | 当前管理员 |
| POST | /b/auth/logout | 退出 |
| GET | /b/dashboard | 概览 |
| GET | /b/sites | 网点列表 |
| POST | /b/sites | 新建网点 |
| GET | /b/sites/{id} | 网点详情 |
| PUT | /b/sites/{id} | 编辑网点 |
| PUT | /b/sites/{id}/status | 启停网点 |
| GET | /b/sites/{id}/groups | 柜组与格口统计 |
| POST | /b/sites/{id}/groups | 新建柜组 |
| PUT | /b/groups/{id}/master | 绑定柜体主机 |
| GET | /b/groups/{id}/cells | 柜门网格 |
| POST | /b/cells/{id}/open | 单门远程开门 |
| PUT | /b/cells/{id}/status | 改柜门状态 |
| POST | /b/groups/{id}/open-all | 全部打开（应急） |
| GET | /b/orders | 订单列表 |
| GET | /b/orders/{id} | 订单详情 |
| POST | /b/orders/{id}/open | 管理员开门 |
| POST | /b/orders/{id}/close | 管理员关单 |
| POST | /b/orders/{id}/refund | 退费 |
| GET | /b/revenue/summary | 营收汇总 |
| GET | /b/revenue/trend | 营收趋势 |
| GET | /b/revenue/sites | 网点明细 |
| GET | /b/revenue/export | 导出 |
| GET | /b/alerts | 异常列表 |
| GET | /b/alerts/{id} | 异常详情 |
| POST | /b/alerts/{id}/handle | 处理异常 |
| GET | /b/pricing | 读取规则 |
| PUT | /b/pricing | 保存规则（生成新版本） |
| GET | /b/pricing/history | 版本历史 |
| GET | /b/admins | 管理员列表 |
| POST | /b/admins | 新增管理员 |
| PUT | /b/admins/{id}/password | 重置密码 |

> 注：表中 B 端列出 33 行，其中 `/b/sites/{id}/groups`、`/b/groups/{id}/master`、`/b/admins` 系列为契约补充项（原型按钮未直接暴露但业务需要），核心 30 个以全文表 3-2 为准。

## 4 柜体协议（5）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | /device/heartbeat | 心跳（master_sn、online_cells、firmware） |
| GET | /device/commands | 长轮询拉指令（30 秒空返回） |
| POST | /device/command-result | 开门回执（order_id、request_id、result、cost_ms） |
| POST | /device/cell-status | 门磁状态上报 |
| POST | /device/alert | 硬件异常上报 |

签名：`sign = HMAC-SHA256(master_sn + timestamp + body, device_secret)`，时间戳偏差 ≤ 5 分钟。

## 5 错误码（17）

| code | HTTP | 含义 | 前端处理 |
| --- | --- | --- | --- |
| 0 | 200 | 成功 | 渲染 data |
| 40001 | 400 | 参数校验失败 | 定位字段，不重试 |
| 40002 | 400 | 金额或时长超范围 | 刷新重选 |
| 40100 | 401 | 未登录 / 令牌失效 | 跳登录 |
| 40101 | 401 | 刷新令牌失效 | 重新微信登录 |
| 40300 | 403 | 跨网点越权 | 提示无权限 |
| 40301 | 403 | 角色无权限 | 隐藏入口 |
| 40400 | 404 | 资源不存在 | 提示并刷新 |
| 40901 | 409 | 格口被占用 | 刷新余量重选柜型 |
| 40902 | 409 | 当前状态不允许该操作 | 刷新订单状态 |
| 40903 | 409 | 幂等命中 | 视为成功 |
| 40904 | 409 | 订单已超时需补交 | 跳补交页 |
| 42900 | 429 | 请求过频 | 退避重试 |
| 50000 | 500 | 内部错误 | 提示并上报 trace_id |
| 50201 | 502 | 柜体通信失败 | 重试一次后联系管理员 |
| 50202 | 502 | 支付渠道失败 | 保留订单稍后重试 |
| 50203 | 502 | 退款失败 | 订单照常结束，转人工 |

## 6 幂等键

| 场景 | 键 | 重复请求 |
| --- | --- | --- |
| 创建订单 | (user_id, request_id) | 返回首单 |
| 开门 / 临时开门 | (order_id, request_id) | 返回首次结果 |
| 结束订单 | order_id + request_id | 返回首次结算 |
| 支付回调 | trade_no | 直接成功不重复执行 |
| 管理员开门 / 退费 | request_id | 返回首次结果 |

## 7 契约纪律

字段一旦确定，前端不得重命名；改接口先改文档再改代码，提交说明列出受影响页面；任何接口返回浮点金额按缺陷处理。
