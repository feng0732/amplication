# Amplication Workspace 成员邀请全链路代码说明

> 基于仓库当前 HEAD（分支 `task/1`，提交 `7656495`）逐行阅读整理。
> 每条结论均标注文件路径与行号；结论分为两类：
> **【代码事实】** = 代码明确保证的行为；**【推断】** = 基于代码语义/上下文的合理推论，需进一步复核。
>
> 本文档经第二轮核对修订：补充了 MailService 日志泄露面、completeInvitation 非原子性与并发接受、邮件发送失败的遗留状态、GitHub OAuth / Auth0 SSO 与邀请 token 的衔接、localStorage token 残留生命周期等分析，并修正了初版"凭证泄露面可控"的结论。

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
                                        ├─ SendGrid 邮件
                                        └─ debug 日志记录完整 token  ← 见 4.5
                                        链接=<CLIENT_HOST>/login?invitation=<token>
─────────────────────────────────────────────────────────────────────────
被邀请人点击链接 ────────────────►  浏览器打开 /login?invitation=<token>
                                    App.tsx 把 token 存入 localStorage
                                    ├─ 账号密码 → SignInForm / Signup
                                    ├─ GitHub OAuth → /github → 回调 → AJWT cookie
                                    └─ Auth0 SSO  → /auth/login → 回调 → AJWT cookie
                                    登录/注册成功后进入 /:workspace 布局
                                    WorkspaceLayout 挂载 CompleteInvitation
                                      └─ mutation completeInvitation ─►
                                        AuthResolver.completeInvitation
                                        └─ AuthService.completeInvitation
                                           └─ WorkspaceService.completeInvitation
                                              ├─ 校验 token
                                              ├─ 在目标 workspace 创建 User
                                              └─ 标记 Invitation 已接受
                                              （以上非原子，见 4.2）
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
  - 成功后通过 `refetchQueries: [{ query: GET_WORKSPACE_MEMBERS }]` 刷新成员列表（`InviteMember.tsx:62`），并上报分析事件 `WorkspaceMemberInvite`（`InviteMember.tsx:55-60`）。**注意：refetch 只在成功时执行，失败时当前页面的成员列表不会更新**（与 4.4 节的遗留状态相关）。

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
  - **在发送前调用 `this.logger.debug("sendInvitation", args)`（`mail.service.ts:38`）——`args` 是完整的 `SendInvitationArgs` 对象，包含明文 `invitationToken`。这是初版文档遗漏的泄露点，详见 4.5 节。**
  - 通过 SendGrid 动态模板发送，模板变量为 `inviter_name` 与 `invitation_url`（`mail.service.ts:40-51`）。
  - 注意【代码事实】：模板变量 `inviter_name` 实际传入的是邀请人账号的 **email**（`workspace.service.ts:304` 传的是 `currentUserAccount.email`），字段名 `invitedByUserFullName` 名不副实。

### 1.4 被邀请人打开链接、token 保存（前端）

- 被邀请人访问 `https://<host>/login?invitation=<token>`。
- `App` 组件的全局 `useEffect` 解析 `location.search`，发现 `params.invitation` 即写入 localStorage：`setInvitationToken(params.invitation as string)`（`packages/amplication-client/src/App.tsx:123-131`）；key 为 `"invitationToken"`（`App.tsx:40`）。**该 effect 对任何页面、任何登录状态都生效**【代码事实】。
- 代码注释说明了用 localStorage 的原因：GitHub OAuth（passport）不支持动态 callback，token 必须跨整页跳转保存（`App.tsx:126-128`）。
- `/login` 是公开路由（`packages/amplication-client/src/routes/appRoutes.tsx:543-549`）；若未登录用户直接访问受保护路由，会被重定向到 `/login` 且**保留 query string**（`packages/amplication-client/src/routes/routesUtil.tsx:69-84`，`search: location.search`），因此 `?invitation=` 参数不会在登录跳转中丢失。

### 1.5 注册或登录（四条路径）

**(a) 账号密码登录**：`SignInForm` 提交 `mutation login`，成功后 `setToken(data.login.token)` 并 `history.replace(from + location.search)`，默认回到 `/`（`packages/amplication-client/src/User/SignInForm.tsx:42-54`）。服务端 `AuthResolver.login` → `AuthService.login` 校验密码后签发 JWT（`packages/amplication-server/src/core/auth/auth.resolver.ts:75-80`、`packages/amplication-server/src/core/auth/auth.service.ts:327-364`）。

**(b) 账号密码注册**：`Signup` 提交 `mutation signup`，成功后 `setToken(data.signup.token)` 并跳转 `/`；若 localStorage 中存在邀请 token，则**不附加** `complete-signup=1`（`packages/amplication-client/src/User/Signup.tsx:60-71`）。服务端 `AuthService.signup` 会创建 Account、并调用 `bootstrapUser` 为该账号**创建一个属于他自己的新 workspace**（`auth.service.ts:290-325`）——即新用户先落在自己的 workspace 里，接受邀请后再切换。

**(c) GitHub OAuth**（详见 4.6 节）：
- 入口：`GitHubLoginButton` 是一个普通 `<a href="/github">`（或环境变量 `REACT_APP_GITHUB_CONTROLLER_LOGIN_URL`），整页跳转到服务端（`packages/amplication-client/src/User/GitHubLoginButton.tsx:11-16`）。
- 服务端 `GET /github` 由 `GitHubAuthGuard` 触发 passport 跳转 GitHub（`packages/amplication-server/src/core/auth/auth.controller.ts:44-49`）；回调 `GET /github/callback`（`auth.controller.ts:51-67`）由 `GitHubStrategy.validate` 查找/创建/更新用户（`packages/amplication-server/src/core/auth/github.strategy.ts:20-51`；新用户走 `createGitHubUser` → `bootstrapUser`，同样先建自己的 workspace，`auth.service.ts:209-235`）。
- 然后 `configureJtw` 把 JWT 写入 `AJWT` cookie（domain 取父域名）并 301 重定向到 `<CLIENT_HOST>/?complete-signup=0|1`（`auth.service.ts:631-655`）。**重定向 URL 只带 `complete-signup` 参数，不带邀请 token**【代码事实】。
- 客户端启动时 `index.tsx:18` 调用 `setTokenFromCookie()` 把 AJWT cookie 转为正式 token 并销毁 cookie（`packages/amplication-client/src/authentication/authentication.ts:27-38`）。

**(d) Auth0 SSO / 企业邮箱**（详见 4.6 节）：
- 入口：`Login.tsx` 的 "Continue with SSO" 链接指向 `REACT_APP_AUTH_LOGIN_URI`（`packages/amplication-client/src/User/Login.tsx:88-97`）；`AuthWithWorkEmail` 提交 `signupWithBusinessEmail` mutation 创建 Auth0 用户并触发重置密码邮件（`packages/amplication-client/src/User/AuthWithWorkEmail.tsx:37-46`；`auth.service.ts:155-207`）。
- 服务端 `GET /auth/login` 调 `response.oidc.login`（express-openid-connect），`returnTo` 固定为常量 `/auth/afterCallback`（`auth.controller.ts:24-27`、`auth.controller.ts:69-95`）；Auth0 回调 `GET|POST /auth/callback`（`auth.controller.ts:97-127`）后进入 `GET /auth/afterCallback` → `authService.loginOrSignUp(profile, response)`（`auth.controller.ts:138-151` → `auth.service.ts:584-629`；新用户 `createUser` → `bootstrapUser`，`auth.service.ts:253-274`），最终同样走 `configureJtw` 的 cookie + 重定向。

无论哪条路径，之后 `useCurrentWorkspace` 在路径 `/` 下查询 `currentWorkspace` 并把用户重定向到 `/{workspaceId}`（`packages/amplication-client/src/Workspaces/hooks/useCurrentWorkspace.ts:27-44`）。

### 1.6 完成邀请（前端触发 + 服务端处理）

- `CompleteInvitation` 组件挂载在已认证的工作区布局内（`packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx:266`），且 `WorkspaceLayout` 只有在 `currentWorkspace` 加载完成后才渲染子树（`WorkspaceLayout.tsx:186` 的 `return currentWorkspace ? (...)`）；组件文件头注释也明确要求"必须位于登录后才能到达的位置"（`packages/amplication-client/src/User/CompleteInvitation.tsx:15-19`）。
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
| 7 | 发邮件（SendGrid），失败时**不回滚**已创建的邀请行（见 4.4） | `workspace.service.ts:301-305` → `mail.service.ts:28-53` |

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
| 4 | 写：`prisma.invitation.update`：`tokenExpiration: 现在-7天`（使步骤 1 的查询不再命中，即"作废"token），并 `newUser: connect 新User.id`。**where 条件只有 `id`，不带 `newUser: null` 前置状态校验** | `workspace.service.ts:372-384` |
| 5 | `billingService.reportUsage(workspace.id, BillingFeature.TeamMembers)` 上报用量（内部 try/catch，不会抛出） | `workspace.service.ts:386-389`；`billing.service.ts:136-148` |
| 6 | 上报分析事件 `InvitationAcceptance`（内部 try/catch，不会抛出） | `workspace.service.ts:391-394`；`segmentAnalytics.service.ts:101-121` |

随后 `AuthService.completeInvitation` 调用 `setCurrentWorkspace(accountId, workspace.id)`（`auth.service.ts:657-667`）：查出新 User 的 `AuthUser`、把 `Account.currentUserId` 指向新 User（`account.service.ts:53-66`）、签发含新 `workspaceId` 的 JWT（`auth.service.ts:366-388`、`auth.service.ts:517-527`）返回给前端。

**上述步骤没有包裹在 Prisma `$transaction` 中，步骤 3/4 与 `setCurrentWorkspace` 之间任一步失败都会留下中间状态**，详见 4.2 节。

### 2.6 接受邀请后三方状态变化【代码事实】

1. **Account（账号）**：`Account.currentUserId` 被更新为被邀请 workspace 中的新 User id（`auth.service.ts:385` → `account.service.ts:53-66`）。账号本身不新建（被邀请人必须先注册/登录）；账号原先所在的 workspace（注册时自动创建的那个）保持存在，可通过 `setCurrentWorkspace` 切回。
2. **Workspace 用户（User 表）**：目标 workspace 新增一行 `User`，`isOwner: false`，关联当前 Account（`workspace.service.ts:347-368`）。`User` 表有 `@@unique([accountId, workspaceId])` 约束（`schema.prisma:119`），同一账号在同一 workspace 不可能出现两行。新 User 无任何团队角色，故 `getUserPermissions` 返回空数组（`user.service.ts:74-105`）——在接受邀请的 workspace 里默认没有任何权限串，直到被分进某个 Team/Role【推断：基于 `getUserPermissions` 只聚合团队角色与 owner 特判】。
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
- 注意【代码事实】：`findMembers` 不过滤 `tokenExpiration`，**已过期但未接受的邀请仍以 Pending 显示**（`workspace.service.ts:533-538`），UI 上不区分"有效待接受"与"已过期"。

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
| 同一账号**并发**重复接受 | 两者都通过检查，后到的 `users.create` 触发 `User.accountId_workspaceId_unique` P2002 → `GraphQLUniqueKeyException`；最终只产生一个成员 | `schema.prisma:119`；`GqlResolverExceptions.filter.ts:56-63` |
| **不同账号并发**接受同一 token | **两个账号都能成功加入**（TOCTOU 竞争，无事务/条件更新保护），详见 4.3 | `workspace.service.ts:316-384` |
| 前端接受失败 | 仅 `console.error`，清空 localStorage 的邀请 token，**页面无错误提示**，用户停留在自己原来的 workspace | `CompleteInvitation.tsx:34-37` |
| 撤销邀请 | `revokeInvitation` 物理删除 Invitation 行；仅命中 `newUser: null` 的记录，已接受的邀请返回 `ConflictException: Invitation cannot be found` | `workspace.service.ts:399-417` |
| 重发邀请 | 复用**同一个 token**（不轮换），仅把 `tokenExpiration` 顺延为现在+7天，再发一次邮件；邮件发送失败不回滚（见 4.4） | `workspace.service.ts:439-452` |
| 移除成员 | `deleteUser` 软删除（置 `deletedAt`）；owner 不可删；跨 workspace 不可删 | `workspace.service.ts:565-587`；`user.service.ts:140-159` |

---

## 4. 需要进一步复核的实现风险与边界条件

### 4.1 【代码事实】邀请实际有效期约为 14 天，而非常量名暗示的 7 天

- 创建时 `tokenExpiration = 创建时刻 + 7天`（`workspace.service.ts:297`）。
- 接受时的有效性过滤是 `tokenExpiration > 现在 - 7天`（`workspace.service.ts:319-321`）。
- 联立两式：可接受 ⟺ `创建时刻 + 7天 > 现在 - 7天` ⟺ `现在 < 创建时刻 + 14天`。
- 即一封"已过期"（超过 7 天）的邀请在**第 8~14 天仍可被接受**；`resendInvitation` 把 `tokenExpiration` 重设为现在+7天（`workspace.service.ts:444`），同样获得 14 天有效窗。
- 【推断】正确的过滤应为 `tokenExpiration: { gt: new Date() }`；当前写法疑似把"过期时间"与"宽限期"混用。完成邀请时把 `tokenExpiration` 写成"现在-7天"（`workspace.service.ts:377`）恰好是利用同一偏移让查询不再命中，说明该偏移是有意为之的"作废"技巧，但这同时把有效窗拉长了一倍，建议复核是否为已知行为。

### 4.2 【代码事实】`completeInvitation` 的多步写入非原子，中途失败留下不一致状态

`completeInvitation` 依次执行：创建 User（`workspace.service.ts:347-368`）→ 更新 Invitation（`workspace.service.ts:372-384`）→ 上报用量/分析（不会抛出，见 2.5 表）→ `setCurrentWorkspace` 更新 `Account.currentUserId` 并签 JWT（`auth.service.ts:657-667` → `auth.service.ts:366-388`）。**全程无 `$transaction`、无补偿逻辑**【代码事实】。各失败点的实际结果：

| 失败点 | 遗留状态 | 后果 |
|---|---|---|
| 步骤 3（创建 User）失败 | 无任何写入，token 仍有效 | 可安全重试【代码事实】 |
| 步骤 4（更新 Invitation）失败 | User 已加入，但 Invitation 仍 `newUser: null` 且 token 仍在有效期内 | 该账号重试会被"已是成员"拦截（`workspace.service.ts:341-345`）；**但另一个账号仍可用同一 token 再加入**（步骤 1 的查询仍命中）；前端走 `onError` 静默清空 token，被邀请人不知道自己其实已加入【代码事实 + 推断（需 DB 层故障才会触发，概率低）】 |
| `setCurrentWorkspace` 失败（如 `account.update` 故障） | 成员已加入、邀请已消费，但 `Account.currentUserId` 未切换 | 客户端收到错误 → 清空 localStorage token → 用户停留在旧 workspace；可通过工作区切换器手动切换恢复（`useWorkspaceSelector.ts:50-62`）【代码事实】 |

### 4.3 【代码事实】同一 token 被不同账号并发接受时，两个账号都会成为成员

- 接受流程是"读-改-写"的 TOCTOU 序列：两个并发请求都可能在对方提交前通过 `findMany`（`newUser: null` 仍成立，`workspace.service.ts:316-324`）和重复成员检查（`workspace.service.ts:334-345`）。
- 随后：两个请求各自 `users.create`——不同 accountId 不违反 `@@unique([accountId, workspaceId])`（`schema.prisma:119`），**两个 User 都会创建成功**。
- 两个请求各自执行 `invitation.update`——其 where 条件只有 `id`（`workspace.service.ts:373-375`），**不带 `newUser: null` 的乐观锁条件**，后提交者覆盖先提交者，`newUserId` 指向最后写入的 User。
- 最终状态【代码事实】：两个账号都成为 workspace 成员（各拿到有效 JWT），`reportUsage` 被调用两次（`workspace.service.ts:386-389`），Invitation 只关联其中一个 User——另一个 User 成为不与邀请记录关联的"额外"成员。
- 对比：同一账号并发接受会被 `User.accountId_workspaceId_unique` 唯一约束兜底（第二个请求在 `users.create` 处 P2002 失败），状态保持一致【代码事实】。
- 【推断】该竞争窗口需要两个不同账号几乎同时（毫秒级）提交同一 token，现实中多见于"邀请链接被转发给多人"的场景；防护手段（条件更新 `where: { id, newUser: null }` 或事务）在代码中不存在。

### 4.4 【代码事实】`inviteUser`/`resendInvitation` 数据库写入先于邮件发送，发送失败不回滚

- `inviteUser`：先 `invitation.create`（`workspace.service.ts:283-299`）后 `mailService.sendInvitation`（`workspace.service.ts:301-305`）；`MailService.sendInvitation` 内 `await this.client.send(msg)` 无 try/catch（`mail.service.ts:51`），`WorkspaceService` 侧也无 try/catch——SendGrid 调用失败时异常直接冒泡到异常过滤器的 generic 分支（`GqlResolverExceptions.filter.ts:79-89`：生产环境返回 `GraphQLInternalServerError`，非生产返回带原始信息的 `ApolloError`）。
- 发送失败后的遗留状态与后续影响【代码事实】：
  1. **Invitation 行已存在**（`newUser: null`，token 有效），但被邀请人永远收不到邮件；
  2. 邀请人看到"服务器错误"，若直接重试同一邮箱 → `ConflictException: Invitation ... already exist`（`workspace.service.ts:271-275`）；
  3. 前端 `InviteMember` 失败时不触发 `refetchQueries`（只在 `onCompleted` 刷新，`InviteMember.tsx:54-63`），当前页面看不到这条遗留邀请，**下次加载成员列表才会出现**（`findMembers` 查 `newUser: null`，`workspace.service.ts:533-538`）；
  4. 恢复路径：在成员列表对该 Pending 邀请点"重发"（`resendInvitation`），或"撤销"（`revokeInvitation` 物理删除，`workspace.service.ts:412-416`）后重新邀请。
- `resendInvitation` 同样先写库（顺延 `tokenExpiration`，`workspace.service.ts:439-446`）后发邮件（`workspace.service.ts:448-452`）：邮件失败时**过期时间已被延长但无人收到邮件**。
- 另外【代码事实】：`resendInvitation` **不轮换 token**（复用原 token，只改过期时间）——若旧链接已泄露，重发不能让旧链接失效；只有 `revokeInvitation`（删行）或接受完成（置过期）能使 token 失效。

### 4.5 【代码事实】邀请 token 的泄露面：GraphQL 不暴露，但服务端 debug 日志明文记录完整 token（对初版结论的修正）

- **不暴露的面**【代码事实】：GraphQL 的 `Invitation` 类型不含 `token` 字段（`packages/amplication-server/src/schema.graphql:1366-1373`；服务端 ObjectType 同样无 token 字段，`packages/amplication-server/src/core/workspace/dto/Invitation.ts`），任何 GraphQL 查询都取不到 token。
- **暴露的面**【代码事实】：
  1. **服务端日志**：`MailService.sendInvitation` 开头执行 `this.logger.debug("sendInvitation", args)`（`mail.service.ts:38`），`args` 是完整 `SendInvitationArgs`（含 `to`、`invitationToken`、`invitedByUserFullName`，见 `workspace.service.ts:301-305` 的调用）。`AmplicationLogger.debug` 把对象参数原样透传给底层 Logger（`libs/util/nestjs/logging/src/logger.service.ts:34-37`）；底层 winston Logger **默认级别就是 Debug**（`libs/util/logging/src/lib/logging.ts:17`：`options.logLevel ?? LogLevel.Debug`），而服务端 `AmplicationLoggerModule.forRoot({ component: SERVICE_NAME })` 未覆盖 logLevel（`packages/amplication-server/src/app.module.ts:66-68`），日志输出到 Console（`logging.ts:10`），生产环境以 JSON 序列化（`logging.ts:48-52`）。**即默认配置下，每一次邀请/重发都会把明文 token 写入应用 stdout 日志**。
  2. **URL 通道**：token 以 query 参数出现在邀请链接中（`mail.service.ts:36`），会进入浏览器历史、以及任何记录 URL 的代理/网关日志【推断：取决于部署拓扑】。
  3. **localStorage**：token 明文存于浏览器 localStorage（`App.tsx:129`），同源任何 JS 可读，且长期残留（见 4.7）。
  4. **SendGrid**：完整邀请 URL 作为模板变量发送（`mail.service.ts:44-49`）。
- 结论修正：初版"泄露面可控"仅对 GraphQL 接口成立。综合代码事实，token 的暴露面至少包括**服务端应用日志（默认级别即输出）**、邮件通道、浏览器 URL/localStorage。其中日志明文记录是最值得复核的一项——它使任何能读应用日志的人都获得可直接兑换的 bearer 凭证。【推断】是否构成实际风险取决于日志收集范围与保留策略，代码层面无法保证。

### 4.6 【代码事实】GitHub OAuth 与 Auth0 SSO 均不携带邀请 token 跨跳转，完全依赖 localStorage 衔接

- 两条 OAuth 路径的服务端重定向都只带 `complete-signup` 参数：`configureJtw` 设置 `AJWT` cookie 后 301 到 `<CLIENT_HOST>/?complete-signup=0|1`（`auth.service.ts:636-654`）。
- GitHub：`/github` → GitHub → `/github/callback`（`auth.controller.ts:44-67`；`github.strategy.ts:20-51`）。passport-github2 的 state 参数未被用来传递业务数据，前端注释明确说明"github-passport does not support dynamic callback"是用 localStorage 的原因（`App.tsx:126-128`）。
- Auth0 SSO：`/auth/login` 的 `returnTo` 是硬编码常量 `/auth/afterCallback`（`auth.controller.ts:27`、`auth.controller.ts:86`），OIDC 往返中没有任何携带邀请 token 的机制。
- 衔接时序【代码事实】：被邀请人先在 `/login?invitation=T` 页面加载时由 `App.tsx:123-131` 把 T 写入 localStorage → 点击 GitHub/SSO 整页跳转离开 → 认证完成回到 `<CLIENT_HOST>/` → `index.tsx:18` 的 `setTokenFromCookie()` 完成登录态 → 进入 workspace 布局后 `CompleteInvitation` 从 localStorage 取出 T 完成邀请。
- 推论【推断】：若被邀请人在"打开邀请链接"和"完成 OAuth 登录"之间更换了浏览器/设备/无痕窗口，localStorage 中的 token 不存在，OAuth 回来后不会触发 `CompleteInvitation`——邀请不会被接受，且无任何提示；用户需要重新点击邀请链接。

### 4.7 【代码事实】localStorage 中邀请 token 的残留生命周期：只在成功/失败两个端点被清除

- **写入**：仅 `App.tsx:129`（任何含 `?invitation=` 的 URL 渲染时，与登录状态无关）。
- **读取**：`CompleteInvitation.tsx:22-25`（仅 `WorkspaceLayout` 内，且 `currentWorkspace` 加载完成才渲染，`WorkspaceLayout.tsx:186`、`WorkspaceLayout.tsx:266`）；`Signup.tsx:62-64`（`getItem` 判断是否来自邀请，**只读不清除**）。
- **清除**：仅两处，都是置为空字符串（`isEmpty` 判断等效于移除）：`CompleteInvitation.tsx:30`（接受成功）、`CompleteInvitation.tsx:36`（接受失败）。
- **不清除的路径**【代码事实】：登出 `unsetToken` 只删除 `@@TOKEN`（`authentication.ts:46-50`）；切换 workspace；JWT 过期；关闭标签页。localStorage 本身无 TTL。
- 由此产生的残留场景：
  1. 【代码事实】被邀请人打开链接后未登录就离开 → token 在该浏览器**无限期残留**；之后任何人（可以是另一个账号）在同一浏览器登录并进入 workspace 布局，`CompleteInvitation` 会立即用残留 token 发起接受——若邀请仍在有效期（创建/重发后 14 天内，见 4.1）则**静默加入**，若已失效则静默清除。
  2. 【代码事实】已登录用户点击邀请链接 → `/login` 路由对已认证用户重定向到 `/`（`routesUtil.tsx:87-92`）→ 布局挂载后 `CompleteInvitation` 立即以**当前账号**完成邀请，全程无"确认加入"界面。
  3. 【推断】场景 1 在共享/公共浏览器上意味着"后登录者顶替接受邀请"；代码中没有任何把 token 与特定账号/会话绑定的机制（与 4.8 的 bearer 语义互为因果）。

### 4.8 【代码事实】邀请 token 是"不记名"凭证，且不与受邀邮箱绑定

- `completeInvitation` 只按 `token` 查邀请（`workspace.service.ts:316-324`），全程不比较 `invitation.email` 与当前账号邮箱；任何已登录账号拿到链接即可接受。
- 【推断】"转发链接即可代领"通常是邀请链接的既定语义，但代码中没有二次确认（如"你正在接受发给 x@y.com 的邀请"），结合 4.7 的残留场景，值得在安全评审中确认。

### 4.9 【代码事实】计费未启用时 `inviteUser` 必然失败

- `canInvite` 在 `!this.billingService.isBillingEnabled` 时返回 `false`（`workspace.service.ts:218-220`），`inviteUser` 随即抛出 `BillingLimitationError("Your workspace exceeds its members limitation.")`（`workspace.service.ts:236-240`）。
- `isBillingEnabled` 仅当环境变量 `BILLING_ENABLED === "true"` 且 Stigg 初始化成功时为真（`packages/amplication-server/src/core/billing/billing.service.ts:78-80`、`billing.service.ts:87-99`）。
- 对比：`shouldAllowWorkspaceCreation` 在计费未启用时返回 `true`（放行，`workspace.service.ts:76-90`）——两者对"计费关闭"的默认策略相反。
- 【推断】对未配置 Stigg 的自托管部署，邀请功能会被完全阻断，且报错文案具有误导性（"超出成员限制"）。这很可能是把 `true` 误写为 `false` 的缺陷，建议复核。单测中相关用例被 `it.skip` 跳过（`packages/amplication-server/src/core/workspace/workspace.service.spec.ts:514`、`workspace.service.spec.ts:534`），无法提供行为佐证。

### 4.10 【代码事实】"接受后被移除"的邮箱无法再次邀请

- 接受邀请后 Invitation 行保留且 `newUserId` 非空（`workspace.service.ts:372-384`）；移除成员只是软删除 User（`user.service.ts:151-158`），Invitation 行不动。
- 再次邀请同一邮箱时：重复成员检查因 `findUsers` 过滤 `deletedAt: null` 而通过（`user.service.ts:43-51`），但重复邀请检查 `workspaceId_email` 命中旧行 → `ConflictException`（`workspace.service.ts:261-275`）。
- 该旧邀请无法通过 `revokeInvitation` 删除（要求 `newUser: null`，`workspace.service.ts:400-406`），也不会出现在成员列表里（`findMembers` 同样过滤 `newUser: null`，`workspace.service.ts:533-538`），UI 上无入口清理。
- 【推断】结果：一旦"邀请→接受→移除成员"，该邮箱对该 workspace 的邀请通道被永久堵死（除非直接改库）。建议复核是否应在移除成员时级联清理其 Invitation 记录。

### 4.11 【代码事实】`completeInvitation` 不复核成员额度

- 额度检查只存在于 `inviteUser`（`workspace.service.ts:236-240`）；`completeInvitation` 全流程（`workspace.service.ts:310-397`）没有任何 entitlement 校验，只在完成后 `reportUsage` 上报（`workspace.service.ts:386-389`）。
- 【推断】邀请发出后若其他成员占满了席位，被邀请人仍能加入，workspace 成员数可超出套餐上限；属于"邀请时卡点、接受时放行"的设计取舍还是疏漏，需产品确认。

### 4.12 【推断】邀请邮箱大小写未归一化，可能产生同人重复邀请

- `inviteUser` 直接使用 `args.data.email` 原值做查重与落库（`workspace.service.ts:244-299`），而注册/登录都会 `toLowerCase()`（`auth.resolver.ts:70`、`auth.resolver.ts:78`）。
- Prisma 对 PostgreSQL 的字符串等值比较默认大小写敏感，故 `A@x.com` 与 `a@x.com` 可同时存在两条 Invitation（`@@unique` 不拦截），重复成员检查也会漏判。
- 缓解因素【代码事实】：即便重复邀请被接受，`completeInvitation` 的"同账号已是成员"检查仍会拦截第二次加入（`workspace.service.ts:334-345`）。影响仅限于列表中出现冗余 Pending 记录。

### 4.13 【代码事实】前端失败路径无用户提示

- `CompleteInvitation` 的 `onError` 只打印控制台并清除 token（`CompleteInvitation.tsx:34-37`）；token 过期、被撤销、已是成员等失败对被邀请人完全静默，用户只会发现自己进入了"自己的"workspace 而非邀请方的 workspace。

---

## 5. 关键文件索引

| 层 | 文件 | 作用 |
|---|---|---|
| 前端 | `packages/amplication-client/src/Workspaces/InviteMember.tsx` | 邀请表单，发起 `inviteUser` |
| 前端 | `packages/amplication-client/src/Workspaces/MemberList.tsx` / `MemberListItem.tsx` | 成员+待接受邀请列表，删除/撤销/重发 |
| 前端 | `packages/amplication-client/src/App.tsx:123-131` | 从 URL 捕获 `?invitation=` 存入 localStorage |
| 前端 | `packages/amplication-client/src/User/CompleteInvitation.tsx` | 登录后自动发起 `completeInvitation` 并刷新 |
| 前端 | `packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx:186,266` | `CompleteInvitation` 挂载点（currentWorkspace 就绪后） |
| 前端 | `packages/amplication-client/src/User/Signup.tsx` / `SignInForm.tsx` / `GitHubLoginButton.tsx` / `AuthWithWorkEmail.tsx` | 四条登录/注册入口 |
| 前端 | `packages/amplication-client/src/authentication/authentication.ts` / `index.tsx:18` | token 存取；AJWT cookie 转换 |
| GraphQL | `packages/amplication-server/src/core/workspace/workspace.resolver.ts` | `inviteUser` / `revokeInvitation` / `resendInvitation` / `workspaceMembers` |
| GraphQL | `packages/amplication-server/src/core/auth/auth.resolver.ts:142-153` | `completeInvitation` |
| 服务 | `packages/amplication-server/src/core/workspace/workspace.service.ts` | 邀请核心逻辑（`inviteUser` 232-308，`completeInvitation` 310-397，`findMembers` 526-555） |
| 服务 | `packages/amplication-server/src/core/auth/auth.service.ts:584-667` | OAuth 登录/注册、`configureJtw`、完成邀请 + 切换 workspace 签新 JWT |
| 服务 | `packages/amplication-server/src/core/auth/auth.controller.ts` | GitHub / Auth0 SSO 的 HTTP 端点 |
| 服务 | `packages/amplication-server/src/core/mail/mail.service.ts:28-53` | SendGrid 邀请邮件与链接拼接；debug 日志记录完整 token |
| 服务 | `libs/util/nestjs/logging/src/logger.service.ts`、`libs/util/logging/src/lib/logging.ts` | 日志透传与默认 Debug 级别 |
| 安全 | `packages/amplication-server/src/guards/gql-auth.guard.ts`、`core/permissions/*`、`core/auth/jwt.strategy.ts` | JWT 认证 + 权限串校验 |
| 数据 | `packages/amplication-prisma-db/prisma/schema.prisma:722-737`（Invitation）、`schema.prisma:119`（User 唯一约束） | 数据模型与约束 |
| 异常 | `packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts` | 异常 → GraphQL 错误映射 |
