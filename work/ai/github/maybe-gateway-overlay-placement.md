# maybe-gateway-overlay 放置建议

这份说明用于记录 `maybe-gateway-overlay` 在当前项目里的推荐放置位置，以及为什么这么放。

## 推荐放置位置

如果 `maybe-gateway-overlay` 的职责是这次讨论里的“多租户网关覆盖层”，建议放在 OpenClaw 的扩展目录里：

```text
openclaw/extensions/maybe-gateway-overlay/
  openclaw.plugin.json
  tsconfig.json
  src/
    plugin.ts
    http.ts
    tenant-router.ts
    auth-broker.ts
    config.ts
    types.ts
```

这样放的原因：

- 它更像控制面扩展，不只是单纯的 agent runtime 代码。
- OpenClaw 已经有成熟的 `extensions/*` 扩展模式，适合接 HTTP 路由、工具、channel、gateway 相关能力。
- 这样可以先避免直接改 `openclaw/src/gateway` 核心代码，只有当 extension API 明确不够用时，才往 core 里下沉。

可参考的源码模式：

- `openclaw/extensions/diffs/src/plugin.ts`
- `openclaw/extensions/qa-channel/src/gateway.ts`

## 不建议一开始放进去的位置

不建议第一步就直接放到下面这些 core 目录：

```text
openclaw/src/gateway/
openclaw/src/agents/
openclaw/src/config/
```

原因：

- 这些目录属于 OpenClaw 核心实现层，改动成本和后续维护成本都更高。
- 你的 `overlay` 现在更像外部集成层，不应该过早和 core 强耦合。

只有在下面这种情况下，才考虑往 core 挪：

- extension 层无法注册你需要的路由
- extension 层无法拿到你需要的 session / agent / auth 上下文
- extension 层无法满足性能或生命周期要求

## Hermes 专用场景

如果 `maybe-gateway-overlay` 实际上不是“多租户控制层”，而是一个 Hermes 专用的消息平台适配器，那么应该走 Hermes 的平台插件目录：

```text
hermes-agent/plugins/platforms/maybe_gateway_overlay/
  plugin.yaml
  adapter.py
```

这样放的原因：

- Hermes 官方推荐第三方或社区平台走 plugin path。
- 这个路径适合“平台适配器”，不适合承载更广义的多租户控制层。

可参考的源码模式：

- `hermes-agent/plugins/platforms/google_chat/adapter.py`
- `hermes-agent/gateway/platforms/ADDING_A_PLATFORM.md`

## 当前项目的结论

基于这次会话里已经确认的目标，当前更合适的落点是：

```text
openclaw/extensions/maybe-gateway-overlay/
```

建议先从最小骨架开始：

```text
openclaw/extensions/maybe-gateway-overlay/src/plugin.ts
openclaw/extensions/maybe-gateway-overlay/src/http.ts
openclaw/extensions/maybe-gateway-overlay/src/tenant-router.ts
```

其中建议职责划分如下：

- `plugin.ts`：注册 OpenClaw 插件入口、HTTP 路由、可能的工具入口
- `http.ts`：处理外部请求，比如 chat API、session API、proxy API
- `tenant-router.ts`：负责 `user_id -> agent_id -> auth_profile_id -> sessionKey` 映射
- `auth-broker.ts`：负责服务端密钥读取和凭证注入
- `config.ts`：统一读取和解析 overlay 自己的配置
- `types.ts`：定义 tenant、agent mapping、request context 等类型
