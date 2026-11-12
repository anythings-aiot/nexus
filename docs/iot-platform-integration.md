# IoT 平台接入指南（nx.iot.v1）

本文面向后续接入 nexus 的 IoT 平台研发与运维，说明平台需要实现的接口、nexus 侧的配置方式，以及 `nx.iot.v1` WebSocket API 的完整协议细节。

协议设计基线见 [nx-iot-v1.md](../design/nx-iot-v1.md)；本文以当前代码实现为准，代码位置：

| 关注点 | 代码 |
| --- | --- |
| 协议信封、常量、错误码 | [frame.go](../../internal/nexus/connector/ws/frame.go) |
| 连接、握手、心跳、重连、ack | [client.go](../../internal/nexus/connector/ws/client.go) |
| 方法参数与结果 | [methods.go](../../internal/nexus/connector/ws/methods.go) |
| 统一能力模型 | [model.go](../../internal/nexus/connector/model.go) |
| 平台配置 | [config.go](../../internal/nexus/config/config.go) |
| REST 兜底适配器 | [connector.go](../../internal/nexus/connector/rest/connector.go) |

## 角色与拓扑

```text
┌──────────────┐        WSS（主通道，nx.iot.v1）        ┌──────────────┐
│              │  ── hello / ping / device.get ... ──▶  │              │
│  nexus (NX)  │  ◀── response / alarm.triggered ... ──  │  IoT 平台    │
│  WS 客户端   │        HTTP（兜底 / 大对象 / 身份）      │  WS 服务端   │
└──────────────┘  ── /internal/v1/nx/*、agent-token ─▶ └──────────────┘
```

- nexus 是 WebSocket **客户端**，平台是**服务端**；平台负责监听、鉴权、路由与事件推送。
- nexus 通过 HTTP 向控制中心（CC）获取 agent 身份令牌，WS 握手时以 Bearer 携带。
- 同一平台方法在 WS 与 HTTP 上共享参数/结果语义；WS 不可用时可按配置回退 HTTP。

## 接入方式选型

`platforms[]` 的 `kind` 决定使用哪种适配器：

| kind | 适用场景 | 事件入站 | 出站查询/动作/回报 | 说明 |
| --- | --- | --- | --- | --- |
| `ws` | 平台能提供常驻 WebSocket 入口 | WS `events` 通道 | WS `commands` 通道 | **推荐**，低时延、支持续传与顺序保证 |
| `http` | anythings 控制中心既有 HTTP API | `POST /internal/v1/nx/triggers/alarm` | CC HTTP API | 兼容路径，作为兜底 |
| `rest` | 任意第二平台，只想接最小 HTTP 契约 | `POST /internal/v1/platforms/{platform}/events` | `POST/GET {base_url}/internal/v1/nx/*` | 按 `capabilities` 声明能力 |

> 当存在启用的 `kind=ws` 平台时，nexus 以该 WS 平台作为活动通道；`nx_agent.trigger.source=http` 时才额外保留 HTTP 触发入口。

## 快速接入

### 平台侧最小实现清单

1. 提供 WSS 入口（本地联调允许 `ws://`），路径建议 `/nexus/v1/ws`。
2. 协商子协议 `nx.iot.v1`，并在握手请求中校验 `Authorization: Bearer <token>`。
3. 实现 `hello@v1`、`ping@v1`。
4. 按「方法白名单」实现所需方法；未实现的方法返回 `CAPABILITY_UNSUPPORTED` 或 `METHOD_NOT_FOUND`。
5. 通过 `events` 通道推送 `alarm.triggered@v1` / `alarm.recovered@v1`，并处理 nexus 的 `ack`。
6. 按「错误码」返回结构化错误帧，按「重连与续传」配合重连。

### nexus 侧配置

```json
{
  "platforms": [
    {
      "platform_id": "anythings",
      "kind": "ws",
      "enabled": true,
      "default": true,
      "tenant": "default",
      "ws_url": "wss://cc.example.com/nexus/v1/ws",
      "token": "<platform-token>",
      "subprotocol": "nx.iot.v1"
    }
  ],
  "nx_agent": {
    "enabled": true,
    "trigger": { "source": "ws", "report_retry": 3 },
    "execution_mode": "manual",
    "aggregation_window": "30s"
  }
}
```

必填与默认值：

| 字段 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- |
| `platform_id` | 是 | — | 平台唯一标识，写入 `meta.platform` 与事件 `platform_id` |
| `kind` | 是 | — | `ws` / `http` / `rest` |
| `enabled` | 是 | `false` | 仅启用平台会建立通道 |
| `ws_url` | `kind=ws` 时必填 | — | WebSocket 入口地址 |
| `token` | `kind=ws` 时必填 | — | 握手 Bearer 令牌 |
| `subprotocol` | 否 | `nx.iot.v1` | 子协议名 |
| `tenant` | 否 | `default` | 写入 `meta.tenant` |
| `default` | 否 | `false` | 最多一个启用平台可设为默认 |

## 连接与握手

### 建立连接

```http
GET /nexus/v1/ws HTTP/1.1
Host: cc.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Protocol: nx.iot.v1
Authorization: Bearer <platform-token>
```

- 子协议必须唯一协商为 `nx.iot.v1`，否则 nexus 主动断开。
- 仅生产环境建议使用 `wss://`；本地联调允许 `ws://`。
- 单帧默认上限 1 MiB，可由握手响应 `max_frame` 收紧。

### hello@v1

连接建立后，nexus 立即在 `control` 通道发送握手请求：

```json
{
  "v": 1,
  "type": "request",
  "id": "nx-1730000000000000000-123456",
  "channel": "control",
  "method": "hello@v1",
  "params": {
    "platform": "anythings",
    "tenant": "default",
    "space": "default",
    "last_seq": 0
  },
  "meta": { "platform": "anythings", "tenant": "default", "space": "default", "client": "nexus", "version": "1.0.0" }
}
```

平台应返回：

```json
{
  "v": 1,
  "type": "response",
  "id": "nx-1730000000000000000-123456",
  "ok": true,
  "result": {
    "session_id": "01J...",
    "resume_supported": true,
    "max_frame": 1048576,
    "heartbeat_s": 15,
    "reconnect_after_ms": 0
  }
}
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `session_id` | string | 是 | 本次会话标识，用于续传关联 |
| `resume_supported` | bool | 否 | 是否支持按 `last_seq` 重放 |
| `max_frame` | int | 否 | 平台侧单帧上限；小于客户端上限时生效 |
| `heartbeat_s` | int | 否 | 心跳间隔秒数，覆盖默认 15s |
| `reconnect_after_ms` | int | 否 | 建议重连等待，优先于指数退避 |

握手失败时平台应发送 `error` 帧或以关闭码结束连接：

| 关闭码 | 含义 |
| --- | --- |
| `4401` | 鉴权失败（令牌缺失/无效） |
| `4403` | 空间/租户不匹配或无权限 |
| `4408` | 握手超时 |

> 当前实现中 `hello.params.space` 固定为 `default`；后续方法调用会按业务凭据在 `meta.space` 中覆盖实际空间。

## WebSocket API

### 帧信封

请求、响应、事件、确认、错误共用同一 JSON 信封：

```json
{
  "v": 1,
  "type": "request",
  "id": "01J...",
  "channel": "commands",
  "method": "device.get@v1",
  "params": { "device_id": "dev_01" },
  "meta": { "platform": "anythings", "tenant": "default", "space": "default", "deadline_ms": 5000 }
}
```

```json
{
  "v": 1,
  "type": "response",
  "id": "01J...",
  "ok": true,
  "result": { "data": { "device_id": "dev_01", "online": true } },
  "meta": { "elapsed_ms": 12 }
}
```

```json
{
  "v": 1,
  "type": "event",
  "channel": "events",
  "event": "alarm.triggered@v1",
  "seq": 42,
  "event_id": "01J...",
  "data": { "...": "见「事件载荷」" },
  "meta": { "platform": "anythings", "space": "default" }
}
```

```json
{ "v": 1, "type": "ack", "channel": "events", "seq": 42 }
```

```json
{
  "v": 1,
  "type": "error",
  "id": "01J...",
  "code": "PERMISSION_DENIED",
  "message": "action not allowed",
  "retryable": false,
  "details": {
    "role_id": "system.nx-agent",
    "required_permission": "action:execute:*",
    "resource": "device:dev_01:action:reboot"
  }
}
```

字段说明：

| 字段 | 类型 | 出现帧 | 说明 |
| --- | --- | --- | --- |
| `v` | int | 全部 | 协议版本，必须为 `1` |
| `type` | string | 全部 | `request` / `response` / `event` / `ack` / `error` |
| `id` | string | request/response/error | 请求标识，同一连接内唯一；响应与错误必须回填 |
| `channel` | string | request/response 可省/event/ack | `control` / `events` / `commands` / `telemetry` |
| `method` | string | request | `namespace.action@version`，如 `data.latest@v1` |
| `params` | object | request | 方法参数 |
| `ok` | bool | response | 是否成功；失败通常改用 `error` 帧 |
| `result` | object | response | 成功结果；业务数据建议包在 `data` 字段内 |
| `event` | string | event | 事件名，如 `alarm.triggered@v1` |
| `seq` | int64 | event/ack | 事件在 `(platform, space)` 流内的递增序号 |
| `event_id` | string | event | 事件全局唯一标识，用于去重 |
| `data` | object | event | 事件载荷 |
| `code` | string | error | 错误码，见「错误码」 |
| `message` | string | error | 错误描述 |
| `retryable` | bool | error | 是否可重试 |
| `details` | object | error | 附加信息，如 `required_permission` |
| `meta` | object | 全部 | 路由、超时、幂等与耗时元数据 |

`meta` 字段：

| 字段 | 出现位置 | 说明 |
| --- | --- | --- |
| `platform` / `tenant` / `space` | 请求 | 平台、租户、空间；NX 自动填充 |
| `deadline_ms` | 请求 | 本次调用剩余超时毫秒数 |
| `idempotency_key` | 请求 | 幂等键；写操作必填 |
| `client` / `version` | 握手请求 | 固定 `nexus` 与客户端版本 |
| `elapsed_ms` | 响应 | 平台处理耗时 |

### 逻辑通道与并发

| channel | 承载 | NX 侧并发上限 | 背压/顺序 |
| --- | --- | --- | --- |
| `control` | `hello@v1`、`ping@v1` | 4 in-flight | 超时取消 |
| `events` | 告警触发/恢复等事件 | 单协程顺序消费 | ACK 窗口 100，超窗暂停推送 |
| `commands` | 查询、动作、命令状态、回报 | 8 in-flight | 超过 `deadline_ms` 取消 |
| `telemetry` | 高频时序（预留） | — | 未实现时返回 `CAPABILITY_UNSUPPORTED` |

### 方法调用约定

- `request.id` 由 nexus 生成，响应必须回填同一 `id`；未匹配的响应会被丢弃。
- 调用超时取 `RequestTimeout`（默认 5s）与调用方 deadline 的较小值，超时后 nexus 不再接受该 `id` 的迟到响应。
- 平台返回 `error` 帧时，`code=PERMISSION_DENIED` 会映射为权限错误并保留 `details`；`code=TIMEOUT` 可用 `errors.Is(err, ws.ErrTimeout)` 判定。
- 所有写操作（`action.execute@v1`、`handling.report@v1`）必须按 `meta.idempotency_key` 幂等处理。

### 方法白名单（v1）

| method | channel | 所需权限（nx-agent 当前角色） | 说明 |
| --- | --- | --- | --- |
| `hello@v1` | control | — | 握手 |
| `ping@v1` | control | — | 心跳 |
| `device.get@v1` | commands | `device:read:*` | 设备详情 |
| `status.get@v1` | commands | `status:read:*` | 设备在线/状态 |
| `data.latest@v1` | commands | `data:read:*` | 最新数据 |
| `action.execute@v1` | commands | `action:execute:*`（自动执行另需 `action:auto_execute:<code>`） | 动作提议/执行 |
| `command.get@v1` | commands | `command:read:*` | 命令状态 |
| `handling.report@v1` | commands | `alarm:write:*` | 处置回报，幂等 |

非白名单方法返回 `METHOD_NOT_FOUND`；参数非法返回 `VALIDATION_FAILED`；能力未声明返回 `CAPABILITY_UNSUPPORTED`。

#### device.get@v1

查询设备详情，用于设备状态类查询。

请求：

```json
{ "device_id": "dev_01", "attribute_key": "temperature" }
```

响应 `result.data`：

```json
{ "device_id": "dev_01", "status": "online", "name": "1# 风机" }
```

nexus 主要读取 `device_id` 与 `status`，`status` 会作为读数 `quality` 使用。

#### data.latest@v1

查询一个或多个测点的最新值。

请求：

```json
{ "device_id": "dev_01", "attribute_key": "temperature", "window": "5m" }
```

响应 `result.data` 为数组：

```json
[
  {
    "endpoint_id": "ep_01",
    "device_id": "dev_01",
    "attribute_key": "temperature",
    "value": 36.5,
    "unit": "℃",
    "quality": "good",
    "measured_at": "2026-11-13T10:00:00Z"
  }
]
```

nexus 按 `device_id` + `attribute_key` 过滤后取第一条，`value` 支持数字或数字字符串。

#### action.execute@v1

下发设备动作。平台内部应复用与 HTTP 执行接口相同的裁决逻辑。

请求 `params`：

```json
{
  "device_id": "dev_01",
  "action_code": "reboot",
  "params": { "delay_s": 3 },
  "reason": "告警处置：设备离线",
  "task_id": "task_01J...",
  "trigger_id": "task_01J...",
  "autonomy": "manual",
  "plan_id": "plan_01J...",
  "plan_version": 3,
  "step_id": "step_1",
  "max_risk": "L1"
}
```

请求 `meta` 必须包含 `idempotency_key`。响应 `result.data`：

```json
{
  "command_id": "cmd_01J...",
  "status": "pending_approval",
  "requires_approval": true,
  "required_permission": "action:approve:reboot"
}
```

`status=pending_approval` 或 `requires_approval=true` 时，nexus 通过 `command.get@v1` 轮询审批终态。

#### command.get@v1

请求：

```json
{ "command_id": "cmd_01J..." }
```

响应 `result.data`：

```json
{ "command_id": "cmd_01J...", "status": "succeeded", "required_permission": "" }
```

#### handling.report@v1

回报告警处置进度或终态。请求 `params` 即处置回报结构：

```json
{
  "schema_version": "v1",
  "report_id": "rpt_...",
  "platform_id": "anythings",
  "event_id": "01J...",
  "space_id": "default",
  "alarm_id": "al_01J...",
  "task_id": "task_01J...",
  "handler_user_id": "u_01",
  "plan_id": "plan_01J...",
  "plan_version": 3,
  "status": "SUCCEEDED",
  "summary": "重启后设备恢复在线",
  "steps": [
    {
      "id": "step_1",
      "type": "action",
      "status": "ok",
      "command_id": "cmd_01J...",
      "action_code": "reboot",
      "detail": { "latency_ms": 1200 }
    }
  ],
  "commands": [{ "command_id": "cmd_01J...", "status": "succeeded" }],
  "error": "",
  "reason": "",
  "occurred_at": "2026-11-13T10:05:00Z"
}
```

- `report_id` 未设置时由 nexus 生成稳定 ID，并作为幂等键。
- 平台应按 `report_id` 去重，重复回报不得产生重复副作用。

### 事件通道

#### 事件名

| event | 说明 |
| --- | --- |
| `alarm.triggered@v1` | 告警创建或再次活跃 |
| `alarm.recovered@v1` | 告警恢复，用于取消未完成处置 |

#### 事件载荷

`data` 为告警事件信封：

```json
{
  "schema_version": "v1",
  "platform_id": "anythings",
  "event_id": "01J...",
  "event_type": "alarm.triggered",
  "event_seq": 42,
  "space_id": "default",
  "alarm": {
    "alarm_id": "al_01J...",
    "dedup_key": "dev_01|temperature|high",
    "rule_id": "rule_01",
    "level": "critical",
    "code": "TEMP_HIGH",
    "message": "温度超过阈值",
    "status": "active",
    "device_id": "dev_01",
    "device_name": "1# 风机",
    "device_type_id": "dt_fan",
    "endpoint_id": "ep_01",
    "metric": "temperature",
    "value": 88.5,
    "threshold": 80,
    "org_id": "org_01",
    "triggered_at": "2026-11-13T10:00:00Z"
  },
  "handler": {
    "user_id": "u_01",
    "username": "zhangsan",
    "display_name": "张三"
  },
  "plan": { "schema_version": "v1", "plan_id": "plan_01J...", "version": 3, "hash": "sha256:..." },
  "plan_missing": false,
  "trace_id": "trace_01J..."
}
```

必填校验：`schema_version=v1`、`event_id`、`space_id`、`alarm.alarm_id`、`event_type` 合法、`event_seq>0`。

#### 顺序、去重与续传

- `seq` 在 `(platform, space)` 流内单调递增；nexus 以 `(platform, event_id)` 去重，以 `(platform, alarm_id, event_seq)` 判定顺序。
- nexus 成功处理后回 `ack` 帧（`channel=events`, `seq` 与事件一致），并推进 `last_seq`。
- 处理失败或连接中断时不回 ack，事件可在重连后重放，允许重复。
- 客户端通过 `hello.params.last_seq` 请求续传；服务端在保留窗口内重放未确认事件（建议默认 1000 条 / 5 分钟）。
- ACK 窗口建议 100，超窗应暂停推送而不是丢弃。

### 心跳与超时

- nexus 每隔 `heartbeat_s`（默认 15s）发送 `ping@v1`：

```json
{ "v": 1, "type": "request", "id": "nx-...", "channel": "control", "method": "ping@v1", "params": {} }
```

- 平台应回 `response`；若超过 `DeadAfter`（默认 60s）未收到任何帧，nexus 判定连接失效并主动重连。
- 方法调用默认超时 5s，可用 `meta.deadline_ms` 判断剩余时间。

### 错误码

| code | 说明 | 是否可重试 |
| --- | --- | --- |
| `AUTH_REQUIRED` | 连接未鉴权 | 否 |
| `AUTH_INVALID` | 令牌无效/过期 | 否 |
| `PERMISSION_DENIED` | 缺少权限，`details.required_permission` 给出所需权限 | 否 |
| `CAPABILITY_UNSUPPORTED` | 平台未声明能力或方法未实现 | 否 |
| `VALIDATION_FAILED` | 参数校验失败 | 否 |
| `VALIDATION_FRAME_TOO_LARGE` | 单帧超过 `max_frame` | 否 |
| `METHOD_NOT_FOUND` | 方法不在白名单 | 否 |
| `CONFLICT` | 幂等键/资源状态冲突 | 视情况 |
| `TIMEOUT` | 超过 `deadline_ms` | 是 |
| `RATE_LIMITED` | 触发限流 | 是（退避） |
| `INTERNAL` | 平台内部错误 | 是（有限次） |

服务端关闭码：

| 关闭码 | 含义 |
| --- | --- |
| `4401` | 鉴权失败 |
| `4403` | 空间/租户不匹配或无权限 |
| `4408` | 握手超时 |
| `1012` | 服务端 draining |
| `1013` | 过载 |

### 重连与续传

- 断线后按 1s / 2s / 4s / 8s / 16s 指数退避，单次上限 30s，并叠加 0.8–1.2 抖动。
- 若 `hello` 返回 `reconnect_after_ms`，优先使用该值（上限 30s）。
- 重连时携带上次成功处理的 `last_seq`；未 ACK 事件可能重复，由 `event_id` 去重。
- 同一进程内 `last_seq` 只增不减；平台可按 `event_seq` 识别并丢弃过期事件。

### 完整交互示例

```text
NX                                           平台
│  ── WSS + subprotocol nx.iot.v1 + Bearer ──▶
│  ── hello@v1 {platform,tenant,space,last_seq} ──▶
│  ◀── response {session_id,max_frame,heartbeat_s} ──
│  ── ping@v1 ────────────────────────────────▶
│  ◀── response {} ─────────────────────────────
│  ◀── event alarm.triggered@v1 seq=42 event_id=01J... ──
│  ── ack {channel:events, seq:42} ─────────────▶
│  ── device.get@v1 {device_id} ───────────────▶
│  ◀── response {data:{device_id,status}} ───────
│  ── action.execute@v1 {...,meta.idempotency_key} ──▶
│  ◀── response {data:{command_id,status:pending_approval}} ──
│  ── command.get@v1 {command_id} ──────────────▶
│  ◀── response {data:{command_id,status:succeeded}} ──
│  ── handling.report@v1 {report_id,...} ──────▶
│  ◀── response {ok:true} ───────────────────────
```

## HTTP 兜底与 REST 平台

### 告警触发（anythings HTTP）

`POST /internal/v1/nx/triggers/alarm`，`Authorization: Bearer <cc_service_token>`，请求体为「事件载荷」中的告警信封。仅在 `nx_agent.trigger.source=http` 时注册。

### 通用 REST 平台

`POST /internal/v1/platforms/{platform}/events`，`{platform}` 为 `platform_id`。

鉴权二选一：

- 未配置 `secret`：`Authorization: Bearer <platform.token 或 cc_service_token>`。
- 配置 `secret`：`X-Timestamp`（Unix 秒或 RFC3339）+ `X-Signature`，允许 ±5 分钟偏差；签名 = `hex(HMAC-SHA256(secret, X-Timestamp + "." + 原始请求体))`，可带 `sha256=` 前缀。

事件体兼容平铺字段与 `alarm` 嵌套字段，`event_id`/`id`、`event_type`/`type`、`event_seq`/`seq`、`space_id`/`space` 互为别名。

出站能力由 `capabilities` 声明：

| 能力 | 出站接口 |
| --- | --- |
| `queries` | `POST {base_url}/internal/v1/nx/metrics/read`、`GET {base_url}/internal/v1/nx/commands/{id}?space_id=...` |
| `actions` | `POST {base_url}/internal/v1/nx/actions/execute` |
| `reports` | `POST {base_url}/internal/v1/nx/reports` |

未声明的能力调用返回 `CAPABILITY_UNSUPPORTED`。

## 观测与排障

nexus 暴露 `GET /metrics`（Prometheus 文本格式）。本机回环直接放行，远程需 `Authorization: Bearer <cc_service_token>`。

| 指标 | 说明 |
| --- | --- |
| `nexus_ws_connections` | 当前 WS 连接数 |
| `nexus_ws_channel_inflight{channel=...}` | 各通道 in-flight |
| `nexus_ws_events_queue_depth` | 事件队列深度 |
| `nexus_command_duration_seconds` | commands 通道调用时延 |
| `nexus_rejections_total{reason=...}` | 拒绝/丢弃计数 |
| `nexus_ws_reconnects_total` | WS 重连次数 |

常见问题：

| 现象 | 排查方向 |
| --- | --- |
| 握手即断开 | 子协议是否为 `nx.iot.v1`；Bearer 令牌是否有效；租户/空间是否匹配 |
| 连接频繁重连 | 是否按 `heartbeat_s` 回包；`reconnect_after_ms` 是否过大；网络/代理是否中断长连接 |
| 事件重复处理 | 是否按 `event_id` 去重；ack 是否在业务处理成功后才发送 |
| 动作重复执行 | 平台是否按 `meta.idempotency_key` 去重 |
| 方法返回权限错误 | 检查 nx-agent 角色权限与 `details.required_permission` |
| 调用超时 | 平台是否按 `deadline_ms` 控制处理时间；大对象是否应走 HTTP |

## 接入验收清单

- [ ] WSS 入口可访问，子协议 `nx.iot.v1` 协商成功，Bearer 鉴权生效。
- [ ] `hello@v1` 返回 `session_id`，并按需返回 `max_frame` / `heartbeat_s` / `reconnect_after_ms`。
- [ ] `ping@v1` 正常回包，空闲连接不会被中间设备提前断开。
- [ ] 所需方法按「方法白名单」返回 `result.data`，错误按「错误码」返回 `error` 帧。
- [ ] `action.execute@v1` 与 `handling.report@v1` 按幂等键去重。
- [ ] `events` 通道 `seq` 单调递增，nexus ack 后不再重放；断线重连可按 `last_seq` 续传。
- [ ] 事件载荷通过校验（`event_id`、`space_id`、`alarm.alarm_id`、`event_seq`）。
- [ ] `/metrics` 中连接数、队列深度、重连次数符合预期。
