# dsh-file-attachment-local-fixed

DeepSeek Harness 文件附件插件的**本地修复分支**，基于 `@wszhoho/dsh-file-attachment` **0.6.2**。

原插件功能完整、体验好，但在 **DSH 0.2.1-rc.2** 上存在两处上游缺陷导致启动报错与启用失败。
本分支在不改变原有交互（拖入 / `Ctrl+V` 粘贴 / 📎 按钮、`@绝对路径` 引用、内联缩略图、
聊天区可点击放大、可配置 VLM 识图）的前提下修复了这些问题，并补了一层鲁棒性防护。

## 修复内容

### 1. `settings` 服务 API 变更导致启动报错

**症状**（每次启动一条红色 error）：

```
TypeError: scopedCtx.settings.register is not a function
    at .../lib/index.js (apply() 末尾 settings 注入处)
```

**根因**：DSH 0.2.1-rc.2 的 `settings` 服务改为 `SettingsForms` 实现
（`static inject = ['configEditor','profileContext']`），只暴露
`configure / describe / update / replace / mutate / write / schema / importLegacyDocument`，
**不再有 `register(ns, schema, options)`**。插件仍调用旧 API。

**修复**：能力探测后再登记；无 `register` 时静默跳过。
副作用是「设置 → 文件附件」卡片在本版 DSH 上不出现，其余功能不受影响。
配置文件此时位于 `~/.dsh/file-attachment.json`。

> 上游同类问题：`wszhoho/dsh-file-attachment` issue #1。

### 2. 路由不随 fiber 释放导致「启用失败」

**症状**（禁用后重新启用时）：

```
启用失败：1 entry did not activate dsh-file-attachment
Error: webserver: duplicate prefix route "/dsh-file-attachment"
```

**根因**：`dsh-host-webserver` 的 `register()` 契约写明
「**@returns the disposer removing the route**」，路由表以 `(kind, path)` 为键，
重复即抛。插件直接调用并**丢弃返回值**，旧 fiber 卸载时路由不释放，
重新启用时新 fiber 注册同一 path 必然冲突。

**修复**：改为 `ctx.effect(() => webServer.register({...}), 'label')`，
与同生态的 `dsh-plugin-image-tools`、`dsh-mineru`、`dsh-disk-doctor` 写法一致。

### 3. 鲁棒性加固（本分支新增）

原 `apply()` 无整体保护：`registerTools` / `registerRoutes` / 能力门控 / settings
任一环节抛错都会使整个条目 `did not activate`，连带客户端半也不注册。

本版为每个子系统加独立降级：

```js
safeCall('registerTools', () => registerTools(ctx))
safeCall('registerRoutes', () => registerRoutes(ctx))
```

任一子系统失败只 `console.warn` 并跳过该项，其余功能照常工作。

实测六种恶劣环境均不再整体失败：

| 场景 | 结果 |
| --- | --- |
| 全部服务可用 | PASS |
| `webServer` 缺失 | PASS |
| `tools` 缺失 | PASS |
| `settings` 缺失 | PASS |
| 全部服务缺失 | PASS |
| `settings` 无 `register` 方法 | PASS |

另外 `registerRoutes` 的守卫由「仅测 `undefined`」加强为同时拦截
`null` 与缺少 `register` 方法的情况。

## 安装

```sh
dsh plugin --profile <profile> add github:CDH2026/dsh-file-attachment-local-fixed
```

或本地路径：

```sh
dsh plugin --profile <profile> add <本目录绝对路径>
```

安装后需重启 DeepSeek Harness 桌面应用。

## package 改名说明

包名由 `@wszhoho/dsh-file-attachment` 改为 `dsh-file-attachment-local-fixed`，
原因：上游包名与 npm scope 归 `wszhoho` 所有，不能冒用发布。

改名连带变动的三处（已同步，勿单独修改）：

- `package.json` → `name`
- `cordis.patch.yml` → `insert[].name`（loader 按此名从 profile 的 `node_modules` 解析）
- `lib/client.js` → `window.__ModuleLoader__.load({ id })`

`lib/index.js` 导出的 `name = 'dsh-file-attachment'` 保持原值，它是插件实例标识而非包名。

## 兼容性

- DSH 内核：`>=0.2.0-rc.1 <0.3.0-0`（实测 0.2.1-rc.2）
- Node.js：`>=18`
- `@deepseek-ai/schemastery`：`^3.18.1`

## 许可

MIT，随上游。原项目：https://github.com/wszhoho/dsh-file-attachment
