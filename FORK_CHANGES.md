# Fork 变更说明 / What this fork changes

本仓库是 [Sliverkiss/cpa-plugin](https://github.com/Sliverkiss/cpa-plugin) 的 fork，
包含一个修复：**WorkBuddy 国际版（Global）账号无法被调度的问题**。

> 上游仓库已于 2026-08-30 归档（archived），无法提交 PR，因此修复发布在此 fork。

---

## 症状

国际版 WorkBuddy 账号在面板中能正常显示、积分也能查询，但**永远收不到任何请求**。
在面板上点选它会被静默忽略，流量始终落到国内（CN）账号上。

## 根因

`workbuddy/model_source_workbuddy.go` 中的 `workBuddyRealmFromAccessToken`
通过 JWT 的 issuer 域名判断账号属于 CN 还是 Global 区域。腾讯现在为国际版签发的
Token，其 issuer 是 `www.codebuddy.ai`，而白名单里没有这个域名：

```go
switch strings.ToLower(issuer.Hostname()) {
case "codebuddy.cn", "www.codebuddy.cn", "copilot.tencent.com":
    return workBuddyRealmCN, nil
case "workbuddy.ai":              // <- www.codebuddy.ai 匹配不到这里
    return workBuddyRealmGlobal, nil
default:
    return "", fmt.Errorf("JWT issuer host is unsupported")
}
```

## 为什么不容易发现

**聊天本身是正常的** —— Token 有效，`www.workbuddy.ai` 网关也接受它。
只有 realm 解析这条路会拒绝，所以单独测试账号看起来完全健康。故障是间接暴露的：

1. `workBuddyRealmFromAccessToken` 返回错误
2. `modelAuthIdentityFor` 随之失败（它第一步就调用 realm 解析）
3. 账号快照被存为 `modelFailed` / `modelErrorAuthInvalid`
4. `snapshotForAuthID(...).State.executable()` 为 false，于是
   `handleSchedulerPick` 直接跳过该账号

此外 `realm` 还参与模型缓存 key 的计算（`modelAuthIdentity.sha256()`），
所以区域判断错误也会连带影响缓存复用。

## 补丁

```go
case "workbuddy.ai", "www.codebuddy.ai":
    return workBuddyRealmGlobal, nil
```

并在 `TestWorkBuddyRealmFromAccessToken` 中补充该 issuer 的测试用例。
改动量：**5 行新增，1 行删除**，不影响 CN 与旧版 Global 账号的既有行为。

---

## 验证结果

环境：CPA v7.3.3 + 插件 0.9.3 + 本补丁，使用真实的国际版账号实测。

| 检查项 | 修复前 | 修复后 |
|---|---|---|
| `model_status.state` | `failed`（模型目录不可用） | `ready` |
| 国际版账号状态 | `failed` / `auth_invalid` | `ready` |
| 路由去向（`usage_events.auth_index`） | 100% 落到 CN 账号 | 点选后 8/8 请求命中 Global 账号 |
| 面板 `active_auth` | 回落到 CN 账号 | 保持为 Global 账号 |

确认 Token 本身没问题（绕过插件直连网关）：

```
POST https://www.workbuddy.ai/v2/chat/completions  ->  200 (SSE)
```

国际版网关有两个要求值得注意：`stream` 必须为 `true`，且首条消息必须是
`system` 消息（否则返回 `code 11101` / `code 11128`）。

---

## 编译方式

本插件是 cgo 编译的 `c-shared` 动态库，需要 C 编译器：

```bash
cd workbuddy
CGO_ENABLED=1 go build -buildmode=c-shared -o workbuddy.dll .
```

Windows + MinGW-w64：

```bash
export CC='D:\path	o\mingw64in\gcc.exe'
export CXX='D:\path	o\mingw64in\g++.exe'
CGO_ENABLED=1 go build -buildmode=c-shared -o workbuddy.dll .
```

> 注意：DLL 必须在 CPA 服务停止时替换，否则文件被占用；替换后重启服务生效。

---

## 其他说明

- 插件全部代码版权归 [Sliverkiss](https://github.com/Sliverkiss) 所有。
- 本 fork 仅新增上述 issuer 修复，未做其他改动。
- 若上游恢复维护，建议优先采用上游版本。
