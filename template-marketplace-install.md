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

## 六、模板选中后 Blueprint 切换机制及对表单的影响

### 6.1 切换流程

用户在 [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx#L15-L30) 下拉框中选择模板后，事件回调 `onChange(templateId)` 触发 [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx#L337-L352) 中的 `handleTemplateChange`：

```typescript
const handleTemplateChange = useCallback(
  (templateId: string) => {
    if (templateId) {
      // 从已加载的 availableTemplates 中查找模板对象
      const template = availableTemplates.find(
        (template) => template.id === templateId
      );

      if (template) {
        // 关键：用模板关联的 blueprintId 驱动表单切换
        handleBlueprintChange(template.blueprintId, templateId);
      }
    } else {
      handleBlueprintChange(undefined, undefined);
    }
  },
  [availableTemplates, handleBlueprintChange]
);
```

`handleTemplateChange` 不直接修改表单，而是提取模板的 `blueprintId` 交给 `handleBlueprintChange` 统一处理。这意味着**模板本身不决定表单字段，真正的控制器是 Blueprint**。模板只是 Blueprint 的一个"载体"或"预设包"。

### 6.2 Blueprint 切换对表单状态的重置

`handleBlueprintChange` 定义在 [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx#L306-L334)：

```typescript
const handleBlueprintChange = useCallback(
  (blueprintId: string, templateId?: string) => {
    let settingsInitialValue = { properties: {} };

    if (blueprintId) {
      const blueprint = blueprintsMapById[blueprintId];
      if (blueprint) {
        // 根据 Blueprint 定义的属性，生成初始空值对象
        settingsInitialValue = {
          properties: blueprint.properties.reduce((acc, property) => {
            acc[property.key] = "";  // 每个属性默认空字符串
            return acc;
          }, {}),
        };
      }
    }

    // 用 Formik 的 enableReinitialize 机制整体重置表单
    setInitialValueWithSettings({
      ...initialValue,
      settings: settingsInitialValue,                 // 重置 Blueprint 级别的设置
      blueprint: { connect: { id: blueprintId || "" } },
      templateId: templateId || "",
    });
  },
  [blueprintsMapById, initialValue]
);
```

### 6.3 对表单的具体影响

切换 Blueprint（通过模板或直接选 Blueprint）会触发以下连锁变化：

| 变化点 | 影响 | 来源 |
|--------|------|------|
| `settings.properties` 重置 | Blueprint 级别的资源设置清空并按新 Blueprint 的属性重建 | `handleBlueprintChange` 中 `settingsInitialValue` |
| `blueprint.connect.id` 更新 | 表单中 Blueprint 字段更新 | `setInitialValueWithSettings` |
| `templateId` 联动更新 | 如果是通过模板切换则记录 templateId，直接选 Blueprint 则清空 | `handleBlueprintChange` 第二个参数 |
| Blueprint 选择框禁用状态 | 选中模板后 BlueprintSelectField 的 `disabled={!!formik.values.templateId}` 生效，防止用户手动改变 Blueprint | [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx#L424-L431) |
| 动态渲染 Blueprint 设置面板 | `CreateResourceFormResourceSettings` 根据 `blueprintId` 渲染对应的 `ResourceSettingsFormFields` | [CreateResourceFormResourceSettings.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceFormResourceSettings.tsx#L21-L64) |
| settings.properties 再次清空 | 切换 Blueprint 时 useEffect 再次兜底清空设置字段（双重保险） | [CreateResourceFormResourceSettings.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceFormResourceSettings.tsx#L30-L35) |
| 校验 schema 动态变更 | `getValidationSchema(blueprintId)` 根据新 Blueprint 的属性结构重建 JSON Schema | [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx#L182-L209) |

Blueprint 设置面板的条件渲染逻辑：

```typescript
// 只有当 blueprint 存在、且定义了 properties，才会渲染配置面板
return (
  blueprint &&
  blueprint.properties &&
  blueprint.properties.length > 0 && (
    <div>
      <Text textStyle={EnumTextStyle.H4}>
        {blueprint?.name} Configuration
      </Text>
      <Panel panelStyle={EnumPanelStyle.Bordered}>
        <ResourceSettingsFormFields
          blueprintId={blueprintId}
          fieldNamePrefix="settings."
        />
      </Panel>
    </div>
  )
);
```

---

## 七、创建成功后自定义属性与资源设置的单独保存机制

模板创建资源的 Mutation（`createResourceFromTemplate`）只接收 `name / description / project / serviceTemplate / gitRepository` 这几个基础字段，**并不包含** `properties`（Catalog 自定义属性）和 `settings`（Blueprint 资源设置）。这两部分数据在创建成功后通过**两次独立的 Mutation** 单独保存。

### 7.1 保存流程总览

在 [useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts#L43-L93) 的 `createResource` 函数中：

```typescript
const createResource = (
  data: models.ResourceCreateInput,
  templateId: string,
  catalogProperties?: Record<string, any>,   // 全局自定义属性
  settings?: models.ResourceSettingsUpdateInput  // Blueprint 级资源设置
) => {
  if (templateId) {
    createServiceFromTemplateInternal({
      variables: {
        data: {
          name: data.name,
          description: data.description,
          gitRepository: data.gitRepository,
          project: data.project,
          serviceTemplate: { id: templateId },
          // 注意：这里没有传 properties 和 settings！
        },
      },
    })
      .then(async (result) => {
        if (result.data?.createResourceFromTemplate) {
          // ===== 创建成功后，再单独保存扩展属性 =====
          await saveResourceSettings(
            result.data.createResourceFromTemplate,
            catalogProperties,
            settings
          );
          onResourceCreated &&
            onResourceCreated(result.data.createResourceFromTemplate);
        }
      })
      .catch(console.error);
  }
  // ... 普通 Component 分支逻辑相同
};
```

### 7.2 saveResourceSettings：并行写入两个独立端点

[useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts#L95-L127) 中定义了保存逻辑：

```typescript
const saveResourceSettings = async (
  resource: models.Resource,
  catalogProperties?: Record<string, any>,
  settings?: models.ResourceSettingsUpdateInput
) => {
  const promises = [];

  // 第一路：保存 Catalog 自定义属性（全局级）
  if (catalogProperties && Object.keys(catalogProperties).length > 0) {
    promises.push(
      updateResource({
        variables: {
          data: { properties: catalogProperties },  // 写到 resource.properties
          resourceId: resource.id,
        },
      })
    );
  }

  // 第二路：保存 Blueprint 资源设置（Blueprint 级）
  if (settings && Object.keys(settings).length > 0) {
    promises.push(
      updateResourceSettings({
        variables: {
          data: { ...settings },                     // 写到独立的 ResourceSettings entity
          resourceId: resource.id,
        },
      })
    );
  }

  return Promise.all(promises);   // 两个请求并行发出
};
```

### 7.3 两条保存路径的差异

| 维度 | 自定义属性（catalogProperties） | 资源设置（settings） |
|------|------------------------------|---------------------|
| 存储位置 | `Resource.properties`（JSON 字段） | 独立的 `ResourceSettings` entity，通过外键关联 Resource |
| GraphQL Mutation | `UPDATE_RESOURCE` → `updateResource` | `UPDATE_RESOURCE_SETTINGS` → `updateResourceSettings` |
| Query 定义位置 | [resourcesQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts#L186-L205) | [resourceSettingsQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ResourceSettings/queries/resourceSettingsQueries.ts#L1-L13) |
| 触发条件 | `catalogProperties` 不为空对象 | `settings` 不为空对象 |
| 表单来源 | `CustomPropertiesFormFields`（工作区级全局自定义属性） | `ResourceSettingsFormFields`（当前 Blueprint 定义的属性） |
| 数据结构 | `Record<string, any>` 扁平键值对 | `ResourceSettingsUpdateInput { properties: Record<string, any> }` 嵌套在 properties 字段下 |

### 7.4 为什么不合并到一次创建请求？

从设计上看，原因有二：

1. **职责分离**：`createResourceFromTemplate` 的后端逻辑只关注模板本身（复制 serviceSettings、拷贝插件、记录版本关联等），不处理可变的自定义属性。
2. **创建时序**：Resource 必须先存在（拿到 `resource.id`），才能更新其 `properties` 或创建关联的 `ResourceSettings` 记录——后者依赖前者的主键。

---

## 八、禁用 Blueprint 的模板为何前端仍显示及后端拒绝机制

### 8.1 前端显示问题：TemplateSelectField 中的过滤 Bug

在 [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx#L18-L27) 中：

```typescript
const options = useMemo(() => {
  return availableTemplates
    // ⚠️ Bug：`|| true` 让整个过滤条件永远返回 true
    .filter((serviceTemplate) => serviceTemplate.blueprint?.enabled || true)
    .map((serviceTemplate) => ({
      value: serviceTemplate.id,
      label: serviceTemplate.name,
      description: serviceTemplate.description,
      color: DEFAULT_COLOR,
    }));
}, [availableTemplates]);
```

逻辑解析：
- 如果 `blueprint?.enabled === true` → `true || true` → **显示**（正确）
- 如果 `blueprint?.enabled === false` → `false || true` → **显示**（错误，本应过滤掉）
- 如果 `blueprint` 为 `null/undefined` → `undefined || true` → **显示**（也不符合预期）

因此无论 Blueprint 是否启用，模板都会出现在下拉列表中。

### 8.2 对比：BlueprintSelectField 的正确实现

作为参照，[BlueprintSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Blueprints/BlueprintSelectField.tsx#L19-L30) 对 Blueprint 自身做了正确过滤：

```typescript
const options = useMemo(() => {
  return findBlueprintsData?.blueprints
    .filter((blueprint) => blueprint.enabled)   // ✅ 没有 || true，只保留启用的
    .map((blueprint) => ({
      value: useKeyAsValue ? blueprint.key : blueprint.id,
      label: blueprint.name,
      enabled: blueprint.enabled,
      description: blueprint.description,
      color: blueprint.color || resourceThemeMap[EnumResourceType.Component].color,
    }));
}, [findBlueprintsData?.blueprints, useKeyAsValue]);
```

这就是为什么**直接选 Blueprint 时看不到禁用的，但通过模板选时可以看到**。两边的过滤策略不一致。

### 8.3 后端拒绝机制：Step 3 中的双重校验

即便前端绕过了过滤（或因 Bug 显示了禁用模板），后端在 [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L276-L290) 的 `createResourceFromTemplate` Step 3 会做严格校验：

```typescript
const blueprint = await this.prisma.blueprint.findUnique({
  where: { id: template.blueprintId },
});

if (!blueprint) {
  throw new AmplicationError(`The template is missing a blueprint`);
}

if (!blueprint.enabled) {
  throw new AmplicationError(
    `The selected template is based on a disabled blueprint.`
  );
}
```

后端检查的时序与错误码：

| 校验条件 | 失败时抛出的错误信息 | 发生在 Step |
|---------|-------------------|------------|
| 模板在可见范围内（来自 `availableServiceTemplatesForProject` 结果） | `Service template not found` | Step 1 |
| 模板有已发布的版本 | `Template version not found` | Step 2 |
| 模板关联的 Blueprint 存在 | `The template is missing a blueprint` | Step 3 |
| Blueprint 的 `enabled === true` | `The selected template is based on a disabled blueprint.` | Step 3 |
| Blueprint 的资源类型是 Service 或 Component | `The template is based on a blueprint with an unsupported resource type...` | Step 3 |

这种"前端宽松展示 + 后端严格拒绝"的模式虽然保证了数据安全，但会导致用户体验不佳——选了模板填完表单提交后才被告知无法使用。修复建议是移除 `TemplateSelectField.tsx` 中 filter 里的 `|| true`。

---

## 九、关键文件索引

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
| [useCreateResource.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/hooks/useCreateResource.ts) | 前端创建资源 Hook（模板分支 + saveResourceSettings 逻辑） |
| [CreateResourceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceForm.tsx) | 创建资源表单 UI（handleTemplateChange / handleBlueprintChange 逻辑） |
| [TemplateSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Components/TemplateSelectField.tsx) | 模板下拉选择组件（含 blueprint.enabled 过滤 Bug 所在） |
| [BlueprintSelectField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Blueprints/BlueprintSelectField.tsx) | Blueprint 下拉选择组件（正确过滤 blueprint.enabled） |
| [CreateResourceFormResourceSettings.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Resource/create-resource-page/CreateResourceFormResourceSettings.tsx) | Blueprint 切换时重置 settings.properties |
| [ResourceSettingsFormFields.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ResourceSettings/ResourceSettingsFormFields.tsx) | 动态渲染 Blueprint 级资源设置表单字段 |
| [CustomPropertiesFormFields.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/CustomProperties/CustomPropertiesFormFields.tsx) | 渲染工作区全局自定义属性表单字段 |
| [useResourceSettings.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ResourceSettings/hooks/useResourceSettings.tsx) | 资源设置查询/更新 Hook |
| [resourceSettingsQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/ResourceSettings/queries/resourceSettingsQueries.ts) | UPDATE_RESOURCE_SETTINGS Mutation |
| [resourcesQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Workspaces/queries/resourcesQueries.ts) | UPDATE_RESOURCE Mutation（保存自定义属性用） |
| [useBlueprints.tsx](file:///d:/fz/0601/solo-dogfeeding/code/54-amplication/packages/amplication-client/src/Blueprints/hooks/useBlueprints.tsx) | Blueprint 查询 Hook |
