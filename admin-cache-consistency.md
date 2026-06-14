# 管理员缓存最终一致性分析

## 一、重判："RemoveUserAllTokens 不清五号是漏洞" —— 不是漏洞，是最终一致性设计

### 1.1 之前的错误定性

> ❌ 之前结论：`RemoveUserAllTokens` 不清理 ⑤ `answer:admin:token:*` 是**漏洞**

这个判断脱离了 fail-safe 语境，孤立地看单一代码路径，得出了"漏洞"的结论。放回完整体系中，这是一个**刻意为之的最终一致性设计**。

### 1.2 管理员缓存 ⑤ 的三重防护体系

管理员缓存（⑤ `answer:admin:token:<AccessToken>`）的清理不是依赖单一入口，而是由三层机制共同保证：

```
┌─────────────────────────────────────────────────────────────┐
│  第一层：主动清理（显式调用 RemoveAdminUserCacheInfo）         │
│                                                              │
│  ├─ Logout 三清 → 立即清 ⑤ 当前 Token                        │
│  └─ AdminAuth 三处异常分支 → 立即清 ⑤ 当前 Token              │
│     ├─ EmailStatus == ToBeVerified → RemoveAdminUserCacheInfo│
│     ├─ UserStatus == Suspended → RemoveAdminUserCacheInfo    │
│     └─ UserStatus == Deleted → RemoveAdminUserCacheInfo      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  第二层：Fail-Safe 联动（GetAdminUserCacheInfo 内部）          │
│                                                              │
│  每次管理员请求都走 GetAdminUserCacheInfo                      │
│    └─ 联动调用 GetUserCacheInfo(accessToken)                  │
│       ├─ ① 已存在 → 合并最新状态回写 ⑤（状态一致）             │
│       └─ ① 不存在 → RemoveAdminUserCacheInfo(当前Token)       │
│                      + return nil（假装 ⑤ 也不存在）           │
│                                                              │
│  ⭐ 关键：RemoveUserAllTokens 删了 ① 之后，下次管理员请求     │
│     GetAdminUserCacheInfo 联动 GetUserCacheInfo 发现 ① 缺失  │
│     → 主动删 ⑤ → return nil → AdminAuth 返回 403             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  第三层：TTL 兜底（Redis 7 天自动过期）                        │
│                                                              │
│  即使第一层和第二层都未触发，⑤ 号缓存最多存活 7 天             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 场景推演：RemoveUserAllTokens 后管理员的最终失效

**场景：管理员 B 被改角色（从管理员降为普通用户）**

```
Admin 操作: UpdateUserRole(B, RoleID=1)
    │
    ├─ SaveUserRole → 数据库更新 role_id=1
    │
    └─ RemoveUserAllTokens(B)
        ├─ 遍历 ③ 删除 B 的所有 ① answer:user:token:*
        ├─ 删除 ③ answer:user-token:mapping:B
        ├─ 删除 ④ answer:user:status:B
        └─ ❌ 不删 ⑤ answer:admin:token:*（之前的"漏洞"点）
    │
    ▼
管理员 B 在设备 1 尝试访问后台 API
    │
    ├─ AdminAuth() → GetAdminUserCacheInfo(B的Token)
    │   │
    │   ├─ 读 ⑤ → 仍存在！（还没清）
    │   │
    │   ├─ ⭐ Fail-Safe 触发：联动 GetUserCacheInfo(B的Token)
    │   │   │
    │   │   ├─ 读 ① answer:user:token:<B的Token>
    │   │   └─ 不存在！已被 RemoveUserAllTokens 删除
    │   │       │
    │   │       └─ return nil
    │   │
    │   ├─ refreshedUserCacheInfo == nil
    │   │   └─ ⭐ 主动清理：RemoveAdminUserCacheInfo(B的Token)
    │   │
    │   └─ return nil, nil
    │
    └─ AdminAuth() → userInfo == nil → 403 Forbidden
         （B 的管理员权限已失效，即使 ⑤ 未被主动删除）
```

**结论**：管理员 B 的 ⑤ 号缓存虽然没被 `RemoveUserAllTokens` 直接删除，但在 B 下次发起管理员请求时，`GetAdminUserCacheInfo` 的 Fail-Safe 会发现 ① 已失效，主动删除 ⑤ 并返回 nil。管理员权限在**下一次请求时**失效，满足最终一致性。

### 1.4 最终一致性 vs 立即一致性的权衡

| 策略 | 优点 | 缺点 |
|-----|------|------|
| **立即一致性**（RemoveUserAllTokens 也清 ⑤） | 管理员权限瞬间失效 | 需要遍历 ③ 找所有 Token，然后逐个删 ⑤；或者额外维护 UserID → AdminToken 映射 |
| **最终一致性**（当前设计） | 实现简单，不需要额外映射 | 管理员权限在下次请求时才失效，存在短暂窗口 |

当前设计选择了**最终一致性**，代价是有一个短暂的窗口期（从 RemoveUserAllTokens 到该管理员下次请求之间）。这个窗口期内，管理员 ⑤ 仍可被读到，但由于 `GetAdminUserCacheInfo` 的 Fail-Safe 会同步状态（RoleID 会被刷新），即使 ⑤ 暂时存在，**状态也是最新的**。

### 1.5 重新定性

> ✅ 修正结论：`RemoveUserAllTokens` 不清理 ⑤ **不是漏洞**，而是最终一致性设计。三重防护体系（主动清理 + Fail-Safe 联动 + TTL 兜底）共同保证管理员缓存在下一次请求时必定失效。设计上优先了实现简洁性，而非跨设备的立即一致性。

---

## 二、修正：UpdateUserRole 中删除四号缓存不是"无意义"

### 2.1 之前的错误判断

> ❌ 之前结论：UpdateUserRole 调用 RemoveUserAllTokens 时，删除 ④ 是无意义的空操作，因为 UpdateUserRole 之前从未写入 ④

这个判断忽略了 ④ 号缓存可能在 UpdateUserRole 之前**已被其他操作写入**。

### 2.2 四号缓存的写入时机

④ 号缓存 `answer:user:status:<UserID>` 在以下场景会被写入：

| 写入场景 | 代码位置 | 写入内容 | 何时发生 |
|---------|---------|---------|---------|
| **1. 用户登录** | [user_service.go L631](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L631) | `{UserID, Status, EmailStatus, RoleID}` | 每次邮箱密码登录 |
| **2. 邮箱验证** | [user_service.go L766](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L766) | `{UserID, EmailStatus=1, Status, RoleID}` | 用户验证邮箱后 |
| **3. 后台改状态** | [user_backyard_repo.go L80](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go#L80) | `{UserID, Status, EmailStatus}` | 管理员修改用户状态 |

### 2.3 真实场景：UpdateUserRole 时四号缓存可能已存在

**场景：管理员先修改用户状态，再修改角色**

```
管理员操作序列：
    │
    ├─ Step 1: PUT /admin/user/status → UpdateUserStatus
    │   └─ user_backyard_repo.UpdateUserStatus
    │       ├─ 数据库 UPDATE status=suspended
    │       └─ ⭐ SetUserStatus → ④ = {Status:9, EmailStatus:1}
    │           （④ 号缓存被写入！TTL=7天）
    │
    └─ Step 2: PUT /admin/user/role → UpdateUserRole
        ├─ SaveUserRole → 数据库 UPDATE role_id=3
        └─ RemoveUserAllTokens(userID)
            ├─ 遍历删 ①
            ├─ ⭐ 删 ④ answer:user:status:<userID>
            │   （④ 存在！删除是有意义的！）
            ├─ 删 ③
            └─ 不删 ② ⑤
```

**场景：用户自己登录后，管理员修改角色**

```
用户操作 + 管理员操作序列：
    │
    ├─ Step 1: 用户登录 → UserLoginWithEmail
    │   └─ SetUserStatus → ④ = {Status:1, EmailStatus:1, RoleID:1}
    │       （登录时写入了 ④！）
    │
    ├─ ... 用户正常使用 ...（④ 号缓存一直存在，TTL 7 天）
    │
    └─ Step 2: 管理员修改角色 → UpdateUserRole
        └─ RemoveUserAllTokens(userID)
            ├─ 遍历删 ①
            ├─ ⭐ 删 ④ answer:user:status:<userID>
            │   （④ 存在！是登录时写入的！）
            ├─ 删 ③
            └─ 不删 ② ⑤
```

### 2.4 如果不删四号会怎样

假设 `RemoveUserAllTokens` 不删除 ④，考虑以下场景：

```
管理员修改角色：User A 从 RoleID=1 → RoleID=2
    │
    ├─ RemoveUserAllTokens → 删 ① ③（假设不删 ④）
    │
    └─ User A 重新登录
        ├─ SetUserCacheInfo → 写入新 ① {RoleID=2}
        ├─ AddUserTokenMapping → 写入新 ③
        └─ SetUserStatus → 写入 ④ {RoleID=2}（覆盖旧值）
```

这个场景下不删 ④ 也不会出错，因为重新登录会覆盖 ④。但如果不删 ④，在 7 天 TTL 内，④ 中仍保留旧状态 `{RoleID=1}`。如果此时管理员又修改了角色（RoleID=2→3），但用户还没重新登录：

```
不删 ④ 时：
    User A Token T2（新登录后）→ ① {RoleID=2}
    ④ 仍保留旧值 {RoleID=1}（登录时写入，未被清理）
    
    GetUserCacheInfo(T2)
        ├─ ① → {RoleID=2}
        ├─ ④ → {RoleID=1}  ← ⚠️ 旧值！
        ├─ 合并 → RoleID=1  ← ⚠️ 错误！新登录的 Token 被旧状态覆盖
```

**这就是为什么要删 ④**：防止过时的 ④ 号缓存"污染"新登录 Token 的状态合并。删除 ④ 保证了角色变更后，用户的下次登录会从数据库读取最新状态，而不是被旧缓存覆盖。

### 2.5 修正结论

> ✅ 修正结论：UpdateUserRole 中 `RemoveUserAllTokens` 删除 ④ **是有意义的**。用户登录和邮箱验证场景都会写入 ④，且 ④ 的 TTL 为 7 天。如果不删 ④，过时的状态缓存会在用户重新登录后污染新 Token 的状态合并，导致角色变更被回退。

---

## 三、AdminAuth 三处清理仅作用于当前 Token 的跨设备窗口风险

### 3.1 三处清理的代码

[auth.go L194-L214](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L194-L214)：

```go
if userInfo != nil {
    if userInfo.EmailStatus == entity.EmailStatusToBeVerified {
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)  // 只删当前 Token
        // 403 Forbidden
    }
    if userInfo.UserStatus == entity.UserStatusSuspended {
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)  // 只删当前 Token
        // 403 Forbidden
    }
    if userInfo.UserStatus == entity.UserStatusDeleted {
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)  // 只删当前 Token
        // 401 Unauthorized
    }
}
```

### 3.2 "仅当前 Token"的含义

`RemoveAdminUserCacheInfo(ctx, token)` 删除的是：

```
⑤ answer:admin:token:<当前Token> → 删除这一个 Key
```

而**不是**删除该用户所有设备的管理员缓存。要删除其他设备的 ⑤，需要知道其他设备的 AccessToken，但这些 Token 只存在于 ③ 号映射中，而 AdminAuth 中间件没有遍历 ③ 的逻辑。

### 3.3 跨设备窗口风险推演

**场景：管理员 A 同时在线两台设备，A 被停用**

```
初始状态：
    ① answer:user:token:T1 = {Status:1, RoleID:2, ...}  ← 设备1
    ① answer:user:token:T2 = {Status:1, RoleID:2, ...}  ← 设备2
    ⑤ answer:admin:token:T1 = {Status:1, RoleID:2, ...} ← 设备1 管理员
    ⑤ answer:admin:token:T2 = {Status:1, RoleID:2, ...} ← 设备2 管理员
    ④ answer:user:status:A = {Status:9, EmailStatus:1}   ← 新写入

    │
    ▼ 设备1 发起管理员请求（先于设备2）
    
AdminAuth() → GetAdminUserCacheInfo(T1)
    ├─ 读 ⑤ T1 → {Status:1, ...}
    ├─ 联动 GetUserCacheInfo(T1)
    │   ├─ 读 ① T1 → {Status:1, ...}
    │   ├─ 读 ④ A → {Status:9, ...}
    │   ├─ 合并 → Status=9
    │   └─ 回写 ① T1 = {Status:9, ...}
    ├─ 合并到 ⑤ T1 → {Status:9, ...}
    └─ return {Status:9, ...}

AdminAuth 状态检查：
    ├─ UserStatus == Suspended(9) → 命中！
    ├─ ⭐ RemoveAdminUserCacheInfo(T1) → 删除 ⑤ T1
    └─ 403 Forbidden

    │
    ▼ 此时设备2 的状态：
    
    ① T2 = {Status:1, ...}  ← 还是旧值！尚未被刷新
    ⑤ T2 = {Status:1, ...}  ← 还存在！未被清理
    ④ A = {Status:9, ...}   ← 仍存在

    │
    ▼ 设备2 发起管理员请求

AdminAuth() → GetAdminUserCacheInfo(T2)
    ├─ 读 ⑤ T2 → {Status:1, ...}（仍在，旧值）
    ├─ 联动 GetUserCacheInfo(T2)
    │   ├─ 读 ① T2 → {Status:1, ...}（仍在，旧值）
    │   ├─ 读 ④ A → {Status:9, ...}
    │   ├─ 合并 → Status=9
    │   └─ 回写 ① T2 = {Status:9, ...}
    ├─ 合并到 ⑤ T2 → {Status:9, ...}
    └─ return {Status:9, ...}

AdminAuth 状态检查：
    ├─ UserStatus == Suspended(9) → 命中！
    ├─ ⭐ RemoveAdminUserCacheInfo(T2) → 删除 ⑤ T2
    └─ 403 Forbidden
```

### 3.4 窗口期量化

| 事件 | 设备1 管理员权限 | 设备2 管理员权限 |
|-----|----------------|----------------|
| A 被停用（SetUserStatus 写 ④） | ✅ 仍有（直到下次请求） | ✅ 仍有（直到下次请求） |
| 设备1 下次管理员请求 | ❌ 403（Fail-Safe + 状态检查清 ⑤ T1） | ✅ 仍有（⑤ T2 未清） |
| 设备2 下次管理员请求 | ❌ 已失效 | ❌ 403（Fail-Safe + 状态检查清 ⑤ T2） |

**窗口期** = 从 ④ 写入到设备 N 下次发起管理员请求的时间间隔。

- 如果设备2 长时间无操作 → ⑤ T2 一直存在但不会被使用
- 如果设备2 持续有请求 → 下一次请求就会触发 Fail-Safe，最多延迟一个请求周期
- 如果设备2 在窗口期内发起管理员请求 → **能通过**（⑤ T2 还是旧值，但 GetUserCacheInfo 会合并 ④ 的新状态，所以实际拿到的 Status=9 → 被 AdminAuth 拦截）

### 3.5 关键澄清：窗口期内管理员权限是否真正"泄露"？

重新审视上面的推演：

设备2 在窗口期内发起管理员请求时，虽然 ⑤ T2 还存在，但 `GetAdminUserCacheInfo` 内部会：
1. 联动 `GetUserCacheInfo(T2)` → 读 ① T2（旧值）→ 读 ④ A（新值）→ 合并 Status=9 → 回写 ① T2
2. 合并到 ⑤ T2 → Status=9
3. 返回 `{Status:9, ...}`
4. AdminAuth 检查 Status=9 → RemoveAdminUserCacheInfo(T2) → 403

**所以窗口期内管理员权限并没有真正泄露！** 因为 Fail-Safe 会在每次请求时同步最新状态。即使 ⑤ 暂时存在，状态已经被 ④ 覆盖为 Suspended，AdminAuth 一定会拦截。

### 3.6 真正的窗口风险：UpdateUserRole 场景

`UpdateUserRole` 场景更危险，因为它调用了 `RemoveUserAllTokens`，会删除所有 ①。此时：

```
UpdateUserRole(A, RoleID=1) → RemoveUserAllTokens(A)
    ├─ 删所有 ① T1, T2
    ├─ 删 ③
    ├─ 删 ④
    └─ 不删 ⑤ T1, T2

    │
    ▼ 设备2 发起管理员请求

AdminAuth() → GetAdminUserCacheInfo(T2)
    ├─ 读 ⑤ T2 → {Status:1, RoleID:2, ...}（仍存在，旧值）
    ├─ 联动 GetUserCacheInfo(T2)
    │   ├─ 读 ① T2 → 不存在！（已被 RemoveUserAllTokens 删除）
    │   └─ return nil
    ├─ refreshedUserCacheInfo == nil
    ├─ ⭐ Fail-Safe：RemoveAdminUserCacheInfo(T2) → 删除 ⑤ T2
    └─ return nil

AdminAuth() → userInfo == nil → 403 Forbidden
```

**结论**：即使在 UpdateUserRole 场景下，Fail-Safe 也能在下一次请求时正确清理 ⑤。管理员权限不会泄露。

### 3.7 唯一的真实窗口：GetAdminUserCacheInfo 自身的 ⑤ 读取 → 状态合并 → 回写 竞态

唯一可能出现短暂不一致的情况是：`GetAdminUserCacheInfo` 在合并状态时，如果 `GetUserCacheInfo` 内部的 ④ 号缓存恰好在读取和合并之间被删除（极端竞态），那么：

```
GetAdminUserCacheInfo(T2)
    ├─ 读 ⑤ T2 → {Status:1, RoleID:2}
    ├─ 联动 GetUserCacheInfo(T2)
    │   ├─ 读 ① T2 → {Status:1, RoleID:2}
    │   ├─ 读 ④ A → 不存在（恰被另一个请求清了）
    │   └─ return {Status:1, RoleID:2}（未合并新状态）
    ├─ 合并到 ⑤ T2 → {Status:1, RoleID:2}（旧状态）
    └─ return {Status:1, RoleID:2}

    → AdminAuth 放行！管理员权限未失效！
```

但这需要 ④ 恰好在 `GetUserCacheInfo` 读 ① 和读 ④ 之间被删除，窗口极小。且下次请求时 ① 可能已被删，Fail-Safe 会兜底。

---

## 四、完整一致性保证模型

### 4.1 各操作对五层缓存的影响

| 操作 | ① Token | ② Visit | ③ Mapping | ④ Status | ⑤ Admin | 管理员权限失效时机 |
|-----|---------|---------|-----------|----------|---------|----------------|
| **Logout** | ✅ 清当前 | ✅ 清当前 | - | - | ✅ 清当前 | 立即（当前设备） |
| **UpdateUserRole** | ✅ 清全部 | ❌ 残留 | ✅ 清 | ✅ 清 | ❌ 残留 | 下次请求（Fail-Safe 发现 ① 缺失 → 清 ⑤） |
| **UpdateUserPassword** | ✅ 清其他 | ❌ 残留 | ✅ 清 | ✅ 清 | ❌ 残留 | 下次请求（Fail-Safe） |
| **UpdateUserStatus** | - | - | - | ✅ 写入 | - | 下次请求（④ 合并 → 状态检查 → 清 ⑤） |
| **用户自删账号** | ✅ 清全部 | ❌ 残留 | ✅ 清 | ✅ 清 | ❌ 残留 | 下次请求（Fail-Safe） |
| **CLI 重置密码** | ✅ 清全部 | ❌ 残留 | ✅ 清 | ✅ 清 | ❌ 残留 | 下次请求（Fail-Safe） |
| **AdminAuth 异常分支** | - | - | - | - | ✅ 清当前 | 立即（当前设备） |

### 4.2 最终一致性保证链

```
管理员权限变更
    │
    ├─ 场景A：通过 RemoveUserAllTokens（改角色/改密码/删账号）
    │   │
    │   ├─ ① 被清 → 下次 GetUserCacheInfo 返回 nil
    │   ├─ GetAdminUserCacheInfo 联动发现 ① 缺失
    │   ├─ Fail-Safe 主动清 ⑤
    │   └─ AdminAuth → 403 ✅
    │
    ├─ 场景B：通过 SetUserStatus（改状态）
    │   │
    │   ├─ ④ 被写入新状态
    │   ├─ 下次 GetUserCacheInfo 合并 ④ → 回写 ①
    │   ├─ GetAdminUserCacheInfo 联动同步最新状态到 ⑤
    │   ├─ AdminAuth 状态检查 → RemoveAdminUserCacheInfo(当前Token)
    │   └─ 403 ✅
    │
    └─ 场景C：主动 Logout
        │
        ├─ 直接清 ⑤ 当前 Token
        └─ AdminAuth → 403 ✅
```

### 4.3 残留缓存的安全影响

| 残留缓存 | 安全风险 | 是否可利用 |
|---------|---------|----------|
| ② VisitToken 残留 | 私有模式下，被踢设备仍可通过 Cookie 访问静态图片 | ⚠️ 低风险：只能看图片，不能操作 API |
| ⑤ AdminToken 残留（非当前设备） | 看似高危 | ✅ 不可利用：下次请求 Fail-Safe 一定会清理或同步最新状态 |
| ④ Status 旧值残留 | 被新登录的 Token 合并时可能回退状态 | ❌ 不会残留：RemoveUserAllTokens 会删 ④，登录会覆盖 ④ |

---

## 五、关键文件索引

| 分析点 | 文件 | 关键行 |
|-------|------|--------|
| GetAdminUserCacheInfo Fail-Safe | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L147-L181 |
| GetUserCacheInfo 状态合并 | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L63-L91 |
| AdminAuth 三处异常清理 | [auth.go (middleware)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L194-L214 |
| RemoveUserAllTokens 实现 | [auth.go (repo)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go) | L211-L239 |
| SetUserStatus 写入（登录） | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go) | L631 |
| SetUserStatus 写入（邮箱验证） | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go) | L766 |
| SetUserStatus 写入（后台改状态） | [user_backyard_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go) | L73-L83 |
| UpdateUserRole 调用 RemoveUserAllTokens | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L231 |
| UpdateUserStatus 不调 RemoveUserAllTokens | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L128-L179 |
| Logout 三清 | [user_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go) | L240-L251 |
