# Git Provider 同步流程分析

## 一、整体架构概览

Git Provider 同步系统由三个核心服务组成，通过 Kafka 消息队列进行异步通信：

```
┌─────────────────────┐      Kafka       ┌──────────────────────┐     Kafka      ┌─────────────────────┐
│  amplication-server │ ───────────────▶ │ git-sync-manager (ee)│ ──────────────▶ │  amplication-server │
│  (BuildService)     │   CREATE_PR_REQ  │  (PullRequestService)│  CREATE_PR_*    │  (BuildController)  │
│  (GitProviderService)│                  │  (DiffService)       │                │                     │
└─────────────────────┘                  └──────────────────────┘                └─────────────────────┘
          ▲                                                                    │
          │ GraphQL                                                            │ Kafka Event
          ▼                                                                    ▼
┌─────────────────────┐                  ┌──────────────────────┐
│  Frontend (Client)  │                  │  DSG (Code Gen)      │
└─────────────────────┘                  └──────────────────────┘
```

---

## 二、外部仓库连接流程

### 2.1 入口层：GraphQL Resolver

所有 Git 相关操作通过 [GitResolver](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.resolver.ts) 暴露 GraphQL API：

| Mutation/Query | 用途 |
|---|---|
| `getGitResourceInstallationUrl` | 获取 Git Provider 授权安装 URL |
| `createOrganization` | 创建 Git 组织（GitHub App/AWS CodeCommit） |
| `completeGitOAuth2Flow` | 完成 OAuth2 授权流程（Bitbucket/GitLab/Azure DevOps） |
| `connectResourceToNewRemoteGitRepository` | 创建新仓库并关联资源 |
| `connectResourceGitRepository` | 关联已有仓库到资源 |
| `remoteGitRepositories` | 获取组织下的仓库列表 |
| `gitOrganizations` | 获取工作区的 Git 组织列表 |

### 2.2 服务层：GitProviderService

核心业务逻辑在 [GitProviderService](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts) 中实现。

#### 2.2.1 OAuth 授权流程（以 GitHub App 为例）

```
用户点击连接
    │
    ▼
getGitInstallationUrl() ──► createGitClientWithoutProperties()
    │                              │
    │                              ▼
    │                        GitFactory.getProvider()
    │                              │
    ▼                              ▼
返回 GitHub App 安装 URL     创建对应 Provider 实例
    │                              (GithubService/BitBucketService 等)
    ▼
用户在 GitHub 完成授权
    │
    ▼
createOrganization(githubInput)
    │
    ├─► 构造 GitHubProviderOrganizationProperties { installationId }
    ├─► GitClientService.create() 初始化客户端
    ├─► gitClientService.getOrganization() 获取远程组织信息
    ├─► projectService.disableDemoRepoForAllWorkspaceProjects()
    └─► prisma.gitOrganization.upsert() 保存/更新组织信息
```

关键代码位置：
- 获取安装 URL：[git.provider.service.ts:L763-L771](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L763-L771)
- 创建 GitHub 组织：[git.provider.service.ts:L615-L745](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L615-L745)
- OAuth2 流程完成：[git.provider.service.ts:L864-L938](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L864-L938)

#### 2.2.2 仓库连接流程

```
connectResourceToNewRemoteGitRepository()
    │
    ├─► createRemoteGitRepository()
    │      ├─► validateGitOrganization()  校验组织归属
    │      ├─► createGitClient(organization)
    │      └─► executeAndUpdateGitProviderProperties()
    │             └─► gitClientService.createRepository()
    │
    └─► connectResourceGitRepository()
           └─► prisma.gitRepository.create() 关联资源与仓库
```

关键代码位置：
- 创建并连接新仓库：[git.provider.service.ts:L276-L330](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L276-L330)
- 创建远程仓库：[git.provider.service.ts:L383-L431](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L383-L431)
- 连接已有仓库：[git.provider.service.ts:L575-L613](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L575-L613)

#### 2.2.3 Token 自动刷新机制

`executeAndUpdateGitProviderProperties()` 是一个包装函数，每次 Git 操作后自动检查并刷新认证数据：

```typescript
async executeAndUpdateGitProviderProperties<T>(
  operation: () => Promise<T>,
  gitOrganization: GitOrganization,
  client: GitClientService
): Promise<T> {
  const result = await operation();

  if (!(await client.isAuthDataRefreshed())) {
    return result;
  }

  const updateAuth = await client.getAuthData();
  // 更新 providerProperties 到数据库
  await this.prisma.gitOrganization.update({ ... });
  return result;
}
```

代码位置：[git.provider.service.ts:L831-L862](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L831-L862)

### 2.3 Provider 工厂层

[GitFactory](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/git/src/git-factory.ts) 是抽象工厂，根据 provider 类型创建具体实现：

| Provider | 实现类 | 认证方式 |
|---|---|---|
| `Github` | `GithubService` | GitHub App (installationId) |
| `Bitbucket` | `BitBucketService` | OAuth2 |
| `GitLab` | `GitLabService` | OAuth2 |
| `AzureDevOps` | `AzureDevOpsService` | OAuth2 |
| `AwsCodeCommit` | `AwsCodeCommitService` | AccessKey + Git Credentials |

Provider 接口定义：[git-provider.interface.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/git/src/git-provider.interface.ts)

---

## 三、生成产物同步流程

### 3.1 完整数据流图

```
用户触发 Commit/Build
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ 1. BuildService.create()                                   │
│    ├─ 创建 Build 记录 (status=Running, gitStatus=Waiting)  │
│    └─ 检查是否有私有插件                                    │
│           ├─ 有：发送 DOWNLOAD_PRIVATE_PLUGINS_REQ         │
│           └─ 无：发送 CODE_GENERATION_REQ                  │
└───────────────────────────────────────────────────────────┘
        │
        ▼  (DSG 完成代码生成)
┌───────────────────────────────────────────────────────────┐
│ 2. BuildController.onCodeGenerationSuccess()               │
│    └─► BuildService.saveToGitProvider()                    │
│           ├─ 组装 gitSettings (仓库/分支/commit 信息)      │
│           └─► 发送 CREATE_PR_REQUEST_TOPIC                 │
└───────────────────────────────────────────────────────────┘
        │  Kafka
        ▼
┌───────────────────────────────────────────────────────────┐
│ 3. PullRequestController.generatePullRequest()             │
│    └─► PullRequestService.createPullRequest()              │
│           ├─► DiffService.listOfChangedFiles()             │
│           │     └─ 对比 oldBuild 与 newBuild 产物目录      │
│           └─► GitClientService.createPullRequest()         │
│                  ├─ Git CLI clone/checkout                 │
│                  ├─ prepareFilesForPullRequest()           │
│                  ├─ accumulativePullRequest()              │
│                  │    ├─ calculateDiffAndResetBranch()     │
│                  │    ├─ gitCli.commit()                   │
│                  │    ├─ provider.getPullRequest()         │
│                  │    └─ provider.createPullRequest()      │
│                  └─ 返回 pullRequestUrl + diffStat         │
└───────────────────────────────────────────────────────────┘
        │  Kafka (Success/Failure)
        ▼
┌───────────────────────────────────────────────────────────┐
│ 4. BuildController.onPullRequestCreated/Failure()          │
│    └─ 更新 Build 状态 + ActionStep 日志 + 上报分析          │
└───────────────────────────────────────────────────────────┘
```

### 3.2 阶段一：Build 创建与代码生成

入口在 [BuildService.create()](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352)

```typescript
async create(args: CreateBuildArgs): Promise<Build> {
  // 1. 创建 Build 记录
  const build = await this.prisma.build.create({
    status: EnumBuildStatus.Running,
    gitStatus: EnumBuildGitStatus.Waiting,
    action: { create: { steps: { create: createInitialStepData(...) } } }
  });

  // 2. 检查私有插件
  const resourcePrivatePlugins = await this.pluginInstallationService
    .getInstalledPrivatePluginsForBuild(resourceId);

  // 3. 根据是否有私有插件决定流程
  if (resourcePrivatePlugins.length > 0) {
    await this.downloadPrivatePlugins(logger, build, user, resourcePrivatePlugins);
  } else {
    await this.generate(logger, build, user);  // 发送 CODE_GENERATION_REQ
  }
}
```

代码生成请求通过 Kafka 发送到 DSG 服务：
```typescript
const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
  value: { resourceId, buildId }
};
await this.kafkaProducerService.emitMessage(
  KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
  codeGenerationEvent
);
```

代码位置：[build.service.ts:L568-L618](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

### 3.3 阶段二：代码生成完成 → 触发 Git 同步

DSG 完成后，[BuildController.onCodeGenerationSuccess()](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101) 被触发：

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(message) {
  // 先执行 Git 同步
  await this.buildService.saveToGitProvider(args.buildId);
  // 再完成 GENERATE 步骤
  await this.buildService.onCodeGenerationSuccess(args.buildId);
}
```

#### BuildService.saveToGitProvider() 核心逻辑

代码位置：[build.service.ts:L1130-L1342](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L1130-L1342)

```typescript
async saveToGitProvider(buildId: string): Promise<void> {
  // 分支 1：使用 Demo 仓库
  if (project.useDemoRepo) {
    gitSettings = {
      gitProvider: EnumGitProvider.Github,
      gitOrganizationName: GITHUB_DEMO_REPO_ORGANIZATION_NAME,
      gitRepositoryName: project.demoRepoName,
      gitProviderProperties: { installationId: GITHUB_DEMO_REPO_INSTALLATION_ID },
      ...
    };
  }
  // 分支 2：使用用户自定义仓库
  else {
    const resourceRepository = await this.resourceService.gitRepository(build.resourceId);

    // 资源未连接 Git 仓库：跳过同步，标记为 NotConnected
    if (!resourceRepository) {
      await this.updateBuildStatuses(
        build.id,
        EnumBuildStatus.Completed,
        EnumBuildGitStatus.NotConnected
      );
      return;
    }

    const gitOrganization = await this.resourceService.gitOrganizationByResource(...);
    const gitProviderArgs = await this.gitProviderService.getGitProviderProperties(gitOrganization);

    gitSettings = {
      gitOrganizationName: gitOrganization.name,
      gitRepositoryName: resourceRepository.name,
      baseBranchName: canUseCustomBaseBranch ? resourceRepository.baseBranchName : "",
      repositoryGroupName: resourceRepository.groupName,
      gitProvider: gitProviderArgs.provider,
      gitProviderProperties: gitProviderArgs.providerOrganizationProperties,
      commit: { title: commitTitle, body: commitBody },
    };
  }

  // 创建 PUSH_TO_GIT 步骤并发送 Kafka 消息
  return this.actionService.run(build.actionId, PUSH_TO_GIT_STEP_NAME, ..., async (step) => {
    const createPullRequestEvent: CreatePrRequest.KafkaEvent = {
      key: { resourceRepositoryId: kafkaEventKey, resourceId: ... },
      value: {
        ...gitSettings,
        resourceId, resourceName, newBuildId, oldBuildId,
        gitResourceMeta, isBranchPerResource, overrideCustomizableFilesInGit,
      }
    };
    await this.kafkaProducerService.emitMessage(
      KAFKA_TOPICS.CREATE_PR_REQUEST_TOPIC,
      createPullRequestEvent
    );
  });
}
```

### 3.4 阶段三：git-sync-manager 处理 PR 创建

#### PullRequestController 消费消息

代码位置：[pull-request.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts)

```typescript
@EventPattern(KAFKA_TOPICS.CREATE_PR_REQUEST_TOPIC)
async generatePullRequest(message, context) {
  try {
    const { pullRequestUrl, diffStat } = await KafkaPacemaker.wrapLongRunningMethod(
      context,
      () => this.pullRequestService.createPullRequest(validArgs)
    );

    // 成功：发送 CREATE_PR_SUCCESS_TOPIC
    await this.producerService.emitMessage(KAFKA_TOPICS.CREATE_PR_SUCCESS_TOPIC, {
      value: { url: pullRequestUrl, diffStat, gitProvider, buildId }
    });
  } catch (error) {
    if (error instanceof NoChangesOnPullRequest) {
      // 无变更也视为成功（跳过 PR 创建）
      await this.producerService.emitMessage(KAFKA_TOPICS.CREATE_PR_SUCCESS_TOPIC, ...);
      return;
    }
    // 失败：发送 CREATE_PR_FAILURE_TOPIC
    await this.producerService.emitMessage(KAFKA_TOPICS.CREATE_PR_FAILURE_TOPIC, {
      value: { buildId, gitProvider, errorMessage: error.message }
    });
  }
}
```

#### PullRequestService.createPullRequest() 核心逻辑

代码位置：[pull-request.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.service.ts)

```typescript
async createPullRequest({ ... }: CreatePrRequest.Value) {
  // 1. 确定分支名称
  const head = isBranchPerResource ? `amplication-${resourceName}` : `amplication`;

  // 2. 获取新旧 build 的差异文件列表
  const changedFiles = await this.diffService.listOfChangedFiles(
    resourceId, oldBuildId, newBuildId
  );

  // 3. 创建 Git 客户端并执行 PR 创建
  const gitClientService = await new GitClientService().create({...});
  return await gitClientService.createPullRequest({
    owner, repositoryName: repo, repositoryGroupName,
    branchName: head, files: changedFiles, ...
  });
}
```

#### DiffService：产物差异比较

代码位置：[diff.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/ee/packages/git-sync-manager/src/diff/diff.service.ts)

```typescript
async listOfChangedFiles(resourceId, previousAmplicationBuildId, newAmplicationBuildId) {
  const newBuildPath = this.buildsPathFactory.get(resourceId, newAmplicationBuildId);
  validateIfBuildExist(newBuildPath);

  // 首次构建：返回所有新文件
  if (!previousAmplicationBuildId) {
    return this.getAllModulesForPath(newBuildPath);
  }

  const oldBuildPath = this.buildsPathFactory.get(resourceId, previousAmplicationBuildId);
  if (!existsSync(oldBuildPath)) {
    return this.getAllModulesForPath(newBuildPath);
  }

  // 使用 dir-compare 对比两个目录
  const res = await compare(oldBuildPath, newBuildPath, { compareContent: true });
  const modules = mapDiffSetToPrModule(res.diffSet, [deleteFilesVisitor]);

  // 合并：所有新文件 + 删除的文件
  return [...await this.getAllModulesForPath(newBuildPath), ...modules];
}
```

#### GitClientService.createPullRequest()：Git 操作核心

代码位置：[git-client.service.ts:L237-L363](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/git/src/git-client.service.ts#L237-L363)

```typescript
async createPullRequest(createPullRequestArgs) {
  try {
    // 1. 解析 baseBranch（用户指定或仓库默认分支）
    // 2. Git CLI clone 仓库到临时目录
    await gitCli.clone();

    // 3. 首次提交：如果 baseBranch 上还没有 Amplication 的提交，创建初始 commit (README.md)
    const firstCommitOnBaseBranch = await gitCli.getFirstCommitSha(baseBranch);
    if (!firstCommitOnBaseBranch) {
      await this.createInitialCommit({...});
    }

    // 4. 处理 .amplicationignore 规则
    const amplicationIgnoreManager = await this.manageAmplicationIgnoreFile(...);

    // 5. 过滤文件（忽略 amplicationignore + 处理可覆盖文件）
    const preparedFiles = await prepareFilesForPullRequest(
      gitResourceMeta, files, amplicationIgnoreManager, overrideCustomizableFilesInGit
    );

    // 6. 累加式 PR（accumulativePullRequest）
    const pullRequestUrl = await this.accumulativePullRequest({
      gitCli, preparedFiles, baseBranch, branchName, ...
    });

    const diffStat = await gitCli.getShortStat();
    await gitCli.deleteRepositoryDir();
    return { pullRequestUrl, diffStat };
  } catch (error) {
    await gitCli.deleteRepositoryDir();
    throw error;
  }
}
```

##### accumulativePullRequest：累加式 PR 策略

代码位置：[git-client.service.ts:L365-L471](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/git/src/git-client.service.ts#L365-L471)

这是同步流程最核心的部分，确保用户自定义修改不会被覆盖：

```
1. restoreAmplicationBranchIfNotExists()
   └─ 确保 amplication 分支存在（从 baseBranch 创建）

2. calculateDiffAndResetBranch(useBeforeLastCommit=false)
   ├─ 查找 amplication 分支上最近的 Amplication 提交
   ├─ git diff 计算该提交之后的所有变更（即用户自定义修改）
   └─ git reset --hard 回退到该提交，强制推送

3. gitCli.commit() 提交本次构建的所有新文件

4. calculateDiffAndResetBranch(useBeforeLastCommit=true)
   └─ 计算本次构建与上次构建之间的差异

5. applyPostCommit() 依次应用第 2 步和第 4 步的 diff patch
   └─ 将用户自定义修改 + 构建差异 重新应用到分支上

6. provider.getPullRequest() / createPullRequest()
   └─ 查找现有 PR，不存在则创建新 PR

7. provider.createPullRequestComment()
   └─ 在 PR 上添加本次构建的评论
```

### 3.5 阶段四：结果回传

[BuildController](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.controller.ts) 消费 Kafka 结果消息：

| Kafka Topic | 处理方法 |
|---|---|
| `CREATE_PR_SUCCESS_TOPIC` | `onPullRequestCreated()` → `BuildService.onCreatePRSuccess()` |
| `CREATE_PR_FAILURE_TOPIC` | `onPullRequestFailure()` → `BuildService.onCreatePRFailure()` |
| `CREATE_PR_LOG_TOPIC` | `onCreatePullRequestLog()` |

---

## 四、失败反馈机制

### 4.1 三层错误处理架构

```
┌───────────────────────────────────────────────────────────────┐
│ Layer 1: git-sync-manager (PullRequestController)             │
│   - try/catch 包裹 createPullRequest()                        │
│   - 特殊异常 NoChangesOnPullRequest → 视为成功跳过            │
│   - 其他异常 → 发送 CREATE_PR_FAILURE_TOPIC                   │
│   - 所有阶段日志 → 发送 CREATE_PR_LOG_TOPIC                   │
└───────────────────────────────────────────────────────────────┘
        │ Kafka
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Layer 2: amplication-server (BuildController)                 │
│   - 消费 FAILURE 消息 → 转发到 BuildService                   │
│   - 消费 LOG 消息 → 写入 ActionStep logs                      │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Layer 3: amplication-server (BuildService)                    │
│   - 更新 Build.status/ gitStatus = Failed                     │
│   - 更新 ActionStep.status = Failed                           │
│   - 写入错误日志到 ActionLog                                  │
│   - resourceService.reportSyncMessage() 同步消息到 Resource   │
│   - analytics.trackManual() 上报 GitSyncError 事件            │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 失败处理详细流程

#### 4.2.1 CREATE_PR 失败处理

代码位置：[build.service.ts:L917-L968](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L917-L968)

```typescript
public async onCreatePRFailure(response: CreatePrFailure.Value) {
  const build = await this.prisma.build.findUnique({...});
  const steps = await this.actionService.getSteps(build.actionId);
  const step = steps.find(s => s.name === PUSH_TO_GIT_STEP_NAME);

  // 1. 同步错误消息到 Resource（前端展示）
  await this.resourceService.reportSyncMessage(
    build.resourceId,
    `Error: ${response.errorMessage}`
  );

  // 2. 写入 Action 步骤日志
  await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FAILED_LOG(response.gitProvider));
  await this.actionService.log(step, EnumActionLogLevel.Error, response.errorMessage);

  // 3. 标记步骤失败
  await this.actionService.complete(step, EnumActionStepStatus.Failed);

  // 4. 更新 Build 状态
  await this.updateBuildStatuses(
    build.id,
    EnumBuildStatus.Failed,
    EnumBuildGitStatus.Failed
  );

  // 5. 上报分析事件
  await this.analytics.trackManual({
    user: { accountId, workspaceId },
    data: {
      properties: { resourceId, projectId, message: response.errorMessage },
      event: EnumEventType.GitSyncError,
    },
  });
}
```

#### 4.2.2 代码生成失败处理

代码位置：[build.service.ts:L511-L534](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L511-L534)

```typescript
public async onCodeGenerationFailure(response: CodeGenerationFailure.Value) {
  const { buildId } = response;

  // 1. 记录错误日志
  await this.onDsgLog({
    buildId,
    level: "error",
    message: response.errorMessage || "Code generation failed",
  });

  // 2. 标记 GENERATE 步骤失败
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);
  await this.actionService.complete(step, EnumActionStepStatus.Failed);

  // 3. 更新 Build 状态，Git 同步状态设置为 Canceled
  await this.updateBuildStatuses(
    buildId,
    EnumBuildStatus.Failed,
    EnumBuildGitStatus.Canceled
  );
}
```

#### 4.2.3 同步消息前端展示

`reportSyncMessage()` 将错误信息写入 Resource 的 `gitRepositorySyncMessage` 字段，前端轮询或订阅获取：

代码位置（部分）：
```typescript
async reportSyncMessage(resourceId: string, message: string): Promise<Resource> {
  return this.prisma.resource.update({
    where: { id: resourceId },
    data: { gitRepositorySyncMessage: message },
  });
}
```

### 4.3 成功路径的异常兜底

即使 PR 创建成功回调，内部处理异常也有兜底：

代码位置：[build.service.ts:L851-L915](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L851-L915)

```typescript
public async onCreatePRSuccess(response: CreatePrSuccess.Value) {
  try {
    // ... 正常成功处理（reportSyncMessage、更新 LOC、完成步骤等）
  } catch (error) {
    // 成功回调内部异常 → 降级为失败处理
    await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FAILED_LOG(response.gitProvider));
    await this.actionService.logInfo(step, error);
    await this.actionService.complete(step, EnumActionStepStatus.Failed);
    await this.updateBuildStatuses(build.id, EnumBuildStatus.Failed, EnumBuildGitStatus.Failed);
    await this.resourceService.reportSyncMessage(build.resourceId, `Error: ${error}`);
  }
}
```

### 4.4 超时保护：过期 Build 自动失败

代码位置：[build.service.ts:L1547-L1561](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L1547-L1561)

```typescript
isBuildStale(build: Build): boolean {
  if (build.status === EnumBuildStatus.Running) {
    const stalePeriod = 5 * 60 * 60 * 1000; // 5 小时
    if (Date.now() - build.createdAt.getTime() > stalePeriod) {
      return true;
    }
  }
  return false;
}
```

`calcBuildStatus()` 在查询时会检查 build 是否超时并自动标记为失败。

---

## 五、关键 Kafka Topic 列表

| Topic | 生产者 | 消费者 | 用途 |
|---|---|---|---|
| `CODE_GENERATION_REQUEST_TOPIC` | BuildService | DSG Service | 请求代码生成 |
| `CODE_GENERATION_SUCCESS_TOPIC` | DSG Service | BuildController | 代码生成成功 |
| `CODE_GENERATION_FAILURE_TOPIC` | DSG Service | BuildController | 代码生成失败 |
| `DSG_LOG_TOPIC` | DSG Service | BuildController | 代码生成日志 |
| `CREATE_PR_REQUEST_TOPIC` | BuildService | PullRequestController | 请求创建 PR |
| `CREATE_PR_SUCCESS_TOPIC` | PullRequestController | BuildController | PR 创建成功 |
| `CREATE_PR_FAILURE_TOPIC` | PullRequestController | BuildController | PR 创建失败 |
| `CREATE_PR_LOG_TOPIC` | PullRequestController | BuildController | PR 创建日志 |
| `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | BuildService | PrivatePluginController | 请求下载私有插件 |
| `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | PrivatePluginController | BuildController | 插件下载成功 |
| `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | PrivatePluginController | BuildController | 插件下载失败 |

---

## 六、核心状态流转

### Build Status

| 状态 | 触发时机 |
|---|---|
| `Running` | Build 创建时 |
| `Completed` | 所有 ActionStep 成功 |
| `Failed` | 任一步骤失败 / 超时 5h |
| `Invalid` | ActionStep 数据异常 |
| `Unknown` | 旧数据兜底（自动重算为 Failed） |

### Build Git Status

| 状态 | 触发时机 |
|---|---|
| `Waiting` | Build 创建时，等待同步 |
| `Completed` | PR 创建成功 |
| `Failed` | PR 创建失败 |
| `Canceled` | 代码生成失败，跳过同步 |
| `NotConnected` | 资源未关联 Git 仓库 |

### ActionStep Status

| 状态 | 说明 |
|---|---|
| `Running` | 执行中 |
| `Success` | 步骤成功 |
| `Failed` | 步骤失败 |
| `Skipped` | 步骤跳过 |

---

## 七、失败反馈边界分析

### 7.1 创建 PR 请求失败的边界情况

#### 7.1.1 Kafka 消息发送失败边界

**KafkaProducerService.emitMessage() 的实现：

[KafkaProducer.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts)

```typescript
async emitMessage(topic, message, schemaIds) {
  const kafkaMessage = await this.serializer.serialize(message, schemaIds);
  return await new Promise((resolve, reject) => {
    this.kafkaClient.emit(topic, kafkaMessage).subscribe({
      error: (err) => reject(err),   // Kafka broker 不可达 / 发送异常
      next: () => resolve(),          // kafkajs producer.send() Promise resolve 回调
    });
  });
}
```

**边界 1：emitMessage 的 `next` 回调含义由 kafkajs 的 producer.send() 决定。** 本项目未显式配置 producer `acks` 参数（见 [createNestjsKafkaConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts)），使用 kafkajs 默认值 `acks=-1`（即 `all`），表示所有 ISR（in-sync replicas）副本都写入后才返回确认。因此 `next` 回调表示 broker 集群已持久化该消息，而非仅进入 producer 本地缓冲区。

真实的消息丢失窗口：若 `next` 回调 resolve 后，所有 ISR 副本同时在刷盘前崩溃（极罕见），消息可能丢失。对于常规的单 broker 临时不可达场景，若 `next` 已 resolve，消息已落盘持久化。

**saveToGitProvider 中对 emitMessage 的异常处理：

代码位置：[build.service.ts:L1285-L1342](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L1285-L1342)

```typescript
return this.actionService.run(build.actionId, PUSH_TO_GIT_STEP_NAME, ..., async (step) => {
  try {
    await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_START_LOG);
    // ... 构造 createPullRequestEvent
    await this.kafkaProducerService.emitMessage(
      KAFKA_TOPICS.CREATE_PR_REQUEST_TOPIC,
      createPullRequestEvent
    );
  } catch (error) {
    // ❌ 边界 2：这里只打印日志，但不标记步骤失败，不更新 build.gitStatus
    logger.error("Failed to emit Create Pull Request Message.", error);
  }
}, true);  // true = leaveStepOpenAfterSuccessfulExecution
```

| 失败场景 | 后果 | 用户感知 |
|---|---|---|
| emitMessage 抛异常（Kafka 不可达） | 仅打印 logger.error，PUSH_TO_GIT 步骤保持 Running，build.gitStatus 保持 Waiting | ❌ Build 状态永不结束，前端轮询永远 Running，直到 5h 后被 isBuildStale 标记 Failed |
| emitMessage 成功但 broker 未持久化 | 消息丢失，同上 | 同上 |
| git-sync-manager 消费后处理异常 | CREATE_PR_FAILURE_TOPIC 正常发出，走 onCreatePRFailure | ✅ 正常失败反馈 |

**边界 3：ActionService.run() 的异常传播逻辑：

[action.service.ts:L275-L295](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295)

```typescript
async run<T>(actionId, stepName, message, stepFunction, leaveStepOpenAfterSuccessfulExecution) {
  const step = await this.createStep(actionId, stepName, message);
  try {
    const result = await stepFunction(step);
    if (!leaveStepOpenAfterSuccessfulExecution) {
      await this.complete(step, EnumActionStepStatus.Success);
    }
    return result;
  } catch (error) {
    this.logger.error(error.message, error);
    await this.log(step, EnumActionLogLevel.Error, error.message);
    await this.complete(step, EnumActionStepStatus.Failed);
    throw error;
  }
}
```

由于 saveToGitProvider 中 stepFunction 内部的 try/catch 吞掉了 emitMessage 异常，run() 不会走到 catch 分支。

#### 7.1.2 长任务心跳机制

git-sync-manager 中使用 KafkaPacemaker 防止 consumer 防止 rebalance：

[pacemaker.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/pacemaker/pacemaker.service.ts)

```typescript
static async wrapLongRunningMethod<T>(kafkaContext, fn, timeout = 3000) {
  const heartbeat = kafkaContext.getHeartbeat();
  // 每 3s 发送一次心跳，防止 consumer 被认为挂起
  while (!isFnDone) {
    await Promise.race([fnPromise, sleep(timeout)]);
    try {
      await heartbeat();
    } catch (error) {
      // swallow the error - heartbeat 失败不影响业务执行
    }
  }
}
```

边界：心跳失败被静默吞掉，若长时间心跳全部失败，consumer group rebalance，消息可能被重复消费，但不会触发重复创建 PR。

#### 7.1.3 NoChangesOnPullRequest 特殊处理

[pull-request.controller.ts:L116-L135](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L116-L135)

```typescript
} catch (error) {
  if (error instanceof NoChangesOnPullRequest) {
    await this.log(validArgs.newBuildId, LogLevel.Warn, "Hey there! Looks like your code hasn't changed since the last build...");
    await this.producerService.emitMessage(KAFKA_TOPICS.CREATE_PR_SUCCESS_TOPIC, {
      value: { url: error.pullRequestUrl, gitProvider, buildId }
    });
    return;
  }
  // 其他错误走 FAILURE
}
```

边界：无变更场景作为 SUCCESS 回传，build.status 被标记 Completed，但 diffStat 为 undefined。

---

### 7.2 前端读取同步消息路径

#### 7.2.1 后端写入路径

reportSyncMessage() 实际更新的字段：

[resource.service.ts:L1585-L1609](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1585-L1609)

```typescript
async reportSyncMessage(resourceId, message) {
  return this.prisma.resource.update({
    where: { id: resourceId },
    data: {
      githubLastMessage: message,    // 同步消息文本
      githubLastSync: new Date(), // 同步时间戳
    }
  });
}
```

Resource GraphQL 模型暴露字段：

[Resource.ts:L71-L79](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/models/Resource.ts#L71-L79)

```typescript
@Field(() => String, { nullable: true })
githubLastMessage?: string;

@Field(() => Date, { nullable: true })
githubLastSync?: Date;
```

注意：字段命名为 githubLast* 但实际用于所有 Git Provider（不限于 GitHub）。

#### 7.2.2 前端查询路径

**资源列表查询 GET_RESOURCES：**

[resourcesQueries.ts:L57-L141](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts#L57-L141)

```graphql
query getResources($where: ...) {
  resources(...) {
    id
    githubLastSync
    gitRepository { ... }
    builds(orderBy: { createdAt: Desc }, take: 1) {
      id
      version
      createdAt
      status
      codeGeneratorVersion
      // 注意：GET_RESOURCES 查询不包含 githubLastMessage，也不包含 builds[].gitStatus
    }
  }
}
```

**单个资源查询 GET_RESOURCE：

[resourcesQueries.ts:L48-L55](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts#L48-L55)

```graphql
query getResource($id: String!) {
  resource(where: { id: $id }) {
    ...ResourceFields  // 包含 githubLastSync + githubLastMessage
  }
}
```

**ResourceGitStatusPanel 组件展示：

[ResourceGitStatusPanel.tsx:L36-L90](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/Resource/git/ResourceGitStatusPanel.tsx#L36-L90)

```tsx
const lastSync = resource?.githubLastSync
  ? new Date(resource.githubLastSync)
  : null;
const lastSyncDate = lastSync ? format(lastSync, DATE_FORMAT) : "Never";
// ❌ 仅展示 Last sync 时间，不展示 githubLastMessage 内容
```

#### 7.2.3 Build 详情轮询

前端通过 useBuildWatchStatus hook 每 5 秒轮询一次：

[useBuildWatchStatus.tsx:L7-L48](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/VersionControl/useBuildWatchStatus.tsx#L7-L48)

```typescript
const POLL_INTERVAL = 5000;

const useBuildWatchStatus = (build) => {
  const { data, startPolling, stopPolling, refetch } = useQuery(GET_BUILD, {
    variables: { buildId: build?.id },
    skip: !shouldReload(build),
  });

  useEffect(() => {
    if (!shouldReload(data?.build)) {
      stopPolling();
    } else {
      startPolling(POLL_INTERVAL);
    }
    data && commitUtils.updateBuildStatus(data.build);
  }, [data, ...]);

  // Build 完成（非 Running 状态才停止轮询
  function shouldReload(build) {
    return build?.status === models.EnumBuildStatus.Running;
  }
```

GET_BUILD 查询字段包含 action.steps.logs：

[useBuildWatchStatus.tsx:L56-L97](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/VersionControl/useBuildWatchStatus.tsx#L56-L97)

```graphql
query build($buildId: String!) {
  build(where: { id: $buildId }) {
    id
    status
    action {
      steps {
        name
        status
        logs {
          message    // 步骤日志（含错误信息）
          level
        }
      }
    }
  }
}
```

ActionLog 组件渲染步骤日志：

[ActionLog.tsx:L47-L206](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-client/src/VersionControl/ActionLog.tsx#L47-L206)

```tsx
// 根据 step.status 判断整体 Action 状态
// step.logs 渲染每一步的日志（红色 Error 级别显示红色）
```

| 信息获取路径 | 包含错误信息 | 展示位置 |
|---|---|---|
| Resource.githubLastMessage | "Error: xxx" | ❌ 后端写入，但 ResourceGitStatusPanel 不展示 |
| Build.action.steps.logs | 步骤详细日志 | BuildPage 日志面板 |
| Build.status / Build.gitStatus | Completed/Failed 状态码 | BuildPage 顶部、Commit 列表 |

---

### 7.3 状态刷新机制

#### 7.3.1 后端动态计算 Build 状态

BuildResolver.status ResolveField：

[build.resolver.ts:L78-L88](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.resolver.ts#L78-L88)

```typescript
@ResolveField()
status(@Parent() build: Build): Promise<EnumBuildStatus> {
  if (this.service.isBuildStale(build)) {
    return this.service.calcBuildStatus(build.id);  // 每次查询都重新计算
  }
  if (build.status === EnumBuildStatus.Unknown) {
    return this.service.calcBuildStatus(build.id);
  }
  return Promise.resolve(EnumBuildStatus[build.status]);
}
```

isBuildStale 判断超时：

[build.service.ts:L1547-L1561](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L1547-L1561)

```typescript
isBuildStale(build) {
  if (build.status === EnumBuildStatus.Running) {
    const stalePeriod = 5 * 60 * 60 * 1000; // 5 小时
    if (Date.now() - build.createdAt.getTime() > stalePeriod) {
      return true;
    }
  }
  return false;
}
```

calcBuildStatus 根据 steps 计算最终状态：

[build.service.ts:L1563-L1618](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L1563-L1618)

```typescript
async calcBuildStatus(buildId) {
  const build = await this.prisma.build.findUnique({ include: ACTION_INCLUDE });

  if (this.isBuildStale(build)) {
    await this.updateBuildStatuses(buildId, EnumBuildStatus.Failed, EnumBuildGitStatus.Failed);
    return EnumBuildStatus.Failed;
  }

  if (build.status !== EnumBuildStatus.Unknown)
    return build.status as EnumBuildStatus;

  // 重新计算：所有 steps 状态聚合
  if (steps.every(step => step.status === Success) return Completed
  if (steps.some(step => step.status === Failed)) return Failed
  // 兜底：Unknown 全部置为 Failed
}
```

#### 7.3.2 前端状态刷新触发

| 刷新机制 | 触发时机 | 覆盖场景 |
|---|---|---|
| useBuildWatchStatus 轮询（5s） | BuildPage 打开时 | Running 状态期间每 5 秒查询一次 build 查询时触发后端 calcBuildStatus 动态计算 |
| 普通 useQuery refetch | 页面切换/返回时 | 返回 BuildPage/Commit 列表查询 builds.status ResolveField 动态计算 |
| isBuildStale 兜底 | 每次 status 时判断 | Running 超过 5h 小时标记 Failed 并落库 |

#### 7.3.3 状态不一致边界场景

| 场景 | build.status DB 值 | 前端展示 | 恢复方式 |
|---|---|---|---|
| Kafka CREATE_PR_REQ 发送失败（被 catch 吞掉） | Running | 永久 Running 5 秒轮询 → 5h 后 isBuildStale 自动标记 Failed |
| git-sync-manager 处理中崩溃，还没回传结果 | Running | Running | 同上 |
| CREATE_PR_SUCCESS_TOPIC 消息送达但 onCreatePRSuccess 内部异常 | Failed（catch 分支显式调用 `updateBuildStatuses(Failed)` 并落库，见 [build.service.ts:L905-L909](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/packages/amplication-server/src/core/build/build.service.ts#L905-L909)） | Failed 下次查询直接展示 | ✅ 正常失败反馈，无需额外恢复 |
| 前端长时间不查询不刷新 | Running/Unknown | calcBuildStatus 重算 |


