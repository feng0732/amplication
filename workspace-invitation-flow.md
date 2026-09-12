# Amplication Workspace 成员邀请全链路代码说明

> 基于仓库当前 HEAD（分支 `task/1`，提交 `7656495`）逐行阅读整理。
> 每条结论均标注文件路径与行号；结论分为两类：
> **【代码事实】** = 代码明确保证的行为；**【推断】** = 基于代码语义/上下文的合理推论，需进一步复核。

涉及的两个主要包：

- 前端：`packages/amplication-client`
- 服务端：`packages/amplication-server`（NestJS + GraphQL + Prisma）

---

## 1. 完整时序：从提交邮箱到页面刷新

### 时序总览

```
邀请人 (已登录)                     服务端                          被邀请人
─────────────────────────────────────────────────────────────────────────
InviteMember.tsx 输入邮箱
  └─ mutation inviteUser ──────►  WorkspaceResolver.inviteUser
                                  └─ WorkspaceService.inviteUser
                                     ├─ 创建 Invitation 行(token=cuid)
                                     └─ MailService.sendInvitation
                                        └─ SendGrid 邮件
                                        链接=<CLIENT_HOST>/login?invitation=<token>
─────────────────────────────────────────────────────────────────────────
被邀请人点击链接 ────────────────►  浏览器打开 /login?invitation=<token>
                                    App.tsx 把 token 存入 localStorage
                                    ├─ 已有账号 → SignInForm 登录
                                    └─ 无账号   → Signup 注册
                                    登录/注册成功后进入 /:workspace 布局
                                    WorkspaceLayout 挂载 CompleteInvitation
                                      └─ mutation completeInvitation ─►
                                        AuthResolver.completeInvitation
                                        └─ AuthService.completeInvitation
                                           └─ WorkspaceService.completeInvitation
                                              ├─ 校验 token
                                              ├─ 在目标 workspace 创建 User
                                              └─ 标记 Invitation 已接受
                                           └─ AuthService.setCurrentWorkspace
                                              └─ 签发指向新 workspace 的 JWT
                                    ◄──── 返回 { token }
                                    setToken(新 JWT) → 清空邀请 token
                                    → history.replace("/") → window.location.reload()
```

### 1.1 邀请人提交邮箱（前端）

- 页面路由：`/:workspace/settings/members` 渲染 `MemberList`（`packages/amplication-client/src/routes/appRoutes.tsx:93-94`），其头部内嵌 `InviteMember` 组件（`packages/amplication-client/src/Workspaces/MemberList.tsx:59`）。
- `InviteMember`（`packages/amplication-client/src/Workspaces/InviteMember.tsx`）：
  - 前端先做权限预检：`permissions.canPerformTask("workspace.member.invite")`，无权限则输入框和按钮禁用（`InviteMember.tsx:52`、`InviteMember.tsx:100-108`）。
  - 提交时调用 `inviteUser({ variables: { email } })`（`InviteMember.tsx:65-73`），GraphQL 定义为 `mutation inviteUser($email: String!) { inviteUser(data: { email: $email }) { id email } }`（`InviteMember.tsx:122-129`）。
  - 成功后通过 `refetchQueries: [{ query: GET_WORKSPACE_MEMBERS }]` 刷新成员列表（`InviteMember.tsx:62`），并上报分析事件 `WorkspaceMemberInvite`（`InviteMember.tsx:55-60`）。

### 1.2 邀请记录创建（服务端）

- GraphQL 入口：`WorkspaceResolver.inviteUser`（`packages/amplication-server/src/core/workspace/workspace.resolver.ts:130-143`），委托给 `WorkspaceService.inviteUser`（`packages/amplication-server/src/core/workspace/workspace.service.ts:232-308`）。
- 服务方法依次执行（详见第 2 节）：
  1. 成员额度检查 `canInvite`（`workspace.service.ts:236-240`）；
  2. 邮箱非空检查（`workspace.service.ts:244-246`）；
  3. 重复成员检查（`workspace.service.ts:248-259`）；
  4. 重复邀请检查（`workspace.service.ts:261-275`）；
  5. `prisma.invitation.create` 写入邀请行：`token: cuid()`、`tokenExpiration: addDays(new Date(), 7)`（`workspace.service.ts:283-299`；常量 `INVITATION_EXPIRATION_DAYS = 7` 见 `workspace.service.ts:41`）。
- 数据模型：`Invitation` 表含 `email / invitedByUserId / workspaceId / newUserId(可空,唯一) / token / tokenExpiration`，并有 `@@unique([workspaceId, email])` 复合唯一约束（`packages/amplication-prisma-db/prisma/schema.prisma:722-737`）。

### 1.3 邀请邮件与链接生成（服务端）

- `WorkspaceService.inviteUser` 调用 `this.mailService.sendInvitation({ to, invitationToken, invitedByUserFullName })`（`workspace.service.ts:301-305`）。
- `MailService.sendInvitation`（`packages/amplication-server/src/core/mail/mail.service.ts:28-53`）：
  - 拼接链接：`const inviteUrl = \`${host}/login?invitation=${args.invitationToken}\``（`mail.service.ts:36`），`host` 取自环境变量 `CLIENT_HOST`（`mail.service.ts:16`、`mail.service.ts:34`）。
  - 通过 SendGrid 动态模板发送，模板变量为 `inviter_name` 与 `invitation_url`（`mail.service.ts:40-51`）。
  - 注意【代码事实】：模板变量 `inviter_name` 实际传入的是邀请人账号的 **email**（`workspace.service.ts:304` 传的是 `currentUserAccount.email`），字段名 `invitedByUserFullName` 名不副实。

### 1.4 被邀请人打开链接、token 保存（前端）

- 被邀请人访问 `https://<host>/login?invitation=<token>`。
- `App` 组件的全局 `useEffect` 解析 `location.search`，发现 `params.invitation` 即写入 localStorage：`setInvitationToken(params.invitation as string)`（`packages/amplication-client/src/App.tsx:123-131`）；key 为 `"invitationToken"`（`App.tsx:40`）。
- 代码注释说明了用 localStorage 的原因：GitHub OAuth（passport）不支持动态 callback，token 必须跨整页跳转保存（`App.tsx:126-128`）。
- `/login` 是公开路由（`packages/amplication-client/src/routes/appRoutes.tsx:543-549`）；若未登录用户直接访问受保护路由，会被重定向到 `/login` 且**保留 query string**（`packages/amplication-client/src/routes/routesUtil.tsx:69-84`，`search: location.search`），因此 `?invitation=` 参数不会在登录跳转中丢失。

### 1.5 注册或登录（前端 + 服务端）

- **登录**：`SignInForm` 提交 `mutation login`，成功后 `setToken(data.login.token)` 并 `history.replace(from + location.search)`，默认回到 `/`（`packages/amplication-client/src/User/SignInForm.tsx:42-54`）。服务端 `AuthResolver.login` → `AuthService.login` 校验密码后签发 JWT（`packages/amplication-server/src/core/auth/auth.resolver.ts:75-80`、`packages/amplication-server/src/core/auth/auth.service.ts:327-364`）。
- **注册**：`Signup` 提交 `mutation signup`，成功后 `setToken(data.signup.token)` 并跳转 `/`；若 localStorage 中存在邀请 token，则**不附加** `complete-signup=1`（`packages/amplication-client/src/User/Signup.tsx:60-71`）。服务端 `AuthService.signup` 会创建 Account、并调用 `bootstrapUser` 为该账号**创建一个属于他自己的新 workspace**（`auth.service.ts:290-325`）——即新用户先落在自己的 workspace 里，接受邀请后再切换。
- 无论哪种方式，之后 `useCurrentWorkspace` 在路径 `/` 下查询 `currentWorkspace` 并把用户重定向到 `/{workspaceId}`（`packages/amplication-client/src/Workspaces/hooks/useCurrentWorkspace.ts:27-44`）。

### 1.6 完成邀请（前端触发 + 服务端处理）

- `CompleteInvitation` 组件挂载在已认证的工作区布局内（`packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx:266`），其文件头注释也明确要求"必须位于登录后才能到达的位置"（`packages/amplication-client/src/User/CompleteInvitation.tsx:15-19`）。
- 触发逻辑（`CompleteInvitation.tsx`）：
  - `useEffect` 发现 localStorage 中 `invitationToken` 非空即发起 mutation（`CompleteInvitation.tsx:40-48`）：`mutation completeInvitation($token: String!) { completeInvitation(data: { token: $token }) { token } }`（`CompleteInvitation.tsx:55-61`）。
  - `onCompleted`：`setToken(data.completeInvitation.token)` 保存**新 JWT** → 清空 localStorage 邀请 token → `history.replace("/")` → `window.location.reload()`（`CompleteInvitation.tsx:28-33`）。
  - `onError`：仅 `console.error` 并清空邀请 token（`CompleteInvitation.tsx:34-37`）——**用户界面无任何提示**【代码事实】。
- GraphQL 入口：`AuthResolver.completeInvitation`（`packages/amplication-server/src/core/auth/auth.resolver.ts:142-153`）→ `AuthService.completeInvitation`（`auth.service.ts:657-667`）→ `WorkspaceService.completeInvitation`（`workspace.service.ts:310-397`），最后由 `AuthService.setCurrentWorkspace` 签发指向被邀请 workspace 的新 JWT（`auth.service.ts:366-388`）。

### 1.7 页面刷新（前端）

- `window.location.reload()`（`CompleteInvitation.tsx:32`）强制整页刷新：内存中的 Apollo 缓存、React 状态全部重建，之后的所有 GraphQL 请求都携带新 JWT（新 JWT 的 payload 含新 workspace 的 `workspaceId` 与新 User 的 `userId`，见 `auth.service.ts:517-527` 的 `prepareToken`）。
- 刷新后 `useCurrentWorkspace` 再次把用户从 `/` 引导到 `/{当前workspaceId}`，此时"当前 workspace"已是被邀请加入的 workspace【代码事实，由 JWT 内容与服务端 `currentWorkspace` 查询共同保证】。

---

## 2. `inviteUser` 与 `completeInvitation` 的认证、权限与数据库行为

### 2.1 公共认证层

- 两个 resolver 类都挂了 `@UseFilters(GqlResolverExceptionsFilter)`；`WorkspaceResolver` 类级别挂了 `@UseGuards(GqlAuthGuard)`（`workspace.resolver.ts:44-47`），`AuthResolver.completeInvitation` 方法级挂了 `@UseGuards(GqlAuthGuard)`（`auth.resolver.ts:142-143`）。
- `GqlAuthGuard extends AuthGuard("jwt")`（`packages/amplication-server/src/guards/gql-auth.guard.ts:20`）：先走 passport-jwt 验签，`JwtStrategy.validate` 再用 payload 中的 `userId` 查库得到 `AuthUser`（含 account、workspace、permissions）（`packages/amplication-server/src/core/auth/jwt.strategy.ts:22-43` → `user.service.ts:53-72`）。**因此两个 mutation 都要求调用者已登录（持有有效 JWT）**【代码事实】。
- JWT 验签后，`GqlAuthGuard.canActivate` 再读取 handler 上的 `@AuthorizeContext` 元数据并调用 `PermissionsService.validateAccess`（`gql-auth.guard.ts:31-45`、`gql-auth.guard.ts:61-83`）。

### 2.2 `inviteUser` 的权限控制

- 装饰器：`@AuthorizeContext(AuthorizableOriginParameter.None, "", "workspace.member.invite")`（`workspace.resolver.ts:133-137`）。
- `AuthorizableOriginParameter.None` 的校验函数直接返回 `{ canAccessWorkspace: true }`（`packages/amplication-server/src/core/permissions/validation-functions.ts:39-41`），即**不针对任何具体资源做归属校验**；真正生效的是权限串 `workspace.member.invite`。
- `PermissionsService.validatePermissions`：用户 permissions 含 `"*"`（workspace owner，见 `user.service.ts:82-84`）直接放行；否则需在团队角色或资源级授权中匹配到 `workspace.member.invite`（`packages/amplication-server/src/core/permissions/permissions.service.ts:49-87`）。
- 目标 workspace 不来自客户端入参，而是取自 JWT 上下文 `currentUser.workspace.id`（`workspace.service.ts:236`、`workspace.service.ts:242`）【代码事实：不能跨 workspace 邀请】。

### 2.3 `inviteUser` 的数据库读写（`workspace.service.ts:232-308`）

| 步骤 | 操作 | 位置 |
|---|---|---|
| 1 | `canInvite`：billing 未启用直接返回 `false`；否则查 Stigg 计量额度 `BillingFeature.TeamMembers`，`currentUsage < usageLimit` 才放行 | `workspace.service.ts:217-230`，调用处 `236-240` |
| 2 | 邮箱为空 → `ConflictException` | `workspace.service.ts:244-246` |
| 3 | 读：`userService.findUsers({ account: { email }, workspace: { id } })`（`findUsers` 自动附加 `deletedAt: null`，见 `user.service.ts:43-51`）；命中 → `ConflictException("...already exist in the workspace.")` | `workspace.service.ts:248-259` |
| 4 | 读：`prisma.invitation.findUnique`（复合键 `workspaceId_email`）；命中 → `ConflictException("Invitation ... already exist")` | `workspace.service.ts:261-275` |
| 5 | 读：`prisma.account.findUnique`（取邀请人账号，用于邮件署名） | `workspace.service.ts:277-281` |
| 6 | 写：`prisma.invitation.create`，`token: cuid()`、`tokenExpiration: 现在+7天` | `workspace.service.ts:283-299` |
| 7 | 发邮件（SendGrid） | `workspace.service.ts:301-305` → `mail.service.ts:28-53` |

额度不足时抛出 `BillingLimitationError`，由异常过滤器转成 `GraphQLBillingError` 返回客户端（`packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts:64-70`）。注意【代码事实】：此处错误携带的 billing feature 是 `BillingFeature.Projects`（`workspace.service.ts:239`），而实际检查的是 `BillingFeature.TeamMembers`（`workspace.service.ts:224`）——疑似复制粘贴错误，可能影响前端按 feature 展示的升级引导。

### 2.4 `completeInvitation` 的权限控制

- 只有 `GqlAuthGuard`（登录即可），**没有** `@AuthorizeContext`，即不校验任何 workspace 权限串（`auth.resolver.ts:142-153`）【代码事实】。这是合理的：被邀请人在接受邀请前还不是目标 workspace 的成员，不可能拥有该 workspace 的权限；token 本身就是授权凭证。
- 前置校验：`if (!user.account) throw new Error("User has no account")`（`auth.resolver.ts:148-150`）。

### 2.5 `completeInvitation` 的数据库读写（`workspace.service.ts:310-397`）

| 步骤 | 操作 | 位置 |
|---|---|---|
| 1 | 读：`prisma.invitation.findMany({ token, tokenExpiration: { gt: 现在-7天 }, newUser: null })`；结果必须恰好 1 条，否则 `ConflictException("Invitation cannot be found or it has expired")` | `workspace.service.ts:316-330` |
| 2 | 读：`userService.findUsers({ account: { id: 当前account }, workspace: { id: 邀请的workspaceId } })`；非空 → `ConflictException("The current account is already a member in this workspace")` | `workspace.service.ts:334-345` |
| 3 | 写：`prisma.workspace.update`，在目标 workspace 下 `users.create({ account: connect 当前account, isOwner: false })`，并 `include.users` 过滤出新 User | `workspace.service.ts:347-370` |
| 4 | 写：`prisma.invitation.update`：`tokenExpiration: 现在-7天`（使步骤 1 的查询不再命中，即"作废"token），并 `newUser: connect 新User.id` | `workspace.service.ts:372-384` |
| 5 | `billingService.reportUsage(workspace.id, BillingFeature.TeamMembers)` 上报用量 | `workspace.service.ts:386-389` |
| 6 | 上报分析事件 `InvitationAcceptance` | `workspace.service.ts:391-394` |

随后 `AuthService.completeInvitation` 调用 `setCurrentWorkspace(accountId, workspace.id)`（`auth.service.ts:657-667`）：查出新 User 的 `AuthUser`、把 `Account.currentUserId` 指向新 User（`account.service.ts:53-66`）、签发含新 `workspaceId` 的 JWT（`auth.service.ts:366-388`、`auth.service.ts:517-527`）返回给前端。

### 2.6 接受邀请后三方状态变化【代码事实】

1. **Account（账号）**：`Account.currentUserId` 被更新为被邀请 workspace 中的新 User id（`auth.service.ts:385` → `account.service.ts:53-66`）。账号本身不新建（被邀请人必须先注册/登录）；账号原先所在的 workspace（注册时自动创建的那个）保持存在，可通过 `setCurrentWorkspace` 切回。
2. **Workspace 用户（User 表）**：目标 workspace 新增一行 `User`，`isOwner: false`，关联当前 Account（`workspace.service.ts:347-368`）。新 User 无任何团队角色，故 `getUserPermissions` 返回空数组（`user.service.ts:74-105`）——在接受邀请的 workspace 里默认没有任何权限串，直到被分进某个 Team/Role【推断：基于 `getUserPermissions` 只聚合团队角色与 owner 特判】。
3. **Invitation（邀请记录）**：行不删除，而是 `tokenExpiration` 被改写成"现在-7天"（恰好落在完成查询的 `gt: 现在-7天` 窗口之外，token 即刻失效），同时 `newUserId` 指向新 User（`workspace.service.ts:372-384`）。此后该记录从成员列表的"待接受"中消失（`findMembers` 只查 `newUser: null`，`workspace.service.ts:533-538`），也无法再被 `revokeInvitation`/`resendInvitation` 命中（两者同样要求 `newUser: null`，`workspace.service.ts:400-406`、`workspace.service.ts:420-425`）。

---

## 3. 成员列表的构成与各分支结果

### 3.1 为什么成员列表同时包含 User 和 Invitation

- GraphQL 查询 `workspaceMembers`（`workspace.resolver.ts:188-197`）→ `WorkspaceService.findMembers`（`workspace.service.ts:526-555`）：
  - 查该 workspace 下所有未删除的 `User`（`workspace.service.ts:527-531`）；
  - 查该 workspace 下所有 `newUser: null` 的 `Invitation`（即尚未被接受的邀请，`workspace.service.ts:533-538`）；
  - 两者分别包装成 `{ type: User | Invitation, member }` 后拼接返回（`workspace.service.ts:540-554`）。`WorkspaceMemberType` 是 GraphQL union 类型（`packages/amplication-server/src/core/workspace/dto/WorkspaceMemberType.ts`）。
- 前端 `MemberList` 用内联 fragment 同时取两种类型（`packages/amplication-client/src/Workspaces/MemberList.tsx:104-127`）；`MemberListItem` 对 Invitation 类型显示邮箱文本和紫色 **"Pending"** 徽标，并提供"重发邮件"按钮；对 User 类型显示头像/姓名，owner 显示 "Owner" 徽标（`packages/amplication-client/src/Workspaces/MemberListItem.tsx:113-128`、`MemberListItem.tsx:150-162`、`MemberListItem.tsx:187-190`）。
- 设计意图【推断】：邀请发出到被接受之间存在时间差，把"待接受邀请"与"正式成员"并列展示，让管理员能看到邀请状态并执行重发/撤销；删除操作也按类型分流——User 走 `deleteUser`，Invitation 走 `revokeInvitation`（`MemberListItem.tsx:81-99`）。

### 3.2 各分支的实际结果

| 分支 | 结果 | 依据 |
|---|---|---|
| 邀请邮箱为空 | `ConflictException: email address is required to invite a user`（HTTP 异常原样透传给客户端） | `workspace.service.ts:244-246`；过滤器 `GqlResolverExceptions.filter.ts:75-78` |
| 重复成员（该邮箱的账号已在 workspace 有未删除 User） | `ConflictException: User with email ... already exist in the workspace.` | `workspace.service.ts:248-259` |
| 重复邀请（同 workspace 同邮箱的 Invitation 行已存在，无论是否已接受） | `ConflictException: Invitation with email ... already exist in the workspace.` | `workspace.service.ts:261-275`；唯一约束 `schema.prisma:736` |
| 并发重复邀请（两请求同时通过检查） | 后到的 `invitation.create` 触发 Prisma P2002 唯一键冲突，过滤器转为 `GraphQLUniqueKeyException` | `schema.prisma:736`；`GqlResolverExceptions.filter.ts:56-63` |
| 成员额度不足 | `BillingLimitationError("Your workspace exceeds its members limitation.")` → `GraphQLBillingError`（feature 字段误标为 `Projects`） | `workspace.service.ts:236-240`；`GqlResolverExceptions.filter.ts:64-70` |
| token 无效 / 已被使用 / 过"期" | `ConflictException: Invitation cannot be found or it has expired`（查询要求 `token` 匹配 + `tokenExpiration > 现在-7天` + `newUser: null`，且恰好 1 条） | `workspace.service.ts:316-330` |
| 同一账号重复接受（已是成员） | `ConflictException: The current account is already a member in this workspace` | `workspace.service.ts:334-345` |
| 前端接受失败 | 仅 `console.error`，清空 localStorage 的邀请 token，**页面无错误提示**，用户停留在自己原来的 workspace | `CompleteInvitation.tsx:34-37` |
| 撤销邀请 | `revokeInvitation` 物理删除 Invitation 行；仅命中 `newUser: null` 的记录，已接受的邀请返回 `ConflictException: Invitation cannot be found` | `workspace.service.ts:399-417` |
| 重发邀请 | 复用**同一个 token**，仅把 `tokenExpiration` 顺延为现在+7天，再发一次邮件 | `workspace.service.ts:439-452` |
| 移除成员 | `deleteUser` 软删除（置 `deletedAt`）；owner 不可删；跨 workspace 不可删 | `workspace.service.ts:565-587`；`user.service.ts:140-159` |

---

## 4. 需要进一步复核的实现风险与边界条件

### 4.1 【代码事实】邀请实际有效期约为 14 天，而非常量名暗示的 7 天

- 创建时 `tokenExpiration = 创建时刻 + 7天`（`workspace.service.ts:297`）。
- 接受时的有效性过滤是 `tokenExpiration > 现在 - 7天`（`workspace.service.ts:319-321`）。
- 联立两式：可接受 ⟺ `创建时刻 + 7天 > 现在 - 7天` ⟺ `现在 < 创建时刻 + 14天`。
- 即一封"已过期"（超过 7 天）的邀请在**第 8~14 天仍可被接受**；`resendInvitation` 把 `tokenExpiration` 重设为现在+7天（`workspace.service.ts:444`），同样获得 14 天有效窗。
- 【推断】正确的过滤应为 `tokenExpiration: { gt: new Date() }`；当前写法疑似把"过期时间"与"宽限期"混用。完成邀请时把 `tokenExpiration` 写成"现在-7天"（`workspace.service.ts:377`）恰好是利用同一偏移让查询不再命中，说明该偏移是有意为之的"作废"技巧，但这同时把有效窗拉长了一倍，建议复核是否为已知行为。

### 4.2 【代码事实】计费未启用时 `inviteUser` 必然失败

- `canInvite` 在 `!this.billingService.isBillingEnabled` 时返回 `false`（`workspace.service.ts:218-220`），`inviteUser` 随即抛出 `BillingLimitationError("Your workspace exceeds its members limitation.")`（`workspace.service.ts:236-240`）。
- `isBillingEnabled` 仅当环境变量 `BILLING_ENABLED === "true"` 且 Stigg 初始化成功时为真（`packages/amplication-server/src/core/billing/billing.service.ts:78-80`、`billing.service.ts:87-99`）。
- 对比：`shouldAllowWorkspaceCreation` 在计费未启用时返回 `true`（放行，`workspace.service.ts:76-90`）——两者对"计费关闭"的默认策略相反。
- 【推断】对未配置 Stigg 的自托管部署，邀请功能会被完全阻断，且报错文案具有误导性（"超出成员限制"）。这很可能是把 `true` 误写为 `false` 的缺陷，建议复核。单测中相关用例被 `it.skip` 跳过（`packages/amplication-server/src/core/workspace/workspace.service.spec.ts:514`、`workspace.service.spec.ts:534`），无法提供行为佐证。

### 4.3 【代码事实】"接受后被移除"的邮箱无法再次邀请

- 接受邀请后 Invitation 行保留且 `newUserId` 非空（`workspace.service.ts:372-384`）；移除成员只是软删除 User（`user.service.ts:151-158`），Invitation 行不动。
- 再次邀请同一邮箱时：重复成员检查因 `findUsers` 过滤 `deletedAt: null` 而通过（`user.service.ts:43-51`），但重复邀请检查 `workspaceId_email` 命中旧行 → `ConflictException`（`workspace.service.ts:261-275`）。
- 该旧邀请无法通过 `revokeInvitation` 删除（要求 `newUser: null`，`workspace.service.ts:400-406`），也不会出现在成员列表里（`findMembers` 同样过滤 `newUser: null`，`workspace.service.ts:533-538`），UI 上无入口清理。
- 【推断】结果：一旦"邀请→接受→移除成员"，该邮箱对该 workspace 的邀请通道被永久堵死（除非直接改库）。建议复核是否应在移除成员时级联清理其 Invitation 记录。

### 4.4 【代码事实】`completeInvitation` 不复核成员额度

- 额度检查只存在于 `inviteUser`（`workspace.service.ts:236-240`）；`completeInvitation` 全流程（`workspace.service.ts:310-397`）没有任何 entitlement 校验，只在完成后 `reportUsage` 上报（`workspace.service.ts:386-389`）。
- 【推断】邀请发出后若其他成员占满了席位，被邀请人仍能加入，workspace 成员数可超出套餐上限；属于"邀请时卡点、接受时放行"的设计取舍还是疏漏，需产品确认。

### 4.5 【代码事实】邀请 token 是"不记名"凭证，且不与受邀邮箱绑定

- `completeInvitation` 只按 `token` 查邀请（`workspace.service.ts:316-324`），全程不比较 `invitation.email` 与当前账号邮箱；任何已登录账号拿到链接即可接受。
- token 通过邮件 URL 与 localStorage 传递；GraphQL 的 `Invitation` 类型不暴露 `token` 字段（`packages/amplication-server/src/schema.graphql:1366-1373`），服务端 `Invitation` ObjectType 同样无 token 字段（`packages/amplication-server/src/core/workspace/dto/Invitation.ts`），泄露面可控。
- 【推断】"转发链接即可代领"通常是邀请链接的既定语义，但代码中没有二次确认（如"你正在接受发给 x@y.com 的邀请"），值得在安全评审中确认。

### 4.6 【推断】邀请邮箱大小写未归一化，可能产生同人重复邀请

- `inviteUser` 直接使用 `args.data.email` 原值做查重与落库（`workspace.service.ts:244-299`），而注册/登录都会 `toLowerCase()`（`auth.resolver.ts:70`、`auth.resolver.ts:78`）。
- Prisma 对 PostgreSQL 的字符串等值比较默认大小写敏感，故 `A@x.com` 与 `a@x.com` 可同时存在两条 Invitation（`@@unique` 不拦截），重复成员检查也会漏判。
- 缓解因素【代码事实】：即便重复邀请被接受，`completeInvitation` 的"同账号已是成员"检查仍会拦截第二次加入（`workspace.service.ts:334-345`）。影响仅限于列表中出现冗余 Pending 记录。

### 4.7 【代码事实】前端失败路径无用户提示

- `CompleteInvitation` 的 `onError` 只打印控制台并清除 token（`CompleteInvitation.tsx:34-37`）；token 过期、被撤销、已是成员等失败对被邀请人完全静默，用户只会发现自己进入了"自己的"workspace 而非邀请方的 workspace。

---

## 5. 关键文件索引

| 层 | 文件 | 作用 |
|---|---|---|
| 前端 | `packages/amplication-client/src/Workspaces/InviteMember.tsx` | 邀请表单，发起 `inviteUser` |
| 前端 | `packages/amplication-client/src/Workspaces/MemberList.tsx` / `MemberListItem.tsx` | 成员+待接受邀请列表，删除/撤销/重发 |
| 前端 | `packages/amplication-client/src/App.tsx:123-131` | 从 URL 捕获 `?invitation=` 存入 localStorage |
| 前端 | `packages/amplication-client/src/User/CompleteInvitation.tsx` | 登录后自动发起 `completeInvitation` 并刷新 |
| 前端 | `packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx:266` | `CompleteInvitation` 挂载点 |
| 前端 | `packages/amplication-client/src/User/Signup.tsx` / `SignInForm.tsx` | 注册/登录后回到 `/` |
| GraphQL | `packages/amplication-server/src/core/workspace/workspace.resolver.ts` | `inviteUser` / `revokeInvitation` / `resendInvitation` / `workspaceMembers` |
| GraphQL | `packages/amplication-server/src/core/auth/auth.resolver.ts:142-153` | `completeInvitation` |
| 服务 | `packages/amplication-server/src/core/workspace/workspace.service.ts` | 邀请核心逻辑（`inviteUser` 232-308，`completeInvitation` 310-397，`findMembers` 526-555） |
| 服务 | `packages/amplication-server/src/core/auth/auth.service.ts:657-667` | 完成邀请 + 切换 workspace 签新 JWT |
| 服务 | `packages/amplication-server/src/core/mail/mail.service.ts:28-53` | SendGrid 邀请邮件与链接拼接 |
| 安全 | `packages/amplication-server/src/guards/gql-auth.guard.ts`、`core/permissions/*`、`core/auth/jwt.strategy.ts` | JWT 认证 + 权限串校验 |
| 数据 | `packages/amplication-prisma-db/prisma/schema.prisma:722-737` | `Invitation` 模型与唯一约束 |
| 异常 | `packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts` | 异常 → GraphQL 错误映射 |
