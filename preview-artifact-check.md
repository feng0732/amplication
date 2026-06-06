# Preview Artifact 与生成结果检查链路梳理

本文档从代码实现角度，端到端梳理 Preview Artifact 的完整检查链路，包含前端确认入口、提交策略、后端 Build 创建、生成成功/失败回调、Preview PR 生成与结果状态展示六个阶段。

---

## 总览：端到端流程

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                前端 (amplication-client)                              │
│                                                                                      │
│  [PendingChangesPage] → [Commit组件] → [提交策略选择] → [COMMIT_CHANGES mutation]    │
│         ↑                    ↑                                                       │
│         │                    │ useCommits hook                                       │
│         │                    │                                                       │
│  [LastCommit状态展示] ← [轮询 GET_LAST_COMMIT] ← [useBuildWatchStatus轮询GET_BUILD] │
│         │                    │                                                       │
│         ▼                    ▼                                                       │
│  [CommitsPage]          [BuildPage + ActionLog]                                      │
│  [CommitPage]           [BuildGitLink → PR链接]                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                      │ GraphQL
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                               后端 (amplication-server)                               │
│                                                                                      │
│  [ProjectResolver.commit()] → [ProjectService.commit()]                              │
│                                     │                                                │
│                                     ▼                                                │
│                            1. 计费限制校验                                            │
│                            2. 获取Changed Entities/Blocks                             │
│                            3. 创建Commit记录                                          │
│                            4. 创建EntityVersion/BlockVersion                          │
│                            5. 释放Entity/Block锁                                      │
│                            6. 根据commitStrategy筛选resources                         │
│                            7. 级联关联resource筛选                                    │
│                            8. 为每个resource调用BuildService.create()                 │
│                                     │                                                │
│                                     ▼                                                │
│                           [BuildService.create()]                                     │
│                                     │                                                │
│                                     ▼                                                │
│                         status=Running, gitStatus=Waiting                            │
│                         创建Action + ADD_TO_QUEUE步骤                                 │
│                                     │                                                │
│                              下载私有插件？ ──是──► DOWNLOAD_PRIVATE_PLUGINS步骤       │
│                                     │否                                              │
│                                     ▼                                                │
│                         [BuildService.generate()]                                     │
│                                     │                                                │
│                                     ▼                                                │
│                      序列化DSGResourceData到共享存储                                   │
│                      发送Kafka: CODE_GENERATION_REQUEST_TOPIC                         │
│                      GENERATE_APPLICATION步骤保持Running                               │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                      │ Kafka
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           Build Manager (amplication-build-manager)                   │
│                                                                                      │
│   [BuildRunnerService] → 读取DSGResourceData → 拆分子Job(server/admin-ui)            │
│                               │                                                      │
│                               ▼                                                      │
│                    每个子Job执行DSG → 产物复制到 BUILD_ARTIFACTS_BASE_FOLDER          │
│                               │                                                      │
│                     全部子Job成功？                                                    │
│                          /        \                                                   │
│                        是          否                                                  │
│                        ▼           ▼                                                  │
│          CODE_GENERATION_SUCCESS_TOPIC   CODE_GENERATION_FAILURE_TOPIC                │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                      │ Kafka
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              后端回调 (amplication-server)                             │
│                                                                                      │
│  ┌─ 成功回调: handleCodeGenerationSuccess()                                           │
│  │    1. GENERATE_APPLICATION步骤 → Success                                            │
│  │    2. 发送Kafka: USER_BUILD_TOPIC                                                   │
│  │    3. 调用saveToGitProvider()                                                       │
│  │         │                                                                          │
│  │         ▼                                                                          │
│  │   project.useDemoRepo ?                                                            │
│  │       /         \                                                                  │
│  │     是            否                                                                │
│  │     ▼             ▼                                                                │
│  │  Preview PR    用户仓库PR                                                           │
│  │  标题固定        标题: "{commit.message} (Amplication build {id})"                  │
│  │  PREVIEW_PR_BODY 包含Build链接                                                      │
│  │         │                                                                          │
│  │         ▼                                                                          │
│  │   创建PUSH_TO_GIT_PROVIDER步骤(Running)                                             │
│  │   发送Kafka: CREATE_PR_REQUEST_TOPIC                                                │
│  │         │                                                                          │
│  │         ▼                                                                          │
│  │   PR创建成功回调 → PUSH步骤Success                                                  │
│  │                   → Build.status=Completed                                         │
│  │                   → Build.gitStatus=Completed                                      │
│  │                   → 更新代码行数统计                                                │
│  │                                                                                    │
│  │   PR创建失败回调 → PUSH步骤Failed                                                   │
│  │                   → Build.status=Failed                                            │
│  │                   → Build.gitStatus=Failed                                         │
│  │                                                                                    │
│  └─ 失败回调: handleCodeGenerationFailure()                                           │
│       1. GENERATE_APPLICATION步骤 → Failed                                             │
│       2. Build.status=Failed                                                           │
│       3. Build.gitStatus=Canceled                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 阶段一：前端确认入口

### 1.1 Pending Changes 页面

**文件**: `packages/amplication-client/src/VersionControl/PendingChangesPage.tsx`

用户查看所有待确认变更的入口页面：

- 通过 `usePendingChanges()` hook 获取按资源分组的变更列表
- 每个变更使用 `PendingChangeWithCompare` 组件展示差异对比
- 变更类型包括 Entity 和 Block 两种
- 支持 Platform 控制台和 Services 两种资源类型分组

```tsx
const PendingChangesPage = ({ match }: Props) => {
  const { pendingChangesByResource } = usePendingChanges(
    currentProject,
    isPlatformConsole
      ? EnumResourceTypeGroup.Platform
      : EnumResourceTypeGroup.Services
  );
  // 按资源分组展示变更列表
};
```

### 1.2 Commit 组件

**文件**: `packages/amplication-client/src/VersionControl/Commit.tsx`

核心提交组件，提供用户输入提交消息和选择提交策略的交互：

- 使用 Formik 管理表单状态
- 支持 `Ctrl+Enter` 快捷键提交
- 提供下拉菜单选择三种提交策略（见阶段二）
- 选择 Specific 策略时弹窗让用户选择具体服务

关键交互逻辑：
```tsx
const handleCommit = (message, commitStrategy, selectedServiceId?) => {
  commitChanges({
    message,
    project: { connect: { id: currentProject?.id } },
    bypassLimitations,
    commitStrategy,
    resourceIds: selectedServiceId ? [selectedServiceId] : null,
    resourceTypeGroup,
  });
};
```

### 1.3 Commit Button

**文件**: `packages/amplication-client/src/VersionControl/CommitButton.tsx`

按钮组件，根据上下文展示不同样式：

- `CommitBtnType.Button`: 常规按钮，Services 显示 "Generate the code"，Platform 显示 "Publish Changes"
- `CommitBtnType.JumboButton`: 大按钮，用于引导页 "Generate the code for my new architecture"

按钮点击时的默认策略选择逻辑：
```typescript
const strategy = hasPendingChanges
  ? EnumCommitStrategy.AllWithPendingChanges   // 有变更时默认只提交有变更的
  : hasMultipleServices && onCommitSpecificService
  ? EnumCommitStrategy.Specific                 // 多服务且无变更时让用户选择
  : EnumCommitStrategy.All;                     // 单服务时全量提交
```

---

## 阶段二：提交策略

### 2.1 EnumCommitStrategy 定义

**文件**: `packages/amplication-server/src/core/resource/dto/EnumCommitStrategy.ts`

```typescript
enum EnumCommitStrategy {
  All = "all",                               // 所有资源
  AllWithPendingChanges = "allWithPendingChanges",  // 仅包含有变更的资源（默认）
  Specific = "specific",                     // 指定资源ID
}
```

### 2.2 前端策略选项

**文件**: `packages/amplication-client/src/VersionControl/Commit.tsx`

```typescript
const COMMIT_STRATEGY_OPTIONS = [
  { strategyType: EnumCommitStrategy.All, label: "All services" },
  { strategyType: EnumCommitStrategy.AllWithPendingChanges, label: "Pending changes (default)" },
  { strategyType: EnumCommitStrategy.Specific, label: "Specific service" },
];
```

- `AllWithPendingChanges` 选项在无 pending changes 时不显示
- 选择 `Specific` 时会弹出服务选择对话框

### 2.3 CommitCreateInput 结构

**文件**: `packages/amplication-server/src/core/resource/dto/CommitCreateInput.ts`

```typescript
class CommitCreateInput {
  message!: string;                                    // 提交消息
  project!: WhereParentIdInput;                        // 关联项目
  resourceTypeGroup!: EnumResourceTypeGroup;           // Services / Platform
  bypassLimitations? = false;                          // 是否绕过计费限制
  commitStrategy?: EnumCommitStrategy;                 // 提交策略
  resourceIds?: string[];                              // strategy=Specific时的资源ID列表
  resourceVersions?: CommitResourceVersionCreateInput[]; // Platform模式下的版本号
  user!: WhereParentIdInput;                           // 提交用户（从上下文注入，不暴露GraphQL）
}
```

### 2.4 后端策略应用

**文件**: `packages/amplication-server/src/core/project/project.service.ts` (commit方法)

```typescript
// 策略1: AllWithPendingChanges - 只保留有实际变更的资源
if (args.data.commitStrategy === EnumCommitStrategy.AllWithPendingChanges) {
  resourcesToBuild = resources.filter((resource) => {
    return (
      changedEntities.some((c) => c.resource.id === resource.id) ||
      changedBlocks.some((c) => c.resource.id === resource.id)
    );
  });
}

// 策略2: Specific - 只保留指定的资源ID
if (args.data.commitStrategy === EnumCommitStrategy.Specific) {
  if (!args.data.resourceIds?.length) {
    throw new Error("resourceIds are required for specific commit strategy");
  }
  resourcesToBuild = resources.filter((r) =>
    args.data.resourceIds.includes(r.id)
  );
}

// 策略3: All - 使用全部资源（默认）

// Services模式下额外处理：级联关联资源
if (resourceTypeGroup === EnumResourceTypeGroup.Services) {
  const cascadingBuildableResourceIds =
    await this.relationService.getCascadingBuildableResourceIds(resourceIds);
  // 对每个级联后的resourceId创建Build
}
```

---

## 阶段三：后端创建 Build

### 3.1 GraphQL 入口

**文件**: `packages/amplication-server/src/core/project/project.resolver.ts`

```typescript
@Mutation(() => Commit, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.ProjectId, "data.project.connect.id")
@InjectContextValue(InjectableOriginParameter.UserId, "data.user.connect.id")
async commit(
  @UserEntity() currentUser: User,
  @Args() args: CreateCommitArgs
): Promise<Commit | null> {
  return await this.projectService.commit(args, currentUser);
}
```

### 3.2 ProjectService.commit() 完整流程

**文件**: `packages/amplication-server/src/core/project/project.service.ts`

```
1. 计费限制校验
   ├── shouldBlockBuild(userId) → 检查BlockBuild权限
   ├── billingService.isBillingEnabled
   │   ├── calculateMeteredUsage → 计算计量用量
   │   ├── billingService.resetUsage → 重置用量
   │   ├── validateSubscriptionPlanLimitationsForWorkspace → 订阅计划限制
   │   └── validateProjectLimitations → 项目级限制（实体数等）
   └── BillingLimitationError → 抛出异常终止流程

2. 获取变更列表
   ├── entityService.getChangedEntities() → 变更的Entity列表
   └── blockService.getChangedBlocks() → 变更的Block列表

3. 创建Commit记录
   ├── prisma.commit.create({ message, project, user })
   └── billingService.reportUsage(CodeGenerationBuilds)

4. 创建版本 & 释放锁（并行）
   ├── 对每个Changed Entity:
   │   ├── entityService.createVersion({ commit, entity })
   │   └── entityService.releaseLock(entityId)
   └── 对每个Changed Block:
       ├── blockService.createVersion({ commit, block })
       └── blockService.releaseLock(blockId)

5. 根据commitStrategy筛选resourcesToBuild
   ├── All → 全部资源
   ├── AllWithPendingChanges → 仅含变更的资源
   └── Specific → 指定resourceIds

6. Services模式: 级联关联 + 创建Build
   ├── relationService.getCascadingBuildableResourceIds() → 包含依赖资源
   └── 对每个resourceId调用 buildService.create()

7. Platform模式: 创建ResourceVersion
   └── 对每个resource调用 resourceVersionService.create()
```

### 3.3 BuildService.create()

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async create(args: CreateOneBuildArgs): Promise<Build> {
  // 1. 创建Action和初始步骤
  const action = await this.actionService.create(
    createInitialAction(message, version)
    // 初始步骤: ADD_TO_QUEUE (状态=Success, 含3条Info日志)
  );

  // 2. 创建Build记录
  const build = await this.prisma.build.create({
    data: {
      resource: ...connect,
      commit: ...connect,
      createdBy: ...connect,
      status: EnumBuildStatus.Running,       // 初始状态
      gitStatus: EnumBuildGitStatus.Waiting, // 初始Git状态
      version,                               // commitId后8位
      message,
      action: { connect: { id: action.id } },
    },
  });

  // 3. 关联BuildPlugin记录
  await Promise.all(
    pluginInstallations.map((plugin) =>
      this.prisma.buildPlugin.create({
        data: {
          build: { connect: { id: build.id } },
          packageName,
          packageVersion,
          requestedFullPackageName,
        },
      })
    )
  );

  // 4. 仅Service/Component类型触发生成流程
  if (resourceType === Service || resourceType === Component) {
    const hasPrivatePlugins = pluginInstallations.some(...);
    if (hasPrivatePlugins) {
      await this.downloadPrivatePlugins(build.id, resourceId);
      // 私有插件下载成功后自动调用 this.generate()
    } else {
      await this.generate(build.id, resourceId);
    }
  }

  return build;
}
```

### 3.4 初始 Action & Steps

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
function createInitialAction(message: string, version: string): Action {
  return {
    steps: {
      create: [
        {
          name: "ADD_TO_QUEUE",
          message: "Adding task to queue",
          status: EnumActionStepStatus.Success,  // 立即标记成功
          logs: {
            create: [
              { level: Info, message: "Create build generation task" },
              { level: Info, message: `Build version: ${version}` },
              { level: Info, message: `Build message: ${message}` },
            ],
          },
        },
      ],
    },
  };
}
```

---

## 阶段四：生成成功或失败回调

### 4.1 代码生成触发

**文件**: `packages/amplication-server/src/core/build/build.service.ts` (generate方法)

```
generate(buildId, resourceId):
  1. 组装DSGResourceData（完整资源快照）
  2. 序列化为JSON
  3. 保存到共享存储: {DSG_RESOURCE_DATA_BASE_FOLDER}/{buildId}/resource-data.json
  4. ActionService.run():
     - 创建步骤 GENERATE_APPLICATION (Running)
     - 发送Kafka消息 CODE_GENERATION_REQUEST_TOPIC
       key: null
       value: { resourceId, buildId }
     - leaveStepOpen = true (保持Running状态等待Kafka回调)
```

### 4.2 Build Manager 处理

**文件**: `packages/amplication-build-manager/src/build-runner/build-runner.service.ts`

```
收到CODE_GENERATION_REQUEST_TOPIC消息:
  1. 从共享存储读取DSGResourceData
  2. 决定代码生成器版本
  3. 判断是否拆分Job:
     ├── hasAdminUiSections → 拆分为 server + admin-ui 两个子Job
     └── 否则 → 仅 server 一个子Job
  4. 每个子Job:
     ├── 准备工作目录: {DSG_JOBS_BASE_FOLDER}/{jobBuildId}/code
     ├── 写入DSGResourceData到工作目录
     ├── 调用DSG Runner执行代码生成
     └── 成功后复制产物: {DSG_JOBS_BASE_FOLDER}/{jobBuildId}/code/**
                        → {BUILD_ARTIFACTS_BASE_FOLDER}/{resourceId}/{buildId}
  5. 子Job状态写入Redis (key=buildId)
  6. 聚合所有子Job状态:
     ├── 全部Success → 发送 CODE_GENERATION_SUCCESS_TOPIC
     └── 任一Failure → 发送 CODE_GENERATION_FAILURE_TOPIC
```

### 4.3 成功回调 - handleCodeGenerationSuccess()

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async handleCodeGenerationSuccess(resourceId: string, buildId: string) {
  // 1. 标记GENERATE_APPLICATION步骤成功
  await this.actionService.completeStep(
    build.actionId,
    "GENERATE_APPLICATION",
    EnumActionStepStatus.Success
  );

  // 2. 发送USER_BUILD_TOPIC事件（产物通知下游服务）
  await this.kafkaService.emitMessage(USER_BUILD_TOPIC, {
    value: {
      buildId,
      commitId,
      commitMessage,
      projectId,
      resourceId,
      resourceName,
      workspaceId,
      projectName,
      externalId,  // 加密用户ID
      createdAt,
      envBaseUrl,
    },
  });

  // 3. 触发Git推送流程
  await this.saveToGitProvider(buildId);
}
```

### 4.4 失败回调 - handleCodeGenerationFailure()

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async handleCodeGenerationFailure(buildId: string, error: string) {
  // 1. 记录错误日志到ActionStep
  await this.actionService.createLog(
    build.actionId,
    "GENERATE_APPLICATION",
    {
      level: EnumActionLogLevel.Error,
      message: error,
      meta: {},
    }
  );

  // 2. 标记GENERATE_APPLICATION步骤失败
  await this.actionService.completeStep(
    build.actionId,
    "GENERATE_APPLICATION",
    EnumActionStepStatus.Failed
  );

  // 3. 更新Build状态
  await this.prisma.build.update({
    where: { id: buildId },
    data: {
      status: EnumBuildStatus.Failed,
      gitStatus: EnumBuildGitStatus.Canceled, // Git推送取消
    },
  });
}
```

---

## 阶段五：Preview PR 生成

### 5.1 saveToGitProvider() - Git推送入口

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async saveToGitProvider(buildId: string) {
  // 1. 创建PUSH_TO_GIT_PROVIDER步骤（Running）
  await this.actionService.createStep(build.actionId, {
    name: `PUSH_TO_GIT_${gitProvider}`,  // e.g. PUSH_TO_GIT_Github
    message: `Push changes to ${gitProvider}`,
    status: EnumActionStepStatus.Running,
  });

  // 2. 判断是否使用Demo/Preview仓库
  if (project.useDemoRepo) {
    // ==== Preview PR 模式 ====
    const organizationName = config.get(GITHUB_DEMO_REPO_ORGANIZATION_NAME);
    const installationId = config.get(GITHUB_DEMO_REPO_INSTALLATION_ID);
    const buildLink = `${clientHost}/${workspaceId}/${projectId}/${resourceId}/git-sync`;

    const commitBody = PREVIEW_PR_BODY.replace("[link]", buildLink);

    gitSettings = {
      gitOrganizationName: organizationName,
      gitRepositoryName: project.demoRepoName,
      gitProvider: EnumGitProvider.Github,
      gitProviderProperties: { installationId },
      commit: {
        title: "Preview PR from Amplication",
        body: commitBody,
      },
    };
    kafkaEventKey = project.demoRepoName;
  } else {
    // ==== 用户自有仓库模式 ====
    const resourceGitRepo = resourceService.gitRepository(resourceId);
    gitSettings = {
      gitOrganizationName: resourceGitRepo.gitOrganizationName,
      gitRepositoryName: resourceGitRepo.gitRepositoryName,
      baseBranchName: resourceGitRepo.baseBranchName,
      gitProvider: resourceGitRepo.gitProvider,
      isBranchPerResource: resourceGitRepo.isBranchPerResource,
      gitResourceMeta: {
        serverPath: resourceGitRepo.gitRepository?.serverPath,
        adminUIPath: resourceGitRepo.gitRepository?.adminUIPath,
      },
      overrideCustomizableFilesInGit: ...,
      commit: {
        title: `${commit.message} (Amplication build ${truncatedBuildId})`,
        body: `Build link: ${buildLink}\n\n${commit.message}`,
      },
    };
  }

  // 3. 发送Kafka消息创建PR
  await this.kafkaService.emitMessage(CREATE_PR_REQUEST_TOPIC, {
    key: kafkaEventKey,
    value: { ...createPrRequest, gitSettings },
  });
}
```

### 5.2 PREVIEW_PR_BODY 模板

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```
Welcome to your first sync with Amplication's Preview Repo! 🚀

You've taken the first step in supercharging your development.
This Preview Repo is a sandbox for you to see what Amplication can do.

Remember, by connecting to your own repository, you'll have even more power -
like customizing the code to fit your needs.

Now, head back to Amplication, connect to your own repo and keep building!
Define data entities, set up roles, and extend your service's functionality
with our versatile plugin system. The possibilities are endless.

[{buildLink}]

Thank you, and let's build something amazing together! 🚀
```

### 5.3 PR 创建成功回调

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async handleCreatePrSuccess(buildId: string, githubUrl: string, diffStat: DiffStat) {
  // 1. 记录PR URL到步骤日志
  await this.actionService.createLog(
    build.actionId,
    `PUSH_TO_GIT_${provider}`,
    {
      level: Info,
      message: `PR was successfully created at ${githubUrl}`,
      meta: { githubUrl },  // 前端通过meta.githubUrl提取PR链接
    }
  );

  // 2. 更新Build代码统计（仅当有实际Entity/Block变更时）
  if (pendingChanges.length > 0) {
    await this.prisma.build.update({
      where: { id: buildId },
      data: {
        linesOfCodeAdded: diffStat.insertions,
        linesOfCodeDeleted: diffStat.deletions,
        filesChanged: diffStat.filesChanged,
      },
    });
  }

  // 3. 标记PUSH步骤成功
  await this.actionService.completeStep(
    build.actionId,
    `PUSH_TO_GIT_${provider}`,
    EnumActionStepStatus.Success
  );

  // 4. 更新Build最终状态
  await this.prisma.build.update({
    where: { id: buildId },
    data: {
      status: EnumBuildStatus.Completed,
      gitStatus: EnumBuildGitStatus.Completed,
    },
  });

  // 5. 上报计费事件
  await this.billingService.reportUsage(workspaceId, BillingFeature.CodePushToGit);
}
```

### 5.4 PR 创建失败回调

**文件**: `packages/amplication-server/src/core/build/build.service.ts`

```typescript
async handleCreatePrFailure(buildId: string, error: string) {
  // 1. 记录资源同步错误
  await this.resourceSyncService.setResourceSyncError(resourceId, error);

  // 2. 标记PUSH步骤失败
  await this.actionService.completeStep(
    build.actionId,
    `PUSH_TO_GIT_${provider}`,
    EnumActionStepStatus.Failed
  );

  // 3. 更新Build最终状态
  await this.prisma.build.update({
    where: { id: buildId },
    data: {
      status: EnumBuildStatus.Failed,
      gitStatus: EnumBuildGitStatus.Failed,
    },
  });

  // 4. 上报分析事件
  await this.analytics.trackWithContext({
    event: EnumEventType.GitSyncError,
    properties: { error, buildId, resourceId },
  });
}
```

---

## 阶段六：结果状态展示

### 6.1 前端轮询机制

#### useCommits Hook - Commit 级别轮询

**文件**: `packages/amplication-client/src/VersionControl/hooks/useCommits.ts`

```typescript
// 轮询间隔: 1000ms (仅当有Running状态的Build时)
const POLL_INTERVAL = 1000;

// GET_LAST_COMMIT查询包含完整Commit+Builds+Action+Steps+Logs
useEffect(() => {
  const hasRunningBuilds = lastCommit?.builds?.some(
    (b) => b.status === EnumBuildStatus.Running
  );
  if (hasRunningBuilds) {
    startPolling(POLL_INTERVAL);
  } else {
    stopPolling();
  }
}, [lastCommit]);
```

GraphQL查询: `COMMIT_FIELDS_FRAGMENT`
```graphql
fragment CommitFields on Commit {
  id message createdAt
  user { account { firstName lastName } }
  changes { originId action originType versionNumber origin resource }
  builds {
    id version status gitStatus archiveURI
    resource { id name resourceType codeGenerator }
    action {
      steps {
        id name message status completedAt
        logs { id createdAt message meta level }
      }
    }
  }
}
```

#### useBuildWatchStatus - Build 级别轮询

**文件**: `packages/amplication-client/src/VersionControl/useBuildWatchStatus.tsx`

```typescript
// 轮询间隔: 5000ms (仅当Build=Running时)
const POLL_INTERVAL = 5000;

function shouldReload(build: Build): boolean {
  return build?.status === EnumBuildStatus.Running;
}

// GET_BUILD查询包含Build+Action+Steps+Logs
useEffect(() => {
  if (!shouldReload(data?.build)) {
    stopPolling();
  } else {
    startPolling(POLL_INTERVAL);
  }
  // 实时更新commitUtils中的build状态
  data && commitUtils.updateBuildStatus(data.build);
}, [data]);
```

### 6.2 Commit 状态聚合

**文件**: `packages/amplication-client/src/VersionControl/hooks/useCommitStatus.ts`

```typescript
// 基于所有Build状态聚合Commit状态
const commitStatus = useMemo(() => {
  if (!commitBuilds?.length) return;
  const buildsInProgress = commitBuilds.some(b => b.status === Running);
  const buildsFailed = commitBuilds.some(b => b.status === Failed);
  const buildsCompleted = commitBuilds.some(b => b.status === Completed);

  if (buildsInProgress) return Running;   // 任一进行中 → Running
  if (buildsFailed) return Failed;         // 任一失败 → Failed
  if (buildsCompleted) return Completed;   // 全部完成 → Completed
}, [commitBuilds]);

// 获取Commit最后一条错误信息
const commitLastError = useMemo(() => {
  if (commitStatus !== Failed) return;
  const failedBuild = commitBuilds.find(b => b.status === Failed);
  const failedStep = failedBuild?.action.steps.find(s => s.status === Failed);
  const failedLog = failedStep?.logs.find(l => l.level === Error);
  return failedLog?.message;
}, [commitBuilds, commitStatus]);
```

### 6.3 Last Commit 展示

**文件**: `packages/amplication-client/src/VersionControl/LastCommit.tsx`

位于 Workspace Footer，展示最近一次 Commit 的状态：

- **状态图标**: `CommitBuildsStatusIcon` - Running显示旋转加载器，其他显示对应图标
- **错误展示**: 若有错误，显示红色错误文本 + "View details" 链接
- **Git链接**: 单Build时直接显示 `BuildGitLink` 按钮，多Build时显示 "View code (multiple builds)" 按钮
- **Commit ID**: 可点击跳转到 Commits 页面

### 6.4 Build 状态图标与样式映射

**文件**: `packages/amplication-client/src/VersionControl/constants.ts`

```typescript
// 图标映射
BUILD_STATUS_TO_ICON = {
  Completed: "check",
  Failed: "close",
  Invalid: "circle_loader",
  Running: "",          // 显示CircularProgress
  Canceled: "",
  Unknown: "",
};

// 颜色映射
BUILD_STATUS_TO_COLOR = {
  Completed: ThemeGreen,
  Failed: ThemeRed,
  Invalid: ThemeRed,
  Running: White,
  Canceled: Black20,
  Unknown: White,
};

// 步骤状态样式
STEP_STATUS_TO_STYLE = {
  Waiting:  { style: Warning,  icon: "refresh_cw" },
  Running:  { style: Warning,  icon: "refresh_cw" },
  Failed:   { style: Negative, icon: "info_i" },
  Success:  { style: Positive, icon: "check" },
};
```

### 6.5 Git PR 链接提取

**文件**: `packages/amplication-client/src/VersionControl/useBuildGitUrl.tsx`

```typescript
// 从PUSH_TO_GIT_*步骤的log meta中提取githubUrl
const PUSH_TO_GIT_STEP_NAME = "PUSH_TO_";

const gitUrl = useMemo(() => {
  const stepGithub = build?.action?.steps?.find((step) =>
    step.name.startsWith(PUSH_TO_GIT_STEP_NAME)
  );
  const log = stepGithub?.logs?.find(
    (log) => !isEmpty(log.meta) && !isEmpty(log.meta.githubUrl)
  );
  return log?.meta?.githubUrl || null;
}, [build?.action]);

// 根据URL判断Git平台并显示按钮标题
const gitPrTitle = useMemo(() => {
  if (gitUrl?.includes("github"))   return `View code (PR #${prNumber})`;
  if (gitUrl?.includes("gitlab"))   return `View code (MR #${prNumber})`;
  if (gitUrl?.includes("bitbucket"))return `View code (PR #${prNumber})`;
  if (gitUrl?.includes("azure"))    return `View code (PR #${prNumber})`;
  return "View code";
}, [gitUrl]);
```

### 6.6 CommitResourceListItem - 构建列表项

**文件**: `packages/amplication-client/src/VersionControl/CommitResourceListItem.tsx`

每个 Build 在 Commit 详情页中的展示项：

```
┌──────────────────────────────────────────────────────────────────┐
│  [状态图标] Build ID [截断ID]   资源名称  [代码生成器图标]   >  │
│  [变更数 N changes]    [步骤最后一条日志]   [View code (PR #123)] │
└──────────────────────────────────────────────────────────────────┘
```

- 使用 `useBuildWatchStatus` 实时更新状态
- 失败时显示红色 "Error. See logs for details"
- 变更数量链接跳转到 Changes 对比页面
- 整个卡片可点击跳转到 Build 详情页

### 6.7 BuildPage - 构建详情页

**文件**: `packages/amplication-client/src/VersionControl/BuildPage.tsx`

Build 详情页展示完整的执行日志：

```
┌─ Header ──────────────────────────────────────────────────────┐
│ ← Return to Commit [commitId]                                  │
│ [资源图标] 资源名称    Commit [commitId]         [View code PR] │
└────────────────────────────────────────────────────────────────┘
┌─ ActionLog ───────────────────────────────────────────────────┐
│ ┌─ Step 1: ADD_TO_QUEUE (✓ Success) ───────────────────────┐  │
│ │ • Create build generation task                            │  │
│ │ • Build version: abc12345                                 │  │
│ └───────────────────────────────────────────────────────────┘  │
│ ┌─ Step 2: GENERATE_APPLICATION (✓ Success) ───────────────┐  │
│ │ ...生成日志...                                             │  │
│ └───────────────────────────────────────────────────────────┘  │
│ ┌─ Step 3: PUSH_TO_GIT_Github (✓ Success) ─────────────────┐  │
│ │ • PR was successfully created at https://github.com/...   │  │
│ └───────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

- `ActionLog` 组件垂直展示所有步骤及日志
- 每个步骤有状态图标（等待/运行中/成功/失败）
- 步骤按时间顺序展开，支持动态高度调整

### 6.8 CommitsPage & CommitPage

**文件**: `packages/amplication-client/src/VersionControl/CommitsPage.tsx`
**文件**: `packages/amplication-client/src/VersionControl/CommitPage.tsx`

- **CommitsPage**: 左侧显示 Commit 历史列表（按时间倒序），右侧显示当前选中 Commit 的 Build 列表
- **CommitPage**: 显示单个 Commit 的所有 Build 资源列表
- 路由结构: `/{workspace}/{project}/commits/{commitId}/builds/{buildId}`

---

## 状态枚举汇总

### EnumBuildStatus (Build整体状态)

| 值 | 含义 | 前端颜色 | 前端图标 |
|----|------|----------|----------|
| Running | 构建进行中 | White | 旋转加载器 |
| Completed | 构建完成 | ThemeGreen | check |
| Failed | 构建失败 | ThemeRed | close |
| Invalid | 构建无效 | ThemeRed | circle_loader |
| Canceled | 构建已取消 | Black20 | 无 |
| Unknown | 状态未知 | White | 无 |

### EnumBuildGitStatus (Git同步状态)

| 值 | 含义 |
|----|------|
| NotConnected | 未连接Git仓库 |
| Waiting | 等待Git同步 |
| Completed | Git同步完成 |
| Failed | Git同步失败 |
| Canceled | Git同步已取消 |
| Unknown | 状态未知 |

### EnumActionStepStatus (步骤状态)

| 值 | 含义 | 样式 | 图标 |
|----|------|------|------|
| Waiting | 等待执行 | Warning | refresh_cw |
| Running | 执行中 | Warning | refresh_cw |
| Failed | 执行失败 | Negative | info_i |
| Success | 执行成功 | Positive | check |

### EnumCommitStrategy (提交策略)

| 值 | 含义 | 适用场景 |
|----|------|----------|
| All | 所有资源 | 单服务时默认 |
| AllWithPendingChanges | 仅含变更的资源 | 有变更时默认 |
| Specific | 指定资源ID | 多服务无变更时用户选择 |

---

## 关键 Kafka Topic 清单

| Topic | 生产者 | 消费者 | 触发时机 |
|-------|--------|--------|----------|
| `CODE_GENERATION_REQUEST_TOPIC` | BuildService | BuildRunnerService | Build.create()中generate()时 |
| `CODE_GENERATION_SUCCESS_TOPIC` | BuildRunnerService | BuildService | 所有子Job代码生成成功 |
| `CODE_GENERATION_FAILURE_TOPIC` | BuildRunnerService | BuildService | 任一子Job代码生成失败 |
| `CREATE_PR_REQUEST_TOPIC` | BuildService | GitSyncManager | saveToGitProvider()组装PR请求后 |
| `CREATE_PR_SUCCESS_TOPIC` | GitSyncManager | BuildService | PR创建成功 |
| `CREATE_PR_FAILURE_TOPIC` | GitSyncManager | BuildService | PR创建失败 |
| `USER_BUILD_TOPIC` | BuildService | 下游服务 | 代码生成成功后产物通知 |
| `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | BuildService | 插件服务 | 有私有插件时 |
| `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | 插件服务 | BuildService | 私有插件下载成功 |
| `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | 插件服务 | BuildService | 私有插件下载失败 |

---

## Build 完整状态流转

```
                    BuildService.create()
                            │
                            ▼
              ┌─────────────────────────────┐
              │  status: Running             │
              │  gitStatus: Waiting          │
              └──────────────┬───────────────┘
                             │
              代码生成失败？ / \ 代码生成成功？
               (handleCodeGenerationFailure) (handleCodeGenerationSuccess)
                      │                  │
                      ▼                  ▼
           ┌──────────────┐    ┌──────────────────────┐
           │ status:      │    │ 触发 saveToGitProvider│
           │ Failed       │    │ 创建PUSH步骤(Running) │
           │ gitStatus:   │    └──────────┬───────────┘
           │ Canceled     │               │
           └──────────────┘       PR失败？/ \ PR成功？
                                     (handleCreatePrFailure) (handleCreatePrSuccess)
                                           │          │
                                           ▼          ▼
                                  ┌────────────┐ ┌──────────────┐
                                  │ status:    │ │ status:      │
                                  │ Failed     │ │ Completed    │
                                  │ gitStatus: │ │ gitStatus:   │
                                  │ Failed     │ │ Completed    │
                                  └────────────┘ └──────────────┘
```

---

## 核心服务职责清单

| 服务 | 所在包 | 核心职责 |
|------|--------|----------|
| ProjectResolver | amplication-server | GraphQL commit mutation入口，授权校验 |
| ProjectService | amplication-server | Commit创建完整流程：计费校验→变更获取→版本创建→Build创建 |
| BuildService | amplication-server | Build CRUD、代码生成触发、Git推送、状态更新、Kafka回调处理 |
| ActionService | amplication-server | Action/ActionStep/ActionLog CRUD，步骤执行引擎run() |
| CommitService | amplication-server | Commit查询、变更列表(PendingChange)聚合 |
| CommitResolver | amplication-server | Commit/Changes GraphQL查询接口 |
| BuildResolver | amplication-server | Build GraphQL查询接口，archiveURI/status动态计算 |
| BuildRunnerService | amplication-build-manager | 代码生成Job执行、产物复制、状态聚合 |
| BuildJobsHandlerService | amplication-build-manager | Build拆分子Job、Redis状态存储与聚合 |
| UserActionService | amplication-server | 用户操作状态评估、元数据更新 |
