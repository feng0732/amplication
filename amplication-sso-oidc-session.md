# Amplication SSO/OIDC 登录、令牌续期与会话失效协作分析

## 一、整体架构概述

Amplication 采用**双轨认证模式**：
1. **GitHub OAuth2** - 通过 Passport.js 的 `passport-github2` 策略实现
2. **OIDC (Auth0)** - 通过 `express-openid-connect` 中间件实现

系统基于 NestJS 框架构建，采用 JWT 作为会话令牌，结合 Cookie（短期临时传递）和 LocalStorage（长期存储）实现前后端分离的会话管理。

---

## 二、身份提供方（IdP）回调流程深度分析

### 2.1 支持的身份提供方

| 提供方 | 实现方式 | 核心模块 |
|--------|---------|---------|
| GitHub | Passport Strategy | [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts) |
| Auth0 (OIDC) | express-openid-connect 中间件 | [oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts) |
| Local (邮箱密码) | 本地数据库校验 | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L327-L364) |

### 2.2 OIDC (Auth0) 回调完整流程

#### 第一步：OIDC 中间件初始化

在 [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.module.ts#L84-L94) 中，`OpenIDConnectAuthMiddleware` 被应用于以下路由：

```
/auth/login
/auth/logout
/auth/callback
/auth/afterCallback
```

中间件配置（[oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts#L20-L38)）：

```typescript
this.middleware = auth({
  authRequired: false,           // 不强制所有请求认证
  authorizationParams: {
    response_type: "code",       // 使用 Authorization Code 流程
    scope: "openid profile email",
  },
  idpLogout: true,               // 登出时同时登出 IdP
  clientID,
  clientSecret,
  baseURL: `${baseURL}/auth`,
  routes: {
    login: false,                // 禁用默认登录路由，使用自定义
    logout: false,               // 禁用默认登出路由，使用自定义
    postLogoutRedirect: `${this.clientHost}/login`,
  },
  issuerBaseURL: `https://${issuerBaseUrl}`,
  secret: clientSecret,
});
```

#### 第二步：登录发起

在 [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts#L70-L95) 的 `/auth/login` 路由：

```typescript
@Get(AUTH_LOGIN_PATH)
async auth0Login(@Req() request: Request, @Res() response: Response) {
  const screenHint = request.query.work_email
    ? { screen_hint: "signup", login_hint: request.query.work_email as string }
    : { screen_hint: "login-id" };
  await response.oidc.login({
    authorizationParams: { ...screenHint },
    returnTo: AUTH_AFTER_CALLBACK_PATH,  // 登录成功后跳转到 /auth/afterCallback
  });
}
```

这会重定向用户到 Auth0 的授权页面，携带 `response_type=code`，使用 **Authorization Code Flow**。

#### 第三步：IdP 回调处理

Auth0 完成认证后，回调到 `/auth/callback`，同时支持 GET 和 POST 方法：

```typescript
// GET /auth/callback
@Get(AUTH_CALLBACK_PATH)
async auth0Callback(...) {
  await response.oidc.callback({
    redirectUri: `${this.host}/${AUTH_CALLBACK_PATH}`,
  });
}

// POST /auth/callback
@Post(AUTH_CALLBACK_PATH)
async auth0CallbackPost(...) {
  await response.oidc.callback({
    redirectUri: `${this.host}/${AUTH_CALLBACK_PATH}`,
  });
}
```

`response.oidc.callback()` 由 `express-openid-connect` 提供，完成以下工作：
1. 接收 Authorization Code
2. 向 IdP Token 端点换取 `access_token`、`id_token`、`refresh_token`
3. 验证 id_token 的签名、过期时间、issuer、audience
4. 将用户信息加密存储到会话 Cookie（`appSession`）

#### 第四步：回调后业务逻辑处理

`express-openid-connect` 处理完令牌交换后，会重定向到之前设置的 `returnTo` 地址，即 `/auth/afterCallback`：

```typescript
@Get(AUTH_AFTER_CALLBACK_PATH)
async authorizationCode(
  @Req() request: GitHubRequest,
  @Res() response: Response
): Promise<void> {
  requiresAuth();                          // 确保用户已通过 OIDC 认证
  const profile = <AuthProfile>request.oidc.user;  // 从会话中取出用户信息
  await this.authService.loginOrSignUp(profile, response);
}
```

#### 第五步：登录或注册逻辑

在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L584-L629) 的 `loginOrSignUp` 方法中：

```typescript
async loginOrSignUp(profile: AuthProfile, response: Response): Promise<void> {
  // 1. 通过 githubId 或 email 查找已有用户
  let user = await this.getAuthUser({
    account: {
      OR: [{ githubId: profile.sub }, { email: profile.email }],
    },
  });

  let isNew: boolean;
  const existingUser = !!user;

  // 2. 新用户：创建账户、工作区
  if (!user) {
    if (this.signupDisabled) {
      // 重定向到前端，携带错误参数
      response.redirect(301, `${this.clientHost}?error=signup_disabled`);
      return;
    }
    user = await this.createUser(profile);
    isNew = true;
  }

  // 3. 免费用户在 signupDisabled 时禁止登录
  if (this.signupDisabled && existingUser &&
      (await this.isFreeTierWorkspace(user.workspace.id))) {
    response.redirect(301, `${this.clientHost}?error=free_tier_disabled`);
    return;
  }

  // 4. 更新用户的 githubId（用于账户关联）
  if (!user.account.githubId || user.account.githubId !== profile.sub) {
    user = await this.updateUser(user, { githubId: profile.sub });
    isNew = false;
  }

  // 5. 埋点追踪
  this.trackCompleteEmailSignup(user.account, profile, existingUser);

  // 6. 签发 JWT 并写入 Cookie 后重定向
  await this.configureJtw(response, user, isNew);
}
```

#### 第六步：JWT 签发与 Cookie 传递

在 `configureJtw` 方法（[auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L631-L655)）中：

```typescript
async configureJtw(response: Response, user: AuthUser, isNew: boolean): Promise<void> {
  const token = await this.prepareToken(user);   // 生成 JWT
  const url = stringifyUrl({
    url: this.clientHost,
    query: { "complete-signup": isNew ? "1" : "0" },
  });

  // 计算 Cookie 的 domain（取主域名的后两级，支持跨子域共享）
  const clientDomain = new URL(url).hostname;
  const cookieDomainParts = clientDomain.split(".");
  const cookieDomain = cookieDomainParts
    .slice(Math.max(cookieDomainParts.length - 2, 0))
    .join(".");

  // 写入临时 Cookie
  response.cookie("AJWT", token, {
    domain: cookieDomain,
    secure: true,       // 仅 HTTPS 传输
  });

  // 301 重定向到前端
  response.redirect(301, url);
}
```

### 2.3 GitHub OAuth2 回调流程

GitHub 回调流程与 OIDC 类似，但使用 Passport 中间件：

1. `GET /github` → `GitHubAuthGuard` 触发重定向到 GitHub 授权页
2. GitHub 认证后回调到 `GET /github/callback`
3. `GitHubAuthGuard` 调用 `GitHubStrategy.validate()` 完成用户查找/创建
4. 在 [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts#L17-L53) 中：

```typescript
async validate(
  accessToken: string,
  refreshToken: string,   // GitHub refresh_token（当前系统未存储使用）
  profile: Profile,
  done: (err: any, user: AuthUser, info: any) => void
): Promise<void> {
  const email = await getEmail(accessToken);
  // 查找或创建用户，返回 isNew 标识
  const user = await this.authService.getAuthUser({...});
  if (!user) {
    return done(null, await this.authService.createGitHubUser(profile, email), { isNew: true });
  }
  // ...
  return done(null, user, { isNew: false });
}
```

5. 控制器中调用 `authService.configureJtw()`，流程与 OIDC 相同

---

## 三、令牌续期机制分析

### 3.1 令牌体系结构

Amplication 使用**两层令牌**：

| 令牌类型 | 生成位置 | 存储位置 | 用途 | 有效期 |
|---------|---------|---------|-----|-------|
| **JWT (User Token)** | [auth.service.ts#prepareToken](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L517-L527) | LocalStorage (`@@TOKEN`) | 日常 API 调用鉴权 | 未设置 expiresIn（无过期） |
| **API Token** | [auth.service.ts#prepareApiToken](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L534-L546) | 数据库（哈希存储） | CI/CD、自动化脚本 | 30 天无访问自动失效 |
| **IdP Token** (access_token/id_token/refresh_token) | Auth0/GitHub | express-openid-connect 加密会话 Cookie | IdP 层会话 | 由 IdP 配置 |
| **AJWT Cookie** | 服务器设置 | 浏览器 Cookie（临时） | SSO 回调后向前端传递 JWT | 会话 Cookie（关闭浏览器即失效） |

JWT 载荷结构（[jwt.dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/dto/jwt.dto.ts)）：

```typescript
export interface JwtDto {
  accountId: string;
  userId?: string | null;
  workspaceId?: string | null;
  roles?: string[] | null;
  permissions: RolesPermissions[];
  type?: EnumTokenType;        // "User" 或 "ApiToken"
  tokenId?: string;            // API Token 专属
}
```

### 3.2 前端令牌获取与持久化

#### Cookie 到 LocalStorage 的传递机制

在应用入口 [index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/index.tsx#L18)：

```typescript
setTokenFromCookie();
```

该函数实现（[authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts#L27-L39)）：

```typescript
export function setTokenFromCookie(): void {
  const tokenFromCookie = getCookie(TEMPORARY_JWT_COOKIE_NAME); // "AJWT"
  if (tokenFromCookie && !token) {
    setToken(tokenFromCookie);          // 存入 LocalStorage 和内存
    // 删除临时 Cookie，避免泄露
    const cookieDomainParts = window.location.hostname.split(".");
    const temporaryCookieDomain = cookieDomainParts
      .slice(Math.max(cookieDomainParts.length - 2, 0))
      .join(".");
    expireCookie(TEMPORARY_JWT_COOKIE_NAME, temporaryCookieDomain);
  }
}
```

这个设计的关键点：
- **安全性**：JWT 仅通过 HTTPS Cookie 短暂传递，随后立即转移到 LocalStorage 并删除 Cookie
- **一次性**：Cookie 为会话级，不持久化；LocalStorage 作为主存储
- **跨域处理**：Cookie domain 取主域名后两级，支持 `app.amplication.com` 与其他子域间共享

#### 令牌存储

```typescript
export function setToken(newToken: string) {
  token = newToken;                              // 内存缓存
  localStorage.setItem(TOKEN_KEY, newToken);     // LocalStorage 持久化
  eventEmitter.emit("change", token);            // 事件通知
}
```

#### GraphQL 请求令牌附加

在 [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/graphqlClient.ts#L45-L56) 中：

```typescript
const authLink = setContext((_, { headers }) => {
  const token = getToken();
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
      [ANALYTICS_SESSION_ID_HEADER_KEY]: getSessionId(),
    },
  };
});
```

WebSocket 订阅也使用相同机制：

```typescript
connectionParams: () => ({
  authorization: `Bearer ${getToken()}`,
}),
```

### 3.3 后端 JWT 验证

在 [jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts#L22-L43) 中：

```typescript
async validate(req, payload: JwtDto): Promise<AuthUser> {
  // API Token 额外校验：检查 tokenId、token hash 和 lastAccessAt
  if (payload.type === EnumTokenType.ApiToken) {
    const jwt = ExtractJwt.fromAuthHeaderAsBearerToken()(req);
    const isValid = await this.authService.validateApiToken({
      userId: payload.userId,
      tokenId: payload.tokenId,
      token: jwt,
    });
    if (!isValid === true) {
      throw new UnauthorizedException();
    }
  }

  // 根据 userId 查询用户（确保用户存在且未被删除）
  const user = await this.authService.getAuthUser({
    id: payload.userId,
  });
  if (!user) {
    throw new UnauthorizedException();
  }
  return user;
}
```

### 3.4 API Token 滑动续期

API Token 采用**滑动过期**策略（[auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L425-L452)）：

```typescript
const TOKEN_EXPIRY_DAYS = 30;

async validateApiToken(args: { userId: string; tokenId: string; token: string }): Promise<boolean> {
  // 过期阈值：当前时间往前推 30 天
  const lastAccessThreshold = subDays(new Date(), TOKEN_EXPIRY_DAYS);

  // 只有在 lastAccessAt 在阈值内才算有效，同时更新 lastAccessAt
  const apiToken = await this.prismaService.apiToken.updateMany({
    where: {
      userId: args.userId,
      id: args.tokenId,
      lastAccessAt: { gt: lastAccessThreshold },  // 30 天内有访问
      user: { deletedAt: null },
    },
    data: {
      lastAccessAt: new Date(),  // 滑动续期：每次有效访问重置计时
    },
  });

  return apiToken.count === 1;
}
```

> **注意**：普通 User JWT Token 没有设置 `expiresIn`，理论上永久有效。续期仅通过「切换工作区」等操作重新签发新 Token 实现。

---

## 四、会话失效机制分析

### 4.1 多层会话失效机制

系统存在**四层会话失效**，从内到外依次为：

```
┌──────────────────────────────────────────────────────────┐
│ Layer 4: LocalStorage Token (客户端)                    │
│   - 用户手动登出时 unsetToken()                          │
│   - 浏览器清除数据/LocalStorage 被清理                   │
├──────────────────────────────────────────────────────────┤
│ Layer 3: User JWT 有效性 (服务端无状态)                  │
│   - JWT 本身不设置 expiresIn                             │
│   - 通过查询 user.deletedAt 判断用户是否被删除            │
│   - 工作区切换时重新签发 Token（旧 Token 理论仍有效）      │
├──────────────────────────────────────────────────────────┤
│ Layer 2: API Token 滑动过期                              │
│   - lastAccessAt 超过 30 天自动失效                       │
│   - 用户主动删除（deleteApiToken）                        │
├──────────────────────────────────────────────────────────┤
│ Layer 1: IdP 会话 (express-openid-connect)              │
│   - 加密 Cookie 存储 IdP 的 access_token/refresh_token   │
│   - 依赖 IdP 侧的会话超时配置                             │
│   - 调用 /auth/logout 触发 IdP 侧登出                    │
└──────────────────────────────────────────────────────────┘
```

### 4.2 主动登出流程

#### 服务端 OIDC 登出

在 [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts#L131-L136)：

```typescript
@Get(AUTH_LOGOUT_PATH)
async auth0Logout(@Req() request: Request, @Res() response: Response) {
  await response.oidc.logout({
    returnTo: `${this.clientHost}/login`,
  });
}
```

`response.oidc.logout()` 执行：
1. 清除服务端 `appSession` Cookie（包含 IdP tokens）
2. 重定向到 Auth0 登出端点，同步销毁 IdP 会话
3. 最终重定向回前端 `/login` 页面

#### 前端登出

前端通过 `unsetToken()` 清除本地凭证（[authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts#L47-L51)）：

```typescript
export function unsetToken(): void {
  token = null;
  localStorage.removeItem(TOKEN_KEY);
  eventEmitter.emit("change", token);
}
```

### 4.3 被动失效场景

#### 场景 1：用户被删除

JwtStrategy 每次请求都会查询 `user`，如果用户已软删除（`deletedAt != null`），`getAuthUser` 返回 null，抛出 `UnauthorizedException`。

#### 场景 2：JWT 无法验证

如果 JWT 签名验证失败（密钥变更、Token 篡改），Passport-JWT 策略自动抛出 401。

#### 场景 3：API Token 30 天未使用

在 `validateApiToken` 中通过 `lastAccessAt` 时间窗口判断，超过 30 天的 Token 即使签名正确也无法通过。

#### 场景 4：认证异常拦截

两个异常过滤器统一处理认证错误，重定向到登录页：

- [AuthExceptionFilter](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/filters/auth-exception.filter.ts) - OIDC 流程
- [GithubAuthExceptionFilter](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/filters/github-auth-exception.filter.ts) - GitHub 流程

```typescript
catch(exception: Error, host: ArgumentsHost) {
  const clientHost = this.configService.get(Env.CLIENT_HOST);
  response.redirect(
    `${clientHost}/login?error=${encodeURIComponent(exception.message)}`
  );
}
```

#### 场景 5：前端路由守卫

在 [PrivateRoute.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/PrivateRoute.tsx) 中：

```typescript
return authenticated ? (
  <>{childNode}</>
) : (
  <Redirect to={{ pathname: "/login", state: { from: location } }} />
);
```

如果本地没有 Token，`useAuthenticated()` 返回 false，自动跳转到登录页。

---

## 五、三者协作关系全景

### 5.1 完整登录时序（OIDC 流程）

```
用户浏览器                     后端 (amplication-server)            Auth0 IdP
    │                                 │                                 │
    │ 1. 点击 "Continue with SSO"     │                                 │
    │────────────────────────────────>│                                 │
    │                                 │                                 │
    │ 2. 302 重定向到 Auth0 授权页     │                                 │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │ 3. 用户输入凭证完成认证           │                                 │
    │──────────────────────────────────────────────────────────────────>│
    │                                 │                                 │
    │ 4. 携带 Authorization Code 回调 │                                 │
    │────────────────────────────────>│                                 │
    │                                 │ 5. Code → Token 交换            │
    │                                 │────────────────────────────────>│
    │                                 │                                 │
    │                                 │ 6. 返回 id_token/access_token    │
    │                                 │<────────────────────────────────│
    │                                 │                                 │
    │                                 │ 7. 校验 id_token，建立会话       │
    │                                 │    (写入 appSession Cookie)     │
    │                                 │                                 │
    │ 8. 302 重定向到 /auth/afterCallback                              │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │────────────────────────────────>│                                 │
    │                                 │ 9. 从 oidc.user 获取 profile    │
    │                                 │ 10. 查询/创建本地 User/Account  │
    │                                 │ 11. 签发 Amplication JWT        │
    │                                 │ 12. 写入 AJWT Cookie            │
    │ 13. 302 重定向到前端应用          │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │ 14. 读取 AJWT Cookie             │                                 │
    │ 15. 存入 LocalStorage            │                                 │
    │ 16. 删除 AJWT Cookie             │                                 │
    │ 17. 后续请求携带 Bearer JWT      │                                 │
    │────────────────────────────────>│                                 │
```

### 5.2 日常请求中的令牌校验流程

```
前端 (Apollo Client)                  后端 (NestJS)
    │                                    │
    │ 1. GraphQL Query/Mutation         │
    │    (authLink 附加 Bearer JWT)      │
    │───────────────────────────────────>│
    │                                    │
    │                                    │ 2. GqlAuthGuard.canActivate()
    │                                    │    └─> super.canActivate()
    │                                    │        └─> JwtStrategy.validate()
    │                                    │            ├─> 校验 JWT 签名
    │                                    │            ├─> [API Token] 检查 lastAccessAt
    │                                    │            │    └─> 通过则更新 lastAccessAt（续期）
    │                                    │            └─> 查询 User（确认未删除）
    │                                    │
    │                                    │ 3. authorizeContext() 权限检查
    │                                    │
    │                                    │ 4. Resolver 执行业务逻辑
    │                                    │
    │ 5. 返回数据 / 401 Unauthorized     │
    │<───────────────────────────────────│
```

### 5.3 令牌续期与会话失效的交互

| 事件 | User JWT 行为 | API Token 行为 | IdP 会话 | 说明 |
|-----|--------------|---------------|---------|------|
| 日常 API 调用 | 无变化 | lastAccessAt 更新（滑动续期） | 无变化 | User JWT 无过期机制 |
| 切换工作区 | 重新签发新 JWT（workspaceId 变更） | 不影响 | 无变化 | 旧 JWT 理论仍有效 |
| 30 天未使用 | 不受影响 | 自动失效 | 可能已过期 | API Token 有滑动窗口 |
| 用户主动登出 | 前端清除 LocalStorage | 不受影响（需手动删除） | IdP 侧同步登出 | 四层会话逐步失效 |
| 用户被软删除 | 下次请求时 JwtStrategy 校验失败 | 下次请求时校验失败 | 不受影响 | 数据库级别的最终防线 |
| JWT 密钥变更 | 全部失效 | 全部失效 | 不受影响 | 需要所有用户重新登录 |

### 5.4 设计要点与潜在风险

#### 设计优势

1. **最小权限 Cookie 传递**：JWT 仅通过临时 Cookie 从后端传递到前端，立即转移到 LocalStorage 并删除 Cookie，降低 CSRF 和 Cookie 窃取风险。

2. **OIDC 标准合规**：使用 Authorization Code Flow，`express-openid-connect` 处理所有协议细节（state 参数、nonce、PKCE 等）。

3. **API Token 滑动续期**：30 天无活动自动失效，每次有效访问重置计时，兼顾安全与用户体验。

4. **多层防御**：从前端路由守卫 → JWT 签名校验 → 用户存在性检查 → API Token 活跃性检查，形成纵深防御。

#### 潜在风险与改进建议

1. **User JWT 无过期时间**：`JwtModule.registerAsync` 中未设置 `signOptions.expiresIn`，导致 JWT 一旦签发永久有效。建议设置合理过期时间（如 24 小时）并实现 refresh token 机制。

2. **无 Token 撤销机制**：除了用户被删除外，没有主动撤销 User JWT 的机制。敏感操作（如修改密码、权限变更）后旧 Token 仍可使用。

3. **IdP Refresh Token 未充分利用**：`express-openid-connect` 存储了 IdP 的 refresh_token，但系统未利用它实现无感续期，当 IdP 会话过期后用户需重新登录。

4. **前端无 401 自动处理**：Apollo Client 未配置 `onError` 链路处理 401 错误，当 Token 失效时需依赖路由守卫或组件级错误处理，用户体验不够流畅。建议添加错误链路自动清除 Token 并跳转登录页。

---

## 六、核心文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| OIDC 中间件 | [oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts) |
| 认证控制器 | [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts) |
| 认证服务（核心） | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts) |
| JWT 策略 | [jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts) |
| JWT 载荷定义 | [jwt.dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/dto/jwt.dto.ts) |
| GitHub 策略 | [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts) |
| Auth0 服务 | [auth0.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/idp/auth0.service.ts) |
| 认证模块配置 | [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.module.ts) |
| GraphQL 认证守卫 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) |
| 认证异常过滤器 | [auth-exception.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/filters/auth-exception.filter.ts) |
| 前端令牌管理 | [authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts) |
| 前端 Cookie 工具 | [cookie.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/util/cookie.ts) |
| 前端 Apollo 客户端 | [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/graphqlClient.ts) |
| 前端路由守卫 | [PrivateRoute.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/PrivateRoute.tsx) |
| 环境变量定义 | [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/env.ts) |
