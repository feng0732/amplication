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

## 六（附）：依赖锁定与 Kafka 发送确认语义 —— 代码事实 vs 外部推断

> 本章专门梳理项目代码中**可直接确认的事实**与**对外部库 / Broker 行为的推断**，明确二者的边界，避免将推断误认为代码承诺。

### F.1 项目依赖锁定事实（可从代码/配置直接确认）

#### F.1.1 package.json 声明的版本范围

[package.json:L60](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/package.json#L60) 与 [package.json:L126](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/package.json#L126)：

```json
"@nestjs/microservices": "^9.3.9",
"kafkajs": "^2.2.4",
```

两者均使用 caret 范围（`^`），理论上允许 minor 级升级。

#### F.1.2 package-lock.json 实际锁定的精确版本

[package-lock.json:L9985-L9992](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/package-lock.json#L9985-L9992) 与 [package-lock.json:L42947-L42953](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/package-lock.json#L42947-L42953)：

| 包名 | 锁定版本 | integrity 校验 |
|---|---|---|
| `@nestjs/microservices` | **9.4.3** | `sha512-piMw8d3C4ppc5St5AhQEtecMhyeBK2Q1VYk4AL3NKtG6U0fzz/6KLiETpWdKXmazeI/m7qac2upOvwmRzle0aA==` |
| `kafkajs` | **2.2.4** | `sha512-j/YeapB1vfPT2iOIUn/vxdyKEuhuY2PxMBvf5JWux6iSaukAccrMtXEY/Lb7OvavDhOWME589bpLrEdnVHjfjA==` |

> **事实**：上述两个版本已被 package-lock.json 的 integrity hash 精确锁定，`npm install` 在未修改锁文件的前提下会使用完全相同的版本。

---

### F.2 Kafka 发送确认语义 —— 代码实现事实（项目代码中可直接确认）

以下内容均可在本项目代码仓库中找到直接证据，不依赖任何外部假设。

#### F.2.1 KafkaProducerService.emitMessage() 的代码行为

[KafkaProducer.service.ts:L22-L38](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts#L22-L38)

| 编号 | 代码事实 | 依据 |
|---|---|---|
| F-1 | 方法签名为 `async emitMessage(topic, message, schemaIds?): Promise<void>`，返回值类型为 `void`，**不包含任何 Kafka 返回元数据**（offset、partition、timestamp 等均被丢弃）。 | 第 22-26 行参数与返回类型；第 33-35 行 `resolve()` 不传值 |
| F-2 | 内部通过 RxJS Observable 的 `subscribe` 将 NestJS `ClientKafka.emit()` 的结果适配为 Promise：`next` 回调 → `resolve()`；`error` 回调 → `reject(err)`；`complete` 回调未订阅，不影响 Promise 状态。 | 第 28-37 行 |
| F-3 | 方法体内**不包含任何重试、超时、补偿、幂等保护**等可靠性增强逻辑。 | 方法体仅包含 serializer + Observable 包装，共 11 行可执行代码 |

#### F.2.2 createNestjsKafkaConfig 的配置事实

[createNestjsKafkaConfig.ts:L8-L34](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts#L8-L34)

| 编号 | 代码事实 | 依据 |
|---|---|---|
| F-4 | 仅显式配置了 `options.client`（`brokers`、`clientId`、`ssl`、`sasl`）和 `options.consumer`（`groupId`、`sessionTimeout`、`rebalanceTimeout`、`heartbeatInterval`、`maxBytesPerPartition`）。 | 第 23-33 行 options 对象字面量 |
| F-5 | **完全未配置任何 producer 级别的参数**，包括但不限于：`acks`、`retry`、`idempotent`、`maxInFlightRequests`、`allowAutoTopicCreation`。 | options 对象中不存在 producer 键 |
| F-6 | 未涉及 broker 端参数（`min.insync.replicas`、`flush.ms` 等本身也不属于客户端配置范畴）。 | — |

#### F.2.3 KafkaModule 注册事实

[Kafka.module.ts:L13-L30](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/Kafka.module.ts#L13-L30)

| 编号 | 代码事实 | 依据 |
|---|---|---|
| F-7 | 通过 `ClientsModule.registerAsync([{ name: KAFKA_CLIENT, useFactory: createNestjsKafkaConfig }])` 注册 NestJS 的 `ClientKafka`。 | 第 15-19 行 |
| F-8 | 未对 `ClientKafka` 做任何自定义子类化、装饰器包装或方法重写；`KafkaProducerService` 直接注入 NestJS 提供的 `ClientKafka` 实例。 | 第 16-17 行 `@Inject(KAFKA_CLIENT) private readonly kafkaClient: ClientKafka` |

#### F.2.4 测试代码中的行为假设

[KafkaProducer.service.spec.ts:L24-L28](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.spec.ts#L24-L28) 与 [KafkaProducer.service.spec.ts:L72-L78](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.spec.ts#L72-L78)

| 编号 | 代码事实 | 依据 |
|---|---|---|
| F-9 | 测试用例通过 `jest.fn().mockReturnValue(of(null))` mock `ClientKafka.emit()`，其中 `of(null)` 是 RxJS 的同步 Observable 创建函数，会立即触发 `next(null)` 然后 `complete()`。 | 第 27 行；第 73 行 |
| F-10 | 测试断言 `emitMessage()` 在 `emit` 返回 Observable.next 后 Promise resolve，说明测试作者对 `emit()` 的语义假设为"至少触发一次 next 即视为发送成功"。 | 第 72-78 行测试用例 |

---

### F.3 Kafka 发送确认语义 —— 对外部依赖行为的推断

> ⚠️ **本节内容全部属于对外部库（NestJS / kafkajs / Kafka Broker）行为的推断**，这些行为不由本项目代码控制，可能随版本升级或部署环境变化而改变。将其列出仅为帮助理解运行时可能发生的行为，**不构成任何代码层面的契约或保证**。

| 编号 | 推断内容 | 推断依据 | 不确定性边界 |
|---|---|---|---|
| I-1 | NestJS 9.4.3 的 `ClientKafka.emit(topic, message)` 内部会调用 `kafkajs.producer.send()`，并将返回的 Promise 适配为 Observable：Promise resolve → Observable.next + complete；Promise reject → Observable.error。 | `@nestjs/microservices` 9.4.3 公开文档与源码实现惯例；测试 F-9 / F-10 中 `of(null)` 的写法也与该语义一致。 | NestJS 内部实现未被 vendored 进本项目，无法从仓库代码直接验证；若未来升级 NestJS 版本且其修改了 Observable 适配策略，本项目无代码层感知手段。 |
| I-2 | kafkajs 2.2.4 的 `producer.send()` 默认 `acks=-1`（即 `all`），表示生产者需等待所有当前 ISR 副本的确认后 Promise 才会 resolve；默认 `timeout=30000`。 | kafkajs 2.2.4 官方文档（kafka.js.org/docs/producing）中 `send` 方法参数表格声明。 | kafkajs 文档未附带版本化的变更追踪；即使文档描述正确，若运行时通过中间件、自定义 transport 或 broker 端 `message.timestamp.type` 等配置间接影响确认语义，生产者侧无法感知。 |
| I-3 | `acks=all` 仅表示 Broker 将消息写入 ISR 副本的 OS page cache，不等价于写入物理磁盘。是否落盘由 broker 端 `flush.ms` / `flush.messages` / OS 刷盘策略共同决定。 | Apache Kafka 官方架构文档与业界共识。 | 属于 Kafka 分布式系统本身的可靠性边界，与本项目代码无关；不同部署环境（云厂商托管 Kafka vs 自建集群）的落盘策略差异极大。 |
| I-4 | `acks=all` 的实际生效还受 broker 端 Topic 配置 `min.insync.replicas` 制约，若 ISR 大小低于该阈值，Broker 会返回 `NOT_ENOUGH_REPLICAS` 错误并使 producer.send() reject。 | Apache Kafka 官方文档与 kafkajs 错误码列表。 | `min.insync.replicas` 为 broker/Topic 级配置，客户端代码不可见也无法控制；本项目无法验证其在目标部署环境中的取值。 |
| I-5 | 即使 acks=-1 成功确认，极端故障场景（所有 ISR 副本所在机器同时掉电且 page cache 未刷盘）下数据仍可能丢失。 | Apache Kafka 架构文档。 | 属于分布式系统通用可靠性边界，与本项目代码无关。 |

---

### F.4 事实与推断的边界总结

| 维度 | 代码事实（仓库内可验证） | 外部推断（依赖运行时环境） |
|---|---|---|
| **依赖版本** | `@nestjs/microservices` **9.4.3**、`kafkajs` **2.2.4**（package-lock.json integrity 精确锁定） | — |
| **Producer 配置** | 完全未配置 `acks` / `retry` / `idempotent` 等参数 | 实际取值依赖 kafkajs 默认值（推断 I-2） |
| **emitMessage 返回值** | `Promise<void>`，不含 Kafka 元数据 | — |
| **Promise resolve 触发** | Observable.next → resolve；Observable.error → reject（F-2） | Observable.next 对应 kafkajs Promise.resolve → 对应 Broker 已返回 acks 响应（推断 I-1 + I-2） |
| **数据持久化** | 项目代码不包含任何持久化相关逻辑 | 依赖 Broker 配置与 OS 刷盘策略（推断 I-3 ~ I-5） |
| **Broker 端参数** | 代码中完全不可见 | `min.insync.replicas` / `flush.ms` 等由运维侧管理（推断 I-3、I-4） |

---

## 七、失败反馈边界分析

### 7.1 创建 PR 请求失败的边界情况

#### 7.1.1 Kafka 消息发送失败边界

> 关于 Kafka 发送确认语义的**代码事实与外部推断**的系统区分，详见第六章（附）：[六（附）：依赖锁定与 Kafka 发送确认语义 —— 代码事实 vs 外部推断](#六附依赖锁定与-kafka-发送确认语义--代码事实-vs-外部推断)。本节聚焦于失败场景的业务后果分析。

**KafkaProducerService.emitMessage() 的实现：

[KafkaProducer.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/42-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts)

```typescript
async emitMessage(topic, message, schemaIds) {
  const kafkaMessage = await this.serializer.serialize(message, schemaIds);
  return await new Promise((resolve, reject) => {
    this.kafkaClient.emit(topic, kafkaMessage).subscribe({
      error: (err) => reject(err),
      next: () => resolve(),
    });
  });
}
```

**边界 1：从业务角度看 `next` 回调的确认范围。**

结合第六章（附）F.2 节代码事实与 F.3 节推断，本项目代码可确认的最保守边界为：
- **代码事实（F-1、F-2）**：`next` → `Promise.resolve()`，返回值为 `void`，无 Kafka 元数据
- **代码事实（F-5）**：项目未配置 producer 级别的 `acks` / `retry` 等参数
- **外部推断（I-1 ~ I-5）**：基于 NestJS 9.4.3 + kafkajs 2.2.4 的运行时行为，`next` 通常表示 broker 已返回 acks 响应，但具体确认等级、持久化程度受外部环境制约

因此，在业务失败场景分析中，**仅在代码事实层面保守表述**：`next` 回调 resolve 仅表示 `emitMessage` 未抛异常；对于消息是否已被 Broker 持久化，不做超出代码事实的任何保证。

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
| emitMessage 抛异常（Kafka 不可达、broker 拒绝等） | 仅打印 logger.error，PUSH_TO_GIT 步骤保持 Running，build.gitStatus 保持 Waiting | ❌ Build 状态永不结束，前端轮询永远 Running，直到 5h 后被 isBuildStale 标记 Failed |
| emitMessage 的 next 回调 resolve 后，消息在后续环节丢失（如 ISR 全部掉电未刷盘、consumer 处理前消息过期被清理等） | 消息丢失，同上 | 同上 |
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


