# Preview Artifact 与生成结果检查代码梳理

## 1. 核心概念与关系总览

整个流程涉及三个核心维度：**产物元数据**、**任务状态**、**用户确认流程**。它们的关系如下：

```
用户确认(PendingChange → Commit)
        ↓
创建 Build(Running, Waiting)
        ↓
Action/ActionStep 执行(Waiting → Running → Success/Failed)
        ↓
生成 Artifact (BUILD_ARTIFACTS_BASE_FOLDER/{resourceId}/{buildId})
        ↓
Push to Git / Preview PR
        ↓
Build 状态更新(Completed, Completed)
```

---

## 2. 产物元数据定义

### 2.1 Build - 主构建记录

**定义位置**: [Build.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/dto/Build.ts)

```typescript
class Build {
  id: string;                           // 构建唯一标识
  createdAt: Date;                      // 创建时间
  resourceId: string;                   // 关联资源ID
  userId: string;                       // 创建用户ID
  status: EnumBuildStatus;              // 构建状态（见3.1）
  gitStatus: EnumBuildGitStatus;        // Git同步状态（见3.2）
  archiveURI?: string;                  // 产物归档ZIP下载路径
  version: string;                      // 构建版本号（取自commitId后8位）
  message?: string;                     // 构建消息
  actionId: string;                     // 关联Action ID
  commitId: string;                     // 关联Commit ID
  codeGeneratorVersion?: string;        // 代码生成器版本
  linesOfCodeAdded?: number;            // 新增代码行数
  linesOfCodeDeleted?: number;          // 删除代码行数
  filesChanged?: number;                // 变更文件数
}
```

**archiveURI 解析逻辑**: [build.resolver.ts#L73-L76](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.resolver.ts#L73-L76)
```typescript
archiveURI(@Parent() build: Build): string {
  return `/generated-apps/${build.id}.zip`;
}
```

### 2.2 DSGResourceData - 代码生成资源数据

**定义位置**: [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts)

这是传递给 Data Service Generator 的完整资源数据，在构建执行前序列化到共享存储：

```typescript
class DSGResourceData {
  resourceType: EnumResourceType;         // 资源类型 (Service/Component等)
  resourceInfo?: AppInfo;                 // 应用信息
  buildId: string;                        // 构建ID
  entities?: Entity[];                    // 实体列表
  roles?: Role[];                         // 角色列表
  pluginInstallations: PluginInstallation[]; // 插件安装列表
  packages?: Package[];                   // 包列表
  moduleContainers?: ModuleContainer[];   // 模块容器
  moduleActions?: ModuleAction[];         // 模块动作
  moduleDtos?: ModuleDto[];               // 模块DTO
  resourceSettings?: ResourceSettings;    // 资源设置
  relations?: Relation[];                 // 实体关系
  serviceTopics?: ServiceTopics[];        // 服务主题
  topics?: Topic[];                       // 主题列表
  otherResources?: DSGResourceData[];     // 关联其他资源
}
```

**存储位置**: [build.service.ts#L541-L559](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L541-L559)
- Server 端保存路径: `{DSG_RESOURCE_DATA_BASE_FOLDER}/{buildId}/resource-data.json`

### 2.3 BuildPlugin - 构建插件信息

**关联位置**: [build.service.ts#L389-L421](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L389-L421)

```typescript
{
  buildId: string;
  packageName: string;
  packageVersion: string;
  requestedFullPackageName: string;  // 如 "@scope/plugin@1.0.0"
}
```

### 2.4 UserBuild - Schema Registry 用户构建事件

**定义位置**: [user-build/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/libs/schema-registry/src/lib/user-build/value.ts)

代码生成成功后通过 Kafka 发送的事件数据：

```typescript
class Value {
  buildId: string;
  commitId: string;
  commitMessage: string;
  projectId: string;
  resourceId: string;
  resourceName: string;
  workspaceId: string;
  projectName: string;
  externalId: string;        // 加密后的用户ID
  createdAt: number;         // 时间戳
  envBaseUrl: string;        // 客户端基础URL
}
```

### 2.5 产物物理存储路径

**代码位置**: [build-runner.service.ts#L372-L396](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L372-L396)

```
Job 工作目录:     {DSG_JOBS_BASE_FOLDER}/{jobBuildId}/{DSG_JOBS_CODE_FOLDER}
产物归档目录:     {BUILD_ARTIFACTS_BASE_FOLDER}/{resourceId}/{buildId}
DSG资源数据:      {DSG_RESOURCE_DATA_BASE_FOLDER}/{buildId}/resource-data.json
DSG Job资源数据:  {DSG_JOBS_BASE_FOLDER}/{jobBuildId}/{DSG_JOBS_RESOURCE_DATA_FILE}
```

---

## 3. 任务状态体系

### 3.1 EnumBuildStatus - 构建整体状态

**定义位置**: [EnumBuildStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/dto/EnumBuildStatus.ts)

```typescript
enum EnumBuildStatus {
  Running = "Running",     // 构建进行中
  Completed = "Completed", // 构建完成（代码生成+Git推送均成功）
  Failed = "Failed",       // 构建失败
  Invalid = "Invalid",     // 构建无效
  Unknown = "Unknown",     // 状态未知（需重新计算）
  Canceled = "Canceled",   // 构建已取消
}
```

**状态计算逻辑**: [build.resolver.ts#L78-L88](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.resolver.ts#L78-L88)
- 若构建"过期"（超过STALE_BUILD_HOURS=5小时），重新计算
- 若状态为 Unknown，重新计算

### 3.2 EnumBuildGitStatus - Git同步状态

**定义位置**: [EnumBuildGitStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/dto/EnumBuildGitStatus.ts)

```typescript
enum EnumBuildGitStatus {
  NotConnected = "NotConnected", // 未连接Git仓库
  Waiting = "Waiting",           // 等待Git同步
  Completed = "Completed",       // Git同步完成（PR创建成功）
  Failed = "Failed",             // Git同步失败
  Canceled = "Canceled",         // Git同步已取消
  Unknown = "Unknown",           // 状态未知
}
```

### 3.3 EnumActionStepStatus - 操作步骤状态

**定义位置**: [EnumActionStepStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/action/dto/EnumActionStepStatus.ts)

```typescript
enum EnumActionStepStatus {
  Waiting = "Waiting",   // 等待执行
  Running = "Running",   // 执行中
  Failed = "Failed",     // 执行失败
  Success = "Success",   // 执行成功
}
```

**完成逻辑**: [action.service.ts#L97-L111](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/action/action.service.ts#L97-L111)
- 步骤完成时会设置 `status` 为 Success/Failed，并记录 `completedAt` 时间

### 3.4 EnumJobStatus - Build Manager Job状态

**定义位置**: [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/types.ts)

```typescript
enum EnumJobStatus {
  InProgress = "in-progress",  // Job进行中
  Success = "success",         // Job成功
  Failure = "failure",         // Job失败
}
```

**Job ID 格式**: `{buildId}-server` 或 `{buildId}-admin-ui`（拆分构建时）
**聚合逻辑**: [build-job-handler.service.ts#L101-L122](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L101-L122)
- 所有子Job成功 → 整体 Success
- 任一子Job失败 → 整体 Failure
- 任一子Job进行中 → 整体 InProgress

状态存储在 Redis 中，Key 为 `buildId`，Value 为各子Job的状态映射。

### 3.5 EnumUserActionStatus - 用户操作状态

**定义位置**: [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/userAction/types.ts)

```typescript
enum EnumUserActionStatus {
  Running = "Running",     // 进行中
  Completed = "Completed", // 完成
  Failed = "Failed",       // 失败
  Invalid = "Invalid",     // 无效（无steps）
}
```

**状态评估逻辑**: [userAction.service.ts#L69-L96](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L69-L96)
- 所有step Success → Completed
- 任一step Failed → Failed
- 否则 → Running

---

## 4. 用户确认流程详解

### 4.1 PendingChange - 待确认变更

**定义位置**: [PendingChange.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/resource/dto/PendingChange.ts)

```typescript
class PendingChange {
  action: EnumPendingChangeAction;           // Create/Update/Delete
  originType: EnumPendingChangeOriginType;   // Block/Entity
  originId: string;                          // 变更来源ID
  origin: Entity | Block;                    // 变更来源对象
  versionNumber: number;                     // 版本号
  resource: Resource;                        // 所属资源
}
```

**变更来源**:
- Entity 变更: [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/entity/entity.service.ts)
- Block 变更: [block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/block/block.service.ts)

### 4.2 Commit - 提交（用户确认入口）

**核心数据模型**（GraphQL Schema）: [models.ts#L388-L396](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/libs/util/code-gen-types/src/models.ts#L388-L396)

```typescript
type Commit = {
  id: string;
  message: string;                   // 提交消息
  createdAt: DateTime;
  userId: string;                    // 提交用户
  changes?: PendingChange[];         // 本次提交包含的变更
  builds?: Build[];                  // 本次提交触发的构建
}
```

**Commit创建输入**: [models.ts#L406-L417](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/libs/util/code-gen-types/src/models.ts#L406-L417)
```typescript
type CommitCreateInput = {
  message: string;
  project: WhereParentIdInput;
  commitStrategy?: EnumCommitStrategy;  // All / AllWithPendingChanges / Specific
  resourceIds?: string[];               // strategy=Specific时指定
  resourceTypeGroup: EnumResourceTypeGroup;
  bypassLimitations?: boolean;
}
```

### 4.3 Build 创建流程

**代码位置**: [build.service.ts#L268-L352](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352)

```
1. 创建 Build 记录
   ├── status: Running
   ├── gitStatus: Waiting
   ├── version: commitId后8位
   ├── 关联 entityVersions (最新版本)
   └── 创建 Action + 初始步骤 ADD_TO_QUEUE (Success)

2. 检查资源类型（仅Service和Component触发生成）

3. 检查私有插件
   ├── 有私有插件 → 先执行 downloadPrivatePlugins
   └── 无私有插件 → 直接执行 generate
```

**初始步骤创建**: [build.service.ts#L179-L210](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L179-L210)
- 步骤名: `ADD_TO_QUEUE`
- 状态: `Success`（立即完成，表示已加入队列）

### 4.4 Build 执行步骤 (Action Steps)

每个 Build 关联一个 Action，Action 包含多个 ActionStep，标准步骤序列：

| 步骤名 | 消息 | 触发条件 |
|--------|------|----------|
| `ADD_TO_QUEUE` | Adding task to queue | 创建Build时自动创建 |
| `DOWNLOAD_PRIVATE_PLUGINS` | Downloading private plugins | 资源有私有插件时 |
| `GENERATE_APPLICATION` | Generating Application | 必有 |
| `PUSH_TO_GIT_PROVIDER` | Push changes to {provider} | 代码生成成功后 |

**步骤执行引擎**: [action.service.ts#L275-L295](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295)
```typescript
async run<T>(
  actionId, stepName, message,
  stepFunction,
  leaveStepOpenAfterSuccessfulExecution = false
) {
  创建Step (Running)
  try {
    result = await stepFunction(step)
    if (!leaveStepOpen) 标记Step Success
    return result
  } catch {
    记录错误日志
    标记Step Failed
    抛出异常
  }
}
```

### 4.5 代码生成流程

**代码位置**: [build.service.ts#L568-L618](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

```
1. 获取DSGResourceData（完整资源数据）
2. 保存到共享存储: {DSG_RESOURCE_DATA_BASE_FOLDER}/{buildId}/resource-data.json
3. 发送Kafka消息: CODE_GENERATION_REQUEST_TOPIC
   ├── key: null
   └── value: { resourceId, buildId }
4. Step保持Running状态（leaveStepOpen=true）
```

**Build Manager 接收处理**: [build-runner.service.ts#L109-L159](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159)
```
1. 从共享存储读取DSGResourceData
2. 决定代码生成器版本
3. 按条件拆分Job（Server + AdminUI）
4. 每个Job调用DSG Runner执行
5. Job完成后复制产物到 BUILD_ARTIFACTS_BASE_FOLDER/{resourceId}/{buildId}
6. 所有Job成功后发送 CODE_GENERATION_SUCCESS_TOPIC
```

### 4.6 代码生成成功回调

**代码位置**: [build.service.ts#L450-L495](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L450-L495)

```
1. 标记 GENERATE_APPLICATION 步骤为 Success
2. 发送 USER_BUILD_TOPIC Kafka事件（含产物元数据）
3. 触发 saveToGitProvider（推送代码到Git）
```

### 4.7 Push to Git / Preview PR 流程

**代码位置**: [build.service.ts#L1130-L1342](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L1130-L1342)

**两种模式**:

#### 模式A: Demo/Preview 仓库 (`useDemoRepo=true`)
- **触发条件**: `project.useDemoRepo === true`
- **PR标题**: `"Preview PR from Amplication"`
- **PR内容**: 使用 `PREVIEW_PR_BODY` 模板，包含引导用户连接自己仓库的说明
- **Git配置**: 使用配置的 GITHUB_DEMO_REPO_ORGANIZATION_NAME 和 demoRepoName
- **模板定义**: [build.service.ts#L210](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L210)

#### 模式B: 用户自有仓库
- **触发条件**: 资源已配置 gitRepository
- **PR标题**: `"{commit.message} (Amplication build {buildId后8位})"`
- **PR内容**: 包含Build链接和提交消息
- **Git配置**: 使用用户配置的仓库信息

#### 公共流程
```
1. 创建 PUSH_TO_GIT_PROVIDER 步骤 (Running)
2. 组装 CreatePrRequest
   ├── git组织/仓库名
   ├── 基础分支
   ├── commit标题/内容
   ├── newBuildId / oldBuildId（用于增量diff）
   ├── gitResourceMeta（serverPath/adminUIPath）
   ├── isBranchPerResource
   └── overrideCustomizableFilesInGit
3. 发送Kafka消息: CREATE_PR_REQUEST_TOPIC
```

### 4.8 Git推送成功/失败回调

**成功回调**: [build.service.ts#L851-L915](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L851-L915)
```
1. 获取本Commit的变更列表(PendingChange)
2. 若有变更，更新 Build 统计: linesOfCodeAdded/Deleted, filesChanged
3. 记录日志: GitHub PR URL, diffStat
4. 标记 PUSH_TO_GIT_PROVIDER 步骤 Success
5. 更新 Build 状态: status=Completed, gitStatus=Completed
6. 上报计费: BillingFeature.CodePushToGit
```

**失败回调**: [build.service.ts#L917-L968](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L917-L968)
```
1. 记录资源同步错误消息
2. 标记 PUSH_TO_GIT_PROVIDER 步骤 Failed
3. 更新 Build 状态: status=Failed, gitStatus=Failed
4. 上报分析事件: EnumEventType.GitSyncError
```

### 4.9 代码生成失败回调

**代码位置**: [build.service.ts#L511-L534](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts#L511-L534)
```
1. 写入错误日志
2. 标记 GENERATE_APPLICATION 步骤 Failed
3. 更新 Build 状态: status=Failed, gitStatus=Canceled
```

---

## 5. 完整状态流转图

### 5.1 Build 状态流转

```
创建Build
    │
    ▼
┌─────────────────────────────────────────┐
│  status: Running                         │
│  gitStatus: Waiting                      │
└────────────────┬────────────────────────┘
                 │
         代码生成成功?
            /        \
          否          是
          │           │
          ▼           ▼
┌────────────┐  ┌───────────────────┐
│status:     │  │  开始Push to Git   │
│Failed      │  └─────────┬─────────┘
│gitStatus:  │            │
│Canceled    │      Git推送成功?
└────────────┘         /     \
                      否      是
                      │       │
                      ▼       ▼
              ┌──────────┐ ┌──────────────┐
              │status:   │ │status:       │
              │Failed    │ │Completed     │
              │gitStatus:│ │gitStatus:    │
              │Failed    │ │Completed     │
              └──────────┘ └──────────────┘
```

### 5.2 ActionStep 状态流转

```
                     createStep()
                         │
                         ▼
                  ┌───────────┐
                  │  Waiting  │
                  └─────┬─────┘
                        │
                  stepFunction执行
                        │
                        ▼
                  ┌───────────┐
                  │  Running  │
                  └─────┬─────┘
                        │
              执行成功? /     \ 执行失败?
                    是 /       \ 否
                      ▼         ▼
               ┌─────────┐  ┌────────┐
               │ Success │  │ Failed │
               └─────────┘  └────────┘
               (设置completedAt)
```

---

## 6. 关键服务文件索引

| 服务 | 文件路径 | 核心职责 |
|------|----------|----------|
| BuildService | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.service.ts) | 构建创建、代码生成触发、Git推送、状态更新 |
| ActionService | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/action/action.service.ts) | 步骤创建、执行、状态管理、日志记录 |
| CommitService | [commit.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/commit/commit.service.ts) | 提交查询、变更列表获取 |
| UserActionService | [userAction.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts) | 用户操作创建、状态评估、元数据更新 |
| BuildRunnerService | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | Job执行、产物复制、成功/失败事件发送 |
| BuildJobsHandlerService | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | Build拆分Job、Job状态聚合（Redis） |
| BuildResolver | [build.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/build/build.resolver.ts) | GraphQL查询接口、状态动态计算、archiveURI解析 |
| CommitResolver | [commit.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/51-amplication/packages/amplication-server/src/core/commit/commit.resolver.ts) | Commit/Changes GraphQL查询接口 |

---

## 7. 关键 Kafka Topic

| Topic | 生产者 | 消费者 | 说明 |
|-------|--------|--------|------|
| `CODE_GENERATION_REQUEST_TOPIC` | BuildService | BuildRunnerService | 请求代码生成 |
| `CODE_GENERATION_SUCCESS_TOPIC` | BuildRunnerService | BuildService | 代码生成成功 |
| `CODE_GENERATION_FAILURE_TOPIC` | BuildRunnerService | BuildService | 代码生成失败 |
| `CREATE_PR_REQUEST_TOPIC` | BuildService | GitSyncManager | 请求创建PR |
| `USER_BUILD_TOPIC` | BuildService | 下游服务 | 用户构建事件（产物通知） |
| `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | BuildService | 插件服务 | 请求下载私有插件 |
