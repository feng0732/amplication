# Template Marketplace 发现与安装机制代码解析

本文档基于代码分析，详细说明 Amplication 中模板（Service Template）的发现、列表展示、筛选参数，以及安装到工作区（创建资源）的完整流程。

---

## 一、整体架构概览

模板系统涉及前端（`packages/amplication-client`）和后端（`packages/amplication-server`）两个主要部分：

| 层级 | 关键目录/文件 | 功能 |
|------|--------------|------|
| 前端 UI 组件 | [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx)、[CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx) | 模板下拉选择、创建资源表单 |
| 前端 Hooks | [useAvailableServiceTemplates.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useAvailableServiceTemplates.ts)、[useServiceTemplate.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts)、[useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts) | 查询模板列表、创建资源 |
| 前端 GraphQL 查询 | [serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts)、[resourcesQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts) | GraphQL 语句定义 |
| 后端 Resolver | [serviceTemplate.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.resolver.ts) | GraphQL 接口层（Query & Mutation） |
| 后端 Service | [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | 核心业务逻辑 |
| 后端 DTO | [FindAvailableTemplatesForProjectArgs.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/FindAvailableTemplatesForProjectArgs.ts)、[ResourceFromTemplateCreateInput.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/ResourceFromTemplateCreateInput.ts) | 请求参数定义 |

---

## 二、模板类型与数据模型

### 2.1 资源类型枚举

模板本质上是一种特殊的 `Resource`，其 `resourceType` 为 `ServiceTemplate`。在 [EnumResourceType.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/EnumResourceType.ts#L1-L12) 中定义：

```typescript
export enum EnumResourceType {
  Service = "Service",
  ProjectConfiguration = "ProjectConfiguration",
  MessageBroker = "MessageBroker",
  PluginRepository = "PluginRepository",
  ServiceTemplate = "ServiceTemplate",   // 模板类型
  Component = "Component",
}
```

### 2.2 模板与 Blueprint 的关系

每个模板必须关联一个 `Blueprint`（蓝图），Blueprint 决定了该模板创建出来的资源类型（Service 或 Component）以及对应的属性结构。在安装模板时，会通过 `template.blueprintId` 查找 Blueprint 并验证其 `enabled` 状态。

---

## 三、模板列表发现机制

模板列表有 **两种** 获取方式，对应不同的使用场景。

### 3.1 项目内模板列表：`serviceTemplates`

对应后端 Resolver 中的 Query `serviceTemplates`，用于获取 **当前项目内** 所有模板。

**前端查询**：[serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts#L15-L40)

```graphql
query getServiceTemplates($projectId: String!, $whereName: StringFilter) {
  serviceTemplates(
    where: { project: { id: $projectId }, name: $whereName }
    orderBy: [{ resourceType: Asc }, { createdAt: Desc }]
  ) {
    id name description ... version { id version message }
  }
}
```

**后端实现**：[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L181-L194)

```typescript
async serviceTemplates(args: FindManyResourceArgs): Promise<Resource[]> {
  return this.resourceService.resources({
    ...args,
    where: {
      ...args.where,
      deletedAt: null,
      archived: { not: true },
      resourceType: { equals: EnumResourceType.ServiceTemplate },
    },
  });
}
```

**筛选参数**（通过 `FindManyResourceArgs` 传递）：
- `where.project.id`：项目 ID（必填，通过权限校验）
- `where.name`：模板名称模糊匹配（可选）
- `orderBy`：排序规则
- `skip` / `take`：分页

---

### 3.2 项目可用模板列表：`availableTemplatesForProject`

这是 **模板市场的核心接口**，用于获取「**当前项目 + 当前工作区内所有公开项目**」的可用模板。只有已发布（有版本号）的模板才会被返回。

**前端查询**：[serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts#L65-L100)

```graphql
query availableTemplatesForProject($projectId: String!) {
  availableTemplatesForProject(
    where: { id: $projectId }
    orderBy: [{ name: Asc }]
  ) {
    id name description ... blueprint { id name enabled }
    project { id name } version { id version message }
  }
}
```

**后端实现**：[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L205-L239)

```typescript
async availableServiceTemplatesForProject(
  args: FindAvailableTemplatesForProjectArgs,
  user: User
): Promise<Resource[]> {
  const workspaceId = user.workspace.id;

  // 1. 找出当前工作区内所有公开的项目
  const publicProjects = await this.projectService.findProjects({
    where: {
      workspace: { id: workspaceId },
      platformIsPublic: { equals: true },
    },
  });

  // 2. 查询：当前项目 + 所有公开项目 中的模板
  return this.prisma.resource.findMany({
    ...args,
    where: {
      projectId: {
        in: [...publicProjects.map((p) => p.id), args.where.id],
      },
      deletedAt: null,
      archived: { not: true },
      resourceType: { equals: EnumResourceType.ServiceTemplate },
      resourceVersions: { some: {} },  // 关键：只返回有版本的（已发布的）
    },
  });
}
```

#### 3.2.1 筛选参数详解（`FindAvailableTemplatesForProjectArgs`）

定义见 [FindAvailableTemplatesForProjectArgs.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/FindAvailableTemplatesForProjectArgs.ts#L1-L18)：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `where.id` | `String` | 是 | 当前项目 ID（用于权限校验和范围包含） |
| `orderBy` | `ResourceOrderByInput[]` | 否 | 排序字段数组，如 `[{ name: Asc }]` |
| `skip` | `Int` | 否 | 跳过条数（分页） |
| `take` | `Int` | 否 | 取多少条（分页） |

**隐式过滤条件**（在 service 层硬编码）：
- `deletedAt: null` → 排除已删除的模板
- `archived: { not: true }` → 排除已归档的模板
- `resourceType: ServiceTemplate` → 只取模板类型资源
- `resourceVersions: { some: {} }` → **只返回有至少一个版本的模板**（即已发布）
- `projectId` 必须属于 `[当前项目ID, 工作区内所有公开项目ID]`

---

### 3.3 前端模板下拉选择：TemplateSelectField

前端在 [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx#L1-L32) 中实现模板下拉框：

```typescript
const TemplateSelectField = ({ projectId, ...rest }: Props) => {
  const { availableTemplates } = useAvailableServiceTemplates(projectId);

  const options = useMemo(() => {
    return availableTemplates
      .filter((serviceTemplate) => serviceTemplate.blueprint?.enabled || true)
      .map((serviceTemplate) => ({
        value: serviceTemplate.id,
        label: serviceTemplate.name,
        description: serviceTemplate.description,
        color: DEFAULT_COLOR,
      }));
  }, [availableTemplates]);

  return <SelectPanelField options={options} {...rest} />;
};
```

Hook `useAvailableServiceTemplates` 封装了查询逻辑，见 [useAvailableServiceTemplates.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useAvailableServiceTemplates.ts#L1-L31)。

---

## 四、模板安装（创建资源）流程

安装模板本质就是调用 `createResourceFromTemplate` Mutation，从模板生成一个新的 Service 或 Component 资源。

### 4.1 前端调用入口

在 [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx#L218-L260) 的 `handleSubmit` 中：

```typescript
const handleSubmit = useCallback((values: CreateResourceFormType) => {
  // ... 分离 Git 设置和其他属性 ...
  const { templateId, ...rest } = sanitizedValues;
  // ... 构造 preparedValues ...

  // 调用 createResource，templateId 不为空时走模板创建分支
  createResource(preparedValues, templateId, properties, settings);
}, [createResource]);
```

在 [useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts#L43-L93) 中根据是否有 `templateId` 走不同分支：

```typescript
const createResource = (data, templateId, catalogProperties, settings) => {
  if (templateId) {
    // 走模板创建分支
    createServiceFromTemplateInternal({
      variables: {
        data: {
          name: data.name,
          description: data.description,
          gitRepository: data.gitRepository,
          project: data.project,
          serviceTemplate: { id: templateId },
        },
      },
    }).then(/* 保存额外设置 */);
  } else {
    // 走普通 Component 创建分支
    createComponentInternal(/* ... */);
  }
};
```

### 4.2 前端 GraphQL Mutation

定义在 [resourcesQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts#L207-L215)：

```graphql
mutation createServiceFromTemplate($data: ResourceFromTemplateCreateInput!) {
  createResourceFromTemplate(data: $data) {
    id name description
  }
}
```

### 4.3 安装请求参数（`ResourceFromTemplateCreateInput`）

定义见 [ResourceFromTemplateCreateInput.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/ResourceFromTemplateCreateInput.ts#L1-L30)：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | `String` | 是 | 新资源名称 |
| `description` | `String` | 是 | 新资源描述 |
| `project.connect.id` | `String` | 是 | 目标项目 ID（安装到哪个项目） |
| `serviceTemplate.id` | `String` | 是 | 选用的模板 ID |
| `gitRepository` | `ConnectGitRepositoryInput` | 否 | Git 仓库设置（可选） |
| `buildAfterCreation` | `Boolean` | 否 | 创建后是否自动提交构建 |

### 4.4 后端安装流程详解

后端核心逻辑在 [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L242-L371) 的 `createResourceFromTemplate` 方法中，整个流程分为以下 8 个步骤：

```
Step 1: 权限与模板可用性校验
   ↓
Step 2: 获取模板最新版本
   ↓
Step 3: 获取并校验 Blueprint
   ↓
Step 4: 根据 Blueprint 类型分发（Service / Component）
   ↓
Step 5: 内部创建资源（拷贝 serviceSettings、替换路径占位符等）
   ↓
Step 6: 记录资源与模板版本的关联
   ↓
Step 7: 拷贝模板上的所有插件安装
   ↓
Step 8: 可选：自动提交构建 + 埋点
```

#### 4.4.1 Step 1：权限与模板可用性校验

```typescript
// 复用 availableServiceTemplatesForProject，确保模板对用户可见
const serviceTemplates = await this.availableServiceTemplatesForProject({
  where: { id: args.data.project.connect.id },
}, user);

const template = serviceTemplates.find(
  (t) => t.id === args.data.serviceTemplate.id
);

if (template === undefined) {
  throw new AmplicationError(`Service template not found`);
}
```

**关键点**：不是直接根据 `templateId` 查数据库，而是先通过「可见模板列表」过滤，确保用户无法使用不可见的模板。

#### 4.4.2 Step 2：获取模板最新版本

```typescript
const templateVersion = await this.resourceVersionService.getLatest(template.id);
if (!templateVersion) {
  throw new AmplicationError(`Template version not found`);
}
```

确保模板有已发布的版本（与列表查询的 `resourceVersions: { some: {} }` 双重保护）。

#### 4.4.3 Step 3：获取并校验 Blueprint

```typescript
const blueprint = await this.prisma.blueprint.findUnique({
  where: { id: template.blueprintId },
});

if (!blueprint || !blueprint.enabled) {
  throw new AmplicationError(`The template is missing a blueprint`);
}

const resourceType = blueprint.resourceType as EnumResourceType;
// 只支持 Service 和 Component 两种类型
if (![EnumResourceType.Component, EnumResourceType.Service].includes(resourceType)) {
  throw new AmplicationError(`Unsupported resource type`);
}
```

#### 4.4.4 Step 4 & 5：内部创建资源（两种分支）

根据 Blueprint 的 `resourceType` 调用不同的内部函数：

- **Service 类型** → `internalCreateServiceFromTemplate`
- **Component 类型** → `internalCreateComponentFromTemplate`

**Service 分支**（[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L373-L426)）做了这些事情：

```typescript
private async internalCreateServiceFromTemplate(args, template, user) {
  // 1. 读取模板的 serviceSettings
  const serviceSettings = await this.serviceSettingsService
    .getServiceSettingsValues({ where: { id: template.id } }, user);

  delete serviceSettings.resourceId;
  const kebabCaseServiceName = kebabCase(args.data.name);

  // 2. 替换路径占位符 {{SERVICE_NAME}} → kebab-case 的资源名
  serviceSettings.adminUISettings.adminUIPath =
    serviceSettings.adminUISettings.adminUIPath.replace(
      "{{SERVICE_NAME}}", kebabCaseServiceName
    );
  serviceSettings.serverSettings.serverPath =
    serviceSettings.serverSettings.serverPath.replace(
      "{{SERVICE_NAME}}", kebabCaseServiceName
    );

  // 3. 调用 resourceService.createService 创建新 Service
  const newService = await this.resourceService.createService({
    data: {
      blueprint: { connect: { id: template.blueprintId } },
      name: args.data.name,
      description: args.data.description,
      gitRepository: args.data.gitRepository,
      resourceType: EnumResourceType.Service,
      project: args.data.project,
      serviceSettings: serviceSettings,
      codeGenerator: template.codeGeneratorName
        ? CODE_GENERATOR_NAME_TO_ENUM[template.codeGeneratorName]
        : EnumCodeGenerator.NodeJs,
    },
  }, user);

  return newService;
}
```

**Component 分支**（[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L428-L452)）相对简单：

```typescript
private async internalCreateComponentFromTemplate(args, template, user) {
  return this.resourceService.createComponent({
    data: {
      name: args.data.name,
      description: args.data.description,
      gitRepository: args.data.gitRepository,
      resourceType: EnumResourceType.Component,
      blueprint: { connect: { id: template.blueprintId } },
      project: args.data.project,
    },
  }, user);
}
```

#### 4.4.5 Step 6：记录模板版本关联

通过 [ResourceTemplateVersionService](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts#L51-L88) 在新资源上挂一个 `ResourceTemplateVersion` Block：

```typescript
await this.resourceTemplateVersionService.updateResourceTemplateVersion({
  where: { id: newResource.id },
  data: {
    serviceTemplateId: template.id,
    version: templateVersion.version,
  },
}, user);
```

这个关联用于后续的模板版本升级（`upgradeServiceToLatestTemplateVersion`）。

#### 4.4.6 Step 7：拷贝插件安装

将模板上的所有 `PluginInstallation` 复制到新资源（[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L454-L490)）：

```typescript
async copyPluginInstallations(sourceResourceId, targetResourceId, user) {
  const plugins = await this.pluginInstallationService
    .getOrderedPluginInstallations(sourceResourceId);

  for (const plugin of plugins) {
    const createInput: PluginInstallationCreateInput = {
      pluginId: plugin.pluginId,
      enabled: plugin.enabled,
      npm: plugin.npm,
      version: plugin.version,
      displayName: plugin.displayName,
      isPrivate: plugin.isPrivate ?? false,
      settings: plugin.settings,
      configurations: plugin.configurations,
      resource: { connect: { id: targetResourceId } },
    };
    await this.pluginInstallationService.create({ data: createInput }, user);
  }
}
```

#### 4.4.7 Step 8：自动提交 & 埋点

```typescript
if (args.data.buildAfterCreation) {
  await this.projectService.commit({
    data: {
      message: "Create resource from template",
      project: { connect: { id: newResource.projectId } },
      resourceTypeGroup: EnumResourceTypeGroup.Services,
      commitStrategy: EnumCommitStrategy.Specific,
      resourceIds: [newResource.id],
      user: { connect: { id: user.id } },
    },
  }, user);
}

await this.analyticsService.trackWithContext({
  event: EnumEventType.CreateResourceFromTemplate,
  properties: {
    templateName: template.name,
    resourceName: newResource.name,
  },
});
```

---

## 五、完整数据流图

```
┌──────────────────────────────────────────────────────────────┐
│                     前端 (amplication-client)                 │
│                                                              │
│  TemplateSelectField                                         │
│       ↓ useAvailableServiceTemplates()                       │
│       ↓ GET_AVAILABLE_TEMPLATES_FOR_PROJECT                  │
│  用户选择模板 → CreateResourceForm → handleSubmit()           │
│                                        ↓                      │
│                              useCreateResource.createResource()
│                                        ↓                      │
│                              CREATE_SERVICE_FROM_TEMPLATE     │
└──────────────────────────────┬───────────────────────────────┘
                               │ GraphQL
┌──────────────────────────────▼───────────────────────────────┐
│                     后端 (amplication-server)                 │
│                                                              │
│  serviceTemplate.resolver.ts                                  │
│    ├── Query: availableTemplatesForProject()                 │
│    │      ↓ serviceTemplate.service.ts                       │
│    │        availableServiceTemplatesForProject()            │
│    │          → 查公开项目 → 过滤有版本的模板                  │
│    │                                                         │
│    └── Mutation: createResourceFromTemplate()                │
│           ↓ serviceTemplate.service.ts                       │
│             createResourceFromTemplate()                     │
│               Step1: 校验模板可见性                            │
│               Step2: 获取模板最新版本                          │
│               Step3: 校验 Blueprint                           │
│               Step4: 根据类型分发                              │
│               Step5: internalCreateServiceFromTemplate /     │
│                      internalCreateComponentFromTemplate     │
│                      → 替换 {{SERVICE_NAME}} 路径占位符       │
│                      → 拷贝 serviceSettings                   │
│               Step6: 写 ResourceTemplateVersion Block        │
│               Step7: copyPluginInstallations()               │
│               Step8: commit + analytics                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 六、关键文件索引

| 文件 | 作用 |
|------|------|
| [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | 模板核心业务逻辑（发现、创建、升级、插件拷贝） |
| [serviceTemplate.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.resolver.ts) | GraphQL 接口定义 |
| [resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) | 管理资源与模板版本的关联 Block |
| [FindAvailableTemplatesForProjectArgs.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/FindAvailableTemplatesForProjectArgs.ts) | 可用模板查询参数 |
| [ResourceFromTemplateCreateInput.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/ResourceFromTemplateCreateInput.ts) | 模板创建资源请求参数 |
| [EnumResourceType.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/dto/EnumResourceType.ts) | 资源类型枚举（含 ServiceTemplate） |
| [serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts) | 前端模板查询 GraphQL |
| [useAvailableServiceTemplates.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useAvailableServiceTemplates.ts) | 前端模板查询 Hook |
| [useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts) | 前端创建资源 Hook（模板分支） |
| [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx) | 创建资源表单 UI |
| [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx) | 模板下拉选择组件 |
