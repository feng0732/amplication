# Import/Export 配置迁移代码链路梳理

本文档从代码实现角度梳理 Amplication 项目配置、模板快照和兼容校验的完整链路。

> 本文档所有路径均为仓库根目录下的相对路径，根目录为 `53-amplication/`

---

## 一、核心架构概览

整个 Import/Export 配置迁移系统围绕 **Block** 抽象模型构建。所有配置项（项目配置、服务设置、模板版本、代码引擎版本等）都以 Block 的形式存储，通过 BlockService 统一管理。

### Block 类型枚举
参考 [packages/amplication-server/src/enums/EnumBlockType.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/enums/EnumBlockType.ts#L3-L19)

```typescript
export enum EnumBlockType {
  ServiceSettings = "ServiceSettings",
  ProjectConfigurationSettings = "ProjectConfigurationSettings",
  Topic = "Topic",
  ServiceTopics = "ServiceTopics",
  PluginInstallation = "PluginInstallation",
  PluginOrder = "PluginOrder",
  Module = "Module",
  ModuleAction = "ModuleAction",
  ModuleDto = "ModuleDto",
  Package = "Package",
  PrivatePlugin = "PrivatePlugin",
  CodeEngineVersion = "CodeEngineVersion",
  Relation = "Relation",
  ResourceSettings = "ResourceSettings",
  ResourceTemplateVersion = "ResourceTemplateVersion",
}
```

---

## 二、项目配置（Project Configuration）

### 2.1 数据模型

项目配置通过 `ProjectConfigurationSettings` Block 存储，其 DTO 定义参考：
[packages/amplication-server/src/core/projectConfigurationSettings/dto/ProjectConfigurationSettings.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/dto/ProjectConfigurationSettings.ts#L1-L14)

```typescript
export class ProjectConfigurationSettings extends IBlock {
  baseDirectory!: string;          // 项目基础目录
  overrideCustomizableFilesInGit?: boolean;  // 是否覆盖 Git 中的可自定义文件
}
```

### 2.2 创建链路

项目配置的创建发生在项目创建时，完整调用链如下：

```
ProjectService.createProject()
    ↓
ResourceService.createProjectConfiguration()
    ↓
ProjectConfigurationSettingsService.createDefault()
    ↓
BlockService.create<ProjectConfigurationSettings>()
```

#### 1) 项目创建入口
[packages/amplication-server/src/core/project/project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.service.ts#L74-L118)

```typescript
async createProject(args: ProjectCreateArgs, userId: string): Promise<Project> {
  // 1. 计费限制校验
  // 2. 创建 Project 记录
  const project = await this.prisma.project.create({ ... });

  // 3. 创建项目配置资源（关键步骤）
  await this.resourceService.createProjectConfiguration(
    project.id,
    project.id,
    userId
  );

  return project;
}
```

#### 2) 创建 ProjectConfiguration 类型的 Resource
[packages/amplication-server/src/core/resource/resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L180-L206)

```typescript
async createProjectConfiguration(
  projectId: string,
  projectName: string,
  userId: string
): Promise<Resource> {
  // 检查是否已存在
  const existingProjectConfiguration = await this.prisma.resource.findFirst({
    where: { projectId, resourceType: EnumResourceType.ProjectConfiguration },
  });
  if (!isEmpty(existingProjectConfiguration)) {
    throw new ProjectConfigurationExistError();
  }

  // 创建 ProjectConfiguration Resource
  const newProjectConfiguration = await this.prisma.resource.create({
    data: {
      resourceType: EnumResourceType.ProjectConfiguration,
      name: projectName,
      project: { connect: { id: projectId } },
    },
  });

  // 创建默认的 ProjectConfigurationSettings Block
  await this.projectConfigurationSettingsService.createDefault(
    newProjectConfiguration.id,
    userId
  );
  return newProjectConfiguration;
}
```

#### 3) 创建默认配置 Block
[packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts#L17-L94)

默认配置常量：
```typescript
export const DEFAULT_PROJECT_CONFIGURATION_SETTINGS = {
  baseDirectory: "/",
  blockType: EnumBlockType.ProjectConfigurationSettings,
  description: "This block is used to store project configuration settings.",
  displayName: "Project Configuration Settings",
  overrideCustomizableFilesInGit: true,
};
```

### 2.3 更新与查询

- **更新**: `ProjectConfigurationSettingsService.update()` → `BlockService.update()`
- **查询**: `ProjectConfigurationSettingsService.findOne()` → `BlockService.findManyByBlockType()`

---

## 三、模板快照（Template Snapshot）

模板快照机制围绕以下几个核心服务协同工作：

| 服务 | 职责 | 文件 |
|------|------|------|
| ServiceTemplateService | 模板的 CRUD、从模板创建资源、模板升级 | [packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) |
| ResourceVersionService | 资源版本管理、版本差异对比 | [packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts) |
| ResourceTemplateVersionService | 记录资源使用的模板版本 | [packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) |
| TemplateCodeEngineVersionService | 记录模板使用的代码引擎版本历史 | [packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts) |
| PluginInstallationService | 插件安装配置及合并到目标资源 | [packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) |

### 3.1 模板创建流程

#### 方式一：直接创建服务模板
[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L57-L89)

```typescript
async createServiceTemplate(args: CreateServiceTemplateArgs, user: User): Promise<Resource> {
  const { serviceSettings, ...rest } = args.data.resource;

  // 1. 创建 ServiceTemplate 类型的 Resource
  const resource = await this.resourceService.createResource({
    data: { ...rest, resourceType: EnumResourceType.ServiceTemplate },
  }, user);

  // 2. 创建默认服务对象（角色、ServiceSettings）
  await this.resourceService.createServiceDefaultObjects(resource, user, false, serviceSettings);

  // 3. 安装插件
  if (args.data.plugins?.plugins) {
    await this.resourceService.installPlugins(resource.id, args.data.plugins.plugins, user);
  }

  return resource;
}
```

#### 方式二：从现有资源创建模板
[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L95-L179)

```typescript
async createTemplateFromExistingResource(args: CreateTemplateFromResourceArgs, user: User): Promise<Resource> {
  const resource = await this.resourceService.resource({ where: { id: resourceId } });

  // 1. 创建 ServiceTemplate Resource
  const template = await this.resourceService.createResource({
    data: {
      name: `${resource.name}-template`,
      resourceType: EnumResourceType.ServiceTemplate,
      codeGenerator: CODE_GENERATOR_NAME_TO_ENUM[resource.codeGeneratorName],
      blueprint: resource.blueprintId ? { connect: { id: resource.blueprintId } } : undefined,
    },
  }, user);

  // 2. 迁移 ServiceSettings（路径替换为模板变量 {{SERVICE_NAME}}）
  // 3. 复制插件安装
  await this.copyPluginInstallations(resourceId, template.id, user);

  return template;
}
```

### 3.2 模板版本发布（快照）

当模板发布新版本时，会对当前所有 Block 和 Entity 进行快照：

[packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L40-L112)

```typescript
async create(args: CreateResourceVersionArgs, userId: string): Promise<ResourceVersion> {
  const resourceId = args.data.resource.connect.id;
  const resource = await this.resourceService.resource({ where: { id: resourceId } });

  // 1. 校验版本号（仅 ServiceTemplate 需要 semver 格式）
  if (resource.resourceType === EnumResourceType.ServiceTemplate) {
    await this.validateVersion(args.data.version, resourceId);
  }

  // 2. 获取当前最新的所有 Entity 版本
  const latestEntityVersions = await this.entityService.getLatestVersions({
    where: { resourceId: resourceId },
  });

  // 3. 获取当前最新的所有 Block 版本
  const latestBlockVersions = await this.blockService.getLatestVersions({
    where: { resourceId: resourceId },
  });

  // 4. 创建 ResourceVersion，关联所有快照
  const resourceVersion = await this.prisma.resourceVersion.create({
    data: {
      ...args.data,
      blockVersions: { connect: latestBlockVersions.map(v => ({ id: v.id })) },
      entityVersions: { connect: latestEntityVersions.map(v => ({ id: v.id })) },
    },
  });

  // 5. 触发版本过期告警（通知所有使用该模板的服务）
  if (resource.resourceType === EnumResourceType.ServiceTemplate) {
    await this.outdatedVersionAlertService.triggerAlertsForTemplateVersion(
      resourceId, previousVersion?.version, args.data.version
    );
  }

  return resourceVersion;
}
```

### 3.3 从模板创建资源（导入流程）

从模板创建资源时会依次执行以下校验，任何一步失败都会导致整个迁移过程中断：

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L242-L371)

```typescript
async createResourceFromTemplate(args: CreateResourceFromTemplateArgs, user: User): Promise<Resource> {
  // 1. 获取可用模板列表（校验模板归属和可用性）
  const serviceTemplates = await this.availableServiceTemplatesForProject(...);

  if (!serviceTemplates || serviceTemplates.length === 0) {
    throw new AmplicationError(`Service template not found`);
  }

  const template = serviceTemplates.find(
    (template) => template.id === args.data.serviceTemplate.id
  );

  // 校验模板是否属于当前项目且用户可用
  if (template === undefined) {
    throw new AmplicationError(`Service template not found`);
  }

  // 2. 获取模板最新版本
  const templateVersion = await this.resourceVersionService.getLatest(
    template.id
  );

  if (!templateVersion) {
    throw new AmplicationError(`Template version not found`);
  }

  // 3. 校验 Blueprint 状态
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

  const resourceType = blueprint.resourceType as EnumResourceType;

  // 4. 校验 Blueprint 资源类型合法性（仅支持 Service 和 Component）
  if (
    ![EnumResourceType.Component, EnumResourceType.Service].includes(
      resourceType
    )
  ) {
    throw new AmplicationError(
      `The template is based on a blueprint with an unsupported resource type. Only components and services are supported`
    );
  }

  // 5. 根据 Blueprint 类型创建 Service 或 Component
  let newResource: Resource;
  if (resourceType === EnumResourceType.Component) {
    newResource = await this.internalCreateComponentFromTemplate(args, template, user);
  } else {
    newResource = await this.internalCreateServiceFromTemplate(args, template, user);
  }

  // 6. 记录资源使用的模板版本（关键：建立资源与模板快照的关联）
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    where: { id: newResource.id },
    data: { serviceTemplateId: template.id, version: templateVersion.version },
  }, user);

  // 7. 复制插件安装（每个插件单独校验）
  await this.copyPluginInstallations(args.data.serviceTemplate.id, newResource.id, user);

  // 8. 可选：创建后立即构建
  if (args.data.buildAfterCreation) {
    await this.projectService.commit(...);
  }

  return newResource;
}
```

### 3.4 插件复制流程

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L454-L489)

```typescript
async copyPluginInstallations(
  sourceResourceId: string,
  targetResourceId: string,
  user: User
) {
  const plugins =
    await this.pluginInstallationService.getOrderedPluginInstallations(
      sourceResourceId
    );

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

    // 逐个创建插件，任何一个插件创建失败都会抛出异常中断流程
    await this.pluginInstallationService.create(
      { data: { ...createInput } },
      user
    );
  }
}
```

插件创建时的校验：
[packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L97-L171)

```typescript
async validatePluginConfiguration(
  resourceId: string,
  configurations: JsonValue,
  user: User
): Promise<void> {
  // 插件需要认证实体但资源未配置认证实体 → 失败
  if (
    configurations &&
    configurations[REQUIRES_AUTHENTICATION_ENTITY] === "true"
  ) {
    const authEntity = await this.resourceService.getAuthEntityName(
      resourceId,
      user
    );
    if (isEmpty(authEntity)) {
      throw new AmplicationError(
        "The plugin requires an authentication entity. Please select the authentication entity in the service settings."
      );
    }
  }
}

async create(args: CreatePluginInstallationArgs, user: User): Promise<PluginInstallation> {
  const { configurations, resource } = args.data;

  // 1. 插件配置校验（如认证实体要求）
  await this.validatePluginConfiguration(resource.connect.id, configurations, user);

  // 2. 重复安装校验
  const existingPlugin = await this.findPluginInstallationByPluginId(
    args.data.pluginId,
    { resource: { id: resource.connect.id } }
  );

  if (existingPlugin.length > 0) {
    throw new AmplicationError(
      `The Plugin ${args.data.pluginId} already installed in resource ${resource.connect.id}`
    );
  }

  // 3. 创建 Block
  const newPlugin = await super.create(args, user);
  await this.setOrder(...);
  return newPlugin;
}
```

### 3.5 模板版本升级流程

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L498-L599)

```typescript
async upgradeServiceToLatestTemplateVersion(args: FindOneArgs, user: User): Promise<Resource> {
  const resourceId = args.where.id;

  // 1. 校验资源存在
  const resource = await this.resourceService.resource({ where: { id: resourceId } });
  if (!resource) {
    throw new AmplicationError(`Resource with id ${resourceId} not found `);
  }

  // 2. 校验资源是否基于模板
  const serviceTemplateVersion =
    await this.resourceService.getServiceTemplateSettings(resourceId, user);
  if (!serviceTemplateVersion) {
    throw new AmplicationError(
      `Service with id ${resourceId} is not based on a template `
    );
  }

  // 3. 校验模板存在
  const template = await this.resourceService.resource({
    where: { id: serviceTemplateVersion.serviceTemplateId },
  });
  if (!template) {
    throw new AmplicationError(
      `Template with id ${serviceTemplateVersion.serviceTemplateId} not found `
    );
  }

  // 4. 校验存在更新版本
  const latestVersion = await this.resourceVersionService.getLatest(template.id);
  if (latestVersion.version === serviceTemplateVersion.version) {
    throw new AmplicationError(
      `Service with id ${resourceId} is already up to date `
    );
  }

  // 5. 对比两个版本间的差异
  const changes = await this.resourceVersionService.compareResourceVersions({
    where: {
      resource: { id: template.id },
      sourceVersion: serviceTemplateVersion.version,
      targetVersion: latestVersion.version,
    },
  });

  const mergeOptions: BlockMergeOptions = { updatedManuallyCreatedBlocks: true };

  // 6. 合并变更：新增、删除、更新 Block
  await Promise.all([
    changes.createdBlocks.map(blockVersion =>
      this.handleMergeCreatedBlock(resourceId, blockVersion, user, mergeOptions)
    ),
    changes.deletedBlocks.map(blockVersion =>
      this.handleMergeDeletedBlock(resourceId, blockVersion, user)
    ),
    changes.updatedBlocks.forEach(diff =>
      this.handleMergeUpdatedBlock(resourceId, diff, user, mergeOptions)
    ),
  ]);

  // 7. 更新资源的模板版本记录
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    where: { id: resourceId },
    data: { version: latestVersion.version, serviceTemplateId: serviceTemplateVersion.serviceTemplateId },
  }, user);

  // 8. 标记版本过期告警为已解决
  await this.outdatedVersionAlertService.resolvesServiceTemplateUpdated({ resourceId });

  return resource;
}
```

### 3.6 Block 合并处理

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L601-L685)

模板升级时只处理两种 Block 类型：`PluginInstallation` 和 `CodeEngineVersion`。

```typescript
async handleMergeCreatedBlock(
  targetResourceId: string,
  blockVersion: BlockVersion,
  user: User,
  options: BlockMergeOptions
): Promise<IBlock | Resource> {
  // 处理插件
  if (blockVersion.block.blockType === EnumBlockType.PluginInstallation) {
    return this.pluginInstallationService.mergeVersionIntoLatest(
      blockVersion, targetResourceId, user, options
    );
  }

  // 处理代码引擎版本
  if (blockVersion.block.blockType === EnumBlockType.CodeEngineVersion) {
    const settings = blockVersion.settings as BlockSettingsProperties<TemplateCodeEngineVersion>;
    return this.resourceService.updateCodeGeneratorVersion({
      data: {
        codeGeneratorVersionOptions: {
          codeGeneratorVersion: settings.codeGeneratorVersion,
          codeGeneratorStrategy: settings.codeGeneratorStrategy,
        },
      },
      where: { id: targetResourceId },
    }, user);
  }
}
```

插件合并逻辑：
[packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L309-L371)

```typescript
async mergeVersionIntoLatest(
  blockVersion: BlockVersion,
  targetResourceId: string,
  user: User,
  options: BlockMergeOptions
) {
  const settings = blockVersion.settings as BlockSettingsProperties<PluginInstallation>;

  const existingPluginInstallations =
    await this.findPluginInstallationByPluginId(settings.pluginId, {
      resource: { id: targetResourceId },
    });

  if (existingPluginInstallations.length > 0) {
    const existingPluginInstallation = existingPluginInstallations[0];
    if (options.updatedManuallyCreatedBlocks) {
      // 已存在且允许更新 → 更新（会触发 validatePluginConfiguration 校验）
      return this.update({
        data: { ...settings, displayName: blockVersion.displayName },
        where: { id: existingPluginInstallation.id },
      }, user);
    } else {
      // 已存在且不允许更新 → 抛出异常
      throw new AmplicationError(
        `The Plugin ${settings.pluginId} already installed in resource ${targetResourceId}`
      );
    }
  } else {
    // 不存在 → 创建新插件（会触发 validatePluginConfiguration 校验）
    return this.create({
      data: {
        ...settings,
        isPrivate: settings.isPrivate,
        displayName: blockVersion.displayName,
        resource: { connect: { id: targetResourceId } },
      },
    }, user);
  }
}
```

### 3.7 版本差异对比算法

[packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L251-L337)

```typescript
async compareResourceVersions(args: CompareResourceVersionsArgs): Promise<ResourceVersionsDiff> {
  // 1. 获取源版本和目标版本关联的所有 BlockVersion
  const sourceBlockVersions = sourceResourceVersion
    ? await this.blockService.getBlockVersionsByResourceVersions(sourceResourceVersion.id)
    : [];
  const targetBlockVersions = await this.blockService.getBlockVersionsByResourceVersions(targetResourceVersion.id);

  // 2. 以 block.id 为键进行对比
  const updated: ResourceVersionsDiffBlock[] = [];
  const deleted: BlockVersion[] = [];
  const created: BlockVersion[] = [];

  for (const sourceBlockVersion of sourceBlockVersions) {
    const targetBlockVersion = targetBlockVersions.find(
      bv => bv.block.id === sourceBlockVersion.block.id
    );
    if (targetBlockVersion) {
      // 版本号不同 → 更新
      if (sourceBlockVersion.versionNumber !== targetBlockVersion.versionNumber) {
        updated.push({ sourceBlockVersion, targetBlockVersion });
      }
    } else {
      // 目标中不存在 → 删除
      deleted.push(sourceBlockVersion);
    }
  }

  for (const targetBlockVersion of targetBlockVersions) {
    const sourceBlockVersion = sourceBlockVersions.find(
      bv => bv.block.id === targetBlockVersion.block.id
    );
    if (!sourceBlockVersion) {
      // 源中不存在 → 新增
      created.push(targetBlockVersion);
    }
  }

  return { updatedBlocks: updated, createdBlocks: created, deletedBlocks: deleted };
}
```

### 3.8 资源模板版本记录

每个资源使用的模板版本以 `ResourceTemplateVersion` Block 形式存储：

[packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts#L19-L88)

```typescript
// DTO: ResourceTemplateVersion { serviceTemplateId, version }

async updateResourceTemplateVersion(args, user) {
  const existing = await this.getResourceTemplateVersionBlock({ where: { id: args.where.id } });
  if (!existing) {
    // 不存在则创建新 Block
    return this.blockService.create<ResourceTemplateVersion>({
      data: { ...DEFAULT_RESOURCE_TEMPLATE_VERSION, resource: { connect: { id: args.where.id } }, ...args.data },
    }, user.id);
  }
  // 存在则更新
  return this.blockService.update<ResourceTemplateVersion>({
    where: { id: existing.id }, data: { ...args.data },
  }, user);
}
```

默认值定义：
[packages/amplication-server/src/core/resourceTemplateVersion/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/constants.ts#L13-L20)

```typescript
export const DEFAULT_RESOURCE_TEMPLATE_VERSION = {
  blockType: EnumBlockType.ResourceTemplateVersion,
  description: "Resource Template Version",
  displayName: "Resource Template Version",
  serviceTemplateId: null,
  version: null,
};
```

### 3.9 模板代码引擎版本记录

[packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts#L14-L85)

```typescript
// DTO: TemplateCodeEngineVersion { codeGeneratorVersion?, codeGeneratorStrategy? }

async update(serviceTemplateId, codeGeneratorVersion, codeGeneratorStrategy, user) {
  const currentValue = await this.getCurrent(serviceTemplateId);
  if (!currentValue) {
    return this.blockService.create<TemplateCodeEngineVersion>({
      data: { resource: { connect: { id: serviceTemplateId } }, ...DEFAULT_VALUE },
    }, user.id);
  } else {
    return this.blockService.update<TemplateCodeEngineVersion>({
      where: { id: currentValue.id },
      data: { codeGeneratorVersion, codeGeneratorStrategy },
    }, user);
  }
}
```

### 3.10 资源模板版本与代码引擎版本的关联

两者的关联通过 **ServiceTemplate Resource** 作为中间节点建立，关联路径如下：

```
Resource (Service 类型)
    └── Block (ResourceTemplateVersion)
          ├── serviceTemplateId ──────────────────────────┐
          └── version (模板版本号)                         │
                                                           ▼
                                              Resource (ServiceTemplate 类型)
                                                          ├── ResourceVersion (版本快照)
                                                          │     └── BlockVersion[] (包含 CodeEngineVersion 的快照)
                                                          └── Block (TemplateCodeEngineVersion)
                                                                ├── codeGeneratorVersion
                                                                └── codeGeneratorStrategy
```

关键关联代码：

**1) 模板更新代码引擎时同步写入 TemplateCodeEngineVersion Block**

[packages/amplication-server/src/core/resource/resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L361-L413)

```typescript
async updateCodeGeneratorVersion(args: UpdateCodeGeneratorVersionArgs, user: User): Promise<Resource | null> {
  const resource = await this.resource({ where: { id: args.where.id } });
  if (isEmpty(resource)) {
    throw new Error(INVALID_RESOURCE_ID);
  }

  // 1. 计费校验：代码引擎版本更新功能是否可用
  const codeGeneratorUpdate = await this.billingService.getBooleanEntitlement(
    user.workspace.id, BillingFeature.CodeGeneratorVersion
  );
  if (codeGeneratorUpdate && !codeGeneratorUpdate.hasAccess)
    throw new AmplicationError("Feature Unavailable. Please upgrade your plan...");

  // 2. 更新 Resource 表的 codeGeneratorVersion 和 codeGeneratorStrategy 字段
  const updatedResource = await this.prisma.resource.update({
    where: args.where,
    data: {
      codeGeneratorVersion: args.data.codeGeneratorVersionOptions.codeGeneratorVersion,
      codeGeneratorStrategy: args.data.codeGeneratorVersionOptions.codeGeneratorStrategy,
    },
  });

  // 3. 仅 ServiceTemplate 类型资源：额外写入 TemplateCodeEngineVersion Block 作为历史记录
  if (resource.resourceType === EnumResourceType.ServiceTemplate) {
    await this.templateCodeEngineVersionService.update(
      resource.id,
      args.data.codeGeneratorVersionOptions.codeGeneratorVersion,
      args.data.codeGeneratorVersionOptions.codeGeneratorStrategy,
      user
    );
  }
  return updatedResource;
}
```

**2) 从模板创建资源时，代码引擎版本的传递**

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L373-L426)

```typescript
private async internalCreateServiceFromTemplate(args, template, user) {
  const serviceSettings = await this.serviceSettingsService.getServiceSettingsValues(...);

  // ... 路径替换逻辑 ...

  const newService = await this.resourceService.createService({
    data: {
      blueprint: { connect: { id: template.blueprintId } },
      name: args.data.name,
      // 模板的 codeGeneratorName 传递给新资源
      codeGenerator: template.codeGeneratorName
        ? CODE_GENERATOR_NAME_TO_ENUM[template.codeGeneratorName]
        : EnumCodeGenerator.NodeJs,
      // ...
    },
  }, user);

  return newService;
}
```

**注意**：从模板创建资源时，`codeGeneratorVersion` 和 `codeGeneratorStrategy` 并没有直接从模板传递给新资源，而是依赖 ResourceService.createService() 中的默认逻辑。只有在后续执行模板版本升级时，才会通过 `handleMergeCreatedBlock` / `handleMergeUpdatedBlock` 将 `TemplateCodeEngineVersion` Block 中的快照值同步到目标资源。

---

## 四、模板快照导入校验失败对迁移的影响

### 4.1 校验失败场景汇总

从模板导入资源的过程中存在多个校验节点，每个节点的失败都会对迁移产生不同程度的影响：

| 校验阶段 | 校验内容 | 失败影响 | 抛出异常位置 |
|----------|----------|----------|--------------|
| 模板可用性校验 | 模板在当前项目中是否可用 | **硬失败**：迁移完全中断，资源尚未创建 | [serviceTemplate.service.ts#L255-L266](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L255-L266) |
| 模板版本存在校验 | 模板至少有一个已发布版本 | **硬失败**：迁移完全中断，资源尚未创建 | [serviceTemplate.service.ts#L272-L274](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L272-L274) |
| Blueprint 存在校验 | 模板关联的 Blueprint 存在且已启用 | **硬失败**：迁移完全中断，资源尚未创建 | [serviceTemplate.service.ts#L282-L290](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L282-L290) |
| Blueprint 资源类型校验 | Blueprint 类型必须是 Service 或 Component | **硬失败**：迁移完全中断，资源尚未创建 | [serviceTemplate.service.ts#L294-L302](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L294-L302) |
| 计费配额校验 | 工作区服务数量未超过套餐限制 | **硬失败**：资源创建前中断 | [resource.service.ts#L223-L245](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L223-L245) |
| 项目配置存在校验 | 项目必须存在 ProjectConfiguration | **硬失败**：资源创建前中断 | [resource.service.ts#L249-L253](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L249-L253) |
| 代码生成器 License 校验 | 用户套餐是否支持所选代码生成器 | **硬失败**：资源创建前中断 | [resource.service.ts#L415-L450](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L415-L450) |
| 资源名称重复校验 | 同项目下资源名称不重复（自动追加序号） | 不会失败，自动重命名 | [resource.service.ts#L255-L279](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L255-L279) |
| 插件重复安装校验 | 目标资源已安装同名插件 | **部分失败**：资源已创建但插件安装中断 | [pluginInstallation.service.ts#L142-L146](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L142-L146) |
| 插件配置校验 | 插件要求认证实体但资源未配置 | **部分失败**：资源已创建但插件安装中断 | [pluginInstallation.service.ts#L97-L119](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L97-L119) |
| Block 父节点类型校验 | Block 的父节点类型合法性 | **硬失败**：Block 创建失败导致整个操作失败 | [block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.service.ts#L83-L103) |

### 4.2 硬失败 vs 部分失败

#### 硬失败（资源未创建）

发生在资源实体写入数据库之前，失败后数据库保持干净状态，无需回滚。包括：
- 模板不可用 / 模板不存在 / 模板无版本
- Blueprint 缺失 / Blueprint 被禁用 / Blueprint 类型不支持
- 计费配额超限
- 代码生成器 License 不可用

#### 部分失败（资源已创建但插件未安装完整）

这是当前实现的一个**设计缺陷**：`createResourceFromTemplate()` 没有使用数据库事务包裹整个流程。

**失败时序**：
```
1. newService = await internalCreateServiceFromTemplate()   ✅ Resource 已写入 DB
2. updateResourceTemplateVersion()                           ✅ ResourceTemplateVersion Block 已写入
3. copyPluginInstallations()
       ├── plugin1.create()    ✅
       ├── plugin2.create()    ❌ 抛出 AmplicationError（如认证实体缺失）
       └── plugin3...          未执行
```

**影响**：
- 资源本身已创建成功
- 资源的模板版本记录已写入（指向正确的模板版本）
- 插件安装停留在失败点，之前的插件已成功安装，之后的未安装
- 资源处于**不一致状态**：模板版本标记正确，但实际插件集合不完整

### 4.3 模板升级时的校验失败影响

模板版本升级同样存在部分失败问题：

[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L557-L579)

```typescript
await Promise.all([
  changes.createdBlocks.map(...),   // 数组中的 Promise 任一失败都会导致 Promise.all 失败
  changes.deletedBlocks.map(...),
  changes.updatedBlocks.forEach(...),  // 注意：这里是 forEach 不是 map，返回 undefined 不影响
]);

// 步骤 7：更新模板版本记录
await this.resourceTemplateVersionService.updateResourceTemplateVersion(...)
```

**失败时序**：
```
1. compareResourceVersions()     ✅ 差异计算完成
2. Promise.all([createdPromises, deletedPromises, updatedPromises])
       ├── PluginInstallation A create()  ✅
       ├── PluginInstallation B create()  ❌ 抛出异常（如缺少认证实体）
       ├── CodeEngineVersion update       取决于执行顺序
       └── ...
3. updateResourceTemplateVersion()  ❌ 未执行
4. resolvesServiceTemplateUpdated() ❌ 未执行
```

**影响**：
- 部分 Block 已成功创建/更新，但模板版本号未更新
- 资源版本记录仍指向旧版本
- 版本过期告警未被标记为已解决
- 资源处于**半升级状态**：部分插件已更新，但系统认为资源仍使用旧模板版本

---

## 五、兼容校验（Compatibility Validation）

### 5.1 版本号校验（Semver）

[packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L171-L192)

```typescript
async validateVersion(version: string, resourceId): Promise<void> {
  if (!version) throw new Error("Version is required");

  // 使用 semver 库校验格式合法性
  if (!valid(version)) {
    throw new Error(`Version ${version} is not a valid semver version`);
  }

  // 校验版本唯一性
  const existingVersion = await this.prisma.resourceVersion.findFirst({
    where: { resourceId: resourceId, version: version },
  });
  if (existingVersion) {
    throw new Error(`Version ${version} already exists for resource ${resourceId}`);
  }
}
```

### 5.2 Blueprint 引擎类型校验

[packages/amplication-server/src/core/blueprint/blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L29-L38)

```typescript
const VALID_TYPES_AND_GENERATORS: Partial<Record<EnumResourceType, (keyof typeof EnumCodeGenerator)[]>> = {
  [EnumResourceType.Component]: [EnumCodeGenerator.Blueprint],
  [EnumResourceType.Service]: [EnumCodeGenerator.NodeJs, EnumCodeGenerator.DotNet],
  [EnumResourceType.MessageBroker]: [EnumCodeGenerator.Blueprint],
};
```

[packages/amplication-server/src/core/blueprint/blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L164-L219)

```typescript
async updateBlueprintEngine(args: UpdateBlueprintEngineArgs): Promise<Blueprint> {
  // 1. 已被资源使用的 Blueprint 不允许修改引擎
  const resources = await this.prisma.resource.findMany({
    where: { blueprintId: args.where.id, deletedAt: null },
  });
  if (resources.length > 0) {
    throw new AmplicationError("Cannot update engine of blueprint because it is already in use by resources");
  }

  // 2. 校验 resourceType 与 codeGenerator 的组合是否合法
  const allowedEngine = VALID_TYPES_AND_GENERATORS[args.data.resourceType];
  if (!allowedEngine?.includes(args.data.codeGenerator)) {
    throw new AmplicationError(`Invalid code generator for resource type: ${args.data.resourceType}`);
  }
  // ...
}
```

### 5.3 自定义属性校验（JSON Schema）

[packages/amplication-server/src/core/customProperty/customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts#L296-L386)

```typescript
async validateCustomProperties(
  customProperties: CustomProperty[],
  data: Record<string, unknown>
): Promise<SchemaValidationResult> {
  const schema = this.getValidationSchema(customProperties);
  return this.jsonSchemaValidationService.validateSchema(schema, data);
}

getValidationSchema(customProperties: CustomProperty[]): JSONSchema {
  const properties: Record<string, JSONSchema> = {};
  const required: string[] = [];

  for (const customProperty of customProperties) {
    const key = customProperty.key;
    const schema: JSONSchema = { title: customProperty.name, type: "string" };

    // 根据属性类型生成对应的 JSON Schema 规则
    if (customProperty.type === EnumCustomPropertyType.Select) {
      schema.enum = customProperty.options.map(o => o.value);
    }
    if (customProperty.type === EnumCustomPropertyType.MultiSelect) {
      schema.type = "array";
      schema.items = { type: "string", enum: customProperty.options.map(o => o.value) };
    }
    if (customProperty.required) {
      required.push(key);
      schema.isNotEmpty = true;
    }
    if (customProperty.validationRule) {
      schema.pattern = customProperty.validationRule;
    }
    properties[key] = schema;
  }

  return {
    additionalProperties: false,  // 不允许未定义属性
    type: "object",
    required,
    properties,
  };
}
```

### 5.4 资源设置属性校验

[packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts#L63-L102)

```typescript
async validateResourceSettingsProperties(resourceId: string, properties: Record<string, unknown>): Promise<void> {
  const resource = await this.resourceService.resource({ where: { id: resourceId } });

  // 获取该资源关联 Blueprint 定义的所有自定义属性
  const blueprintProperties = await this.blueprintService.properties({
    where: { id: resource.blueprintId },
  });

  // 基于 Blueprint 的属性定义进行 JSON Schema 校验
  const validationResults = await this.customPropertyService.validateCustomProperties(
    blueprintProperties, properties
  );
  if (!validationResults.isValid) {
    throw new AmplicationError(`Validation failed for resource settings properties: ${validationResults.errorText}`);
  }
}
```

### 5.5 资源通用属性校验

[packages/amplication-server/src/core/resource/resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1524-L1552)

```typescript
async validateResourceProperties(values: Record<string, unknown>, user: User): Promise<void> {
  // 获取工作区中所有启用的自定义属性
  const customProperties = await this.customPropertyService.customProperties({
    where: { workspace: { id: user.workspace.id }, enabled: true },
  });

  const validationResults = await this.customPropertyService.validateCustomProperties(
    customProperties, values
  );
  if (!validationResults.isValid) {
    throw new AmplicationError(`Validation failed for resource properties: ${validationResults.errorText}`);
  }
}
```

### 5.6 Block 父节点类型校验

[packages/amplication-server/src/core/block/block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.service.ts#L83-L103)

```typescript
blockTypeAllowedParents: { [key in EnumBlockType]: Set<EnumBlockType | null> } = {
  [EnumBlockType.ServiceSettings]: ALLOW_NO_PARENT_ONLY,
  [EnumBlockType.ProjectConfigurationSettings]: ALLOW_NO_PARENT_ONLY,
  [EnumBlockType.ModuleAction]: new Set([EnumBlockType.Module]),
  [EnumBlockType.ModuleDto]: new Set([EnumBlockType.Module]),
  // ... 其他类型
};

// create() 中进行校验
if (!this.canUseParentType(EnumBlockType[blockType], parentBlock && EnumBlockType[parentBlock.blockType])) {
  throw new ConflictException(`Block type ${parentBlock?.blockType} is not allowed as a parent for block type ${blockType}`);
}
```

### 5.7 代码生成器可用性校验

[packages/amplication-server/src/core/resource/resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L415-L450)

```typescript
async getAndValidateCodeGeneratorName(codeGenerator, user): Promise<string | null> {
  // 1. 计费限制：Node.js Only 套餐限制
  const blockEntitlement = await this.billingService.getBooleanEntitlement(
    user.workspace.id, BillingFeature.CodeGeneratorNodeJsOnly
  );
  if (blockEntitlement?.hasAccess && codeGenerator !== EnumCodeGenerator.NodeJs) {
    throw new AmplicationError("Feature Unavailable. Please upgrade your plan...");
  }

  // 2. 特定代码生成器的 License 校验
  const { codeGeneratorName, license } = CODE_GENERATOR_ENUM_TO_NAME_AND_LICENSE[codeGenerator];
  if (license) {
    const entitlement = await this.billingService.getBooleanEntitlement(user.workspace.id, license);
    if (entitlement && !entitlement.hasAccess) {
      throw new AmplicationError("Feature Unavailable. Please upgrade your plan...");
    }
  }
  return codeGeneratorName;
}
```

常量映射表：
```typescript
const CODE_GENERATOR_ENUM_TO_NAME_AND_LICENSE = {
  [EnumCodeGenerator.DotNet]: { codeGeneratorName: "CodeGeneratorDotNet", license: BillingFeature.CodeGeneratorDotNet },
  [EnumCodeGenerator.NodeJs]: { codeGeneratorName: null, license: null },
  [EnumCodeGenerator.Blueprint]: { codeGeneratorName: null, license: null },
};
```

---

## 六、版本过期告警（Outdated Version Alert）

### 6.1 告警触发

当模板发布新版本时触发告警：

[packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L179-L246)

```typescript
async triggerAlertsForTemplateVersion(templateResourceId, outdatedVersion, latestVersion) {
  // 1. 查找所有使用该模板的服务（通过 ResourceTemplateVersion Block 查询）
  const serviceIds = await this.resourceTemplateVersionService.getServiceIdsByTemplateId(
    templateResourceId
  );

  // 2. 为每个服务创建 TemplateVersion 类型的告警
  for (const serviceId of serviceIds) {
    const currentTemplateVersion = await this.resourceService.getServiceTemplateSettings(serviceId, null);
    await this.create({
      data: {
        resource: { connect: { id: serviceId } },
        type: EnumOutdatedVersionAlertType.TemplateVersion,
        outdatedVersion: currentTemplateVersion.version,
        latestVersion,
      },
    }, template.name);
  }

  // 3. 通过 Kafka 异步发送通知
  await this.raiseNotifications(template.name, latestVersion, serviceIds);
}
```

### 6.2 Kafka 通知机制

告警创建后通过 Kafka `TECH_DEBT_CREATED_TOPIC` 向工作区用户发送通知。

### 6.3 告警类型

参考 [packages/amplication-server/src/core/outdatedVersionAlert/dto/EnumOutdatedVersionAlertType.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto/EnumOutdatedVersionAlertType.ts)

- `TemplateVersion` - 模板版本过期
- `PluginVersion` - 插件版本过期

### 6.4 告警解决

模板升级成功后自动标记为已解决：

```typescript
await this.outdatedVersionAlertService.resolvesServiceTemplateUpdated({ resourceId });
```

---

## 七、前端交互链路

### 7.1 GraphQL 查询定义

[packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts#L1-L113)

| 查询/Mutation | 用途 |
|--------------|------|
| `GET_SERVICE_TEMPLATES` | 获取项目中的模板列表 |
| `CREATE_SERVICE_TEMPLATE` | 创建新模板 |
| `CREATE_TEMPLATE_FROM_RESOURCE` | 从现有资源创建模板 |
| `GET_AVAILABLE_TEMPLATES_FOR_PROJECT` | 获取项目可用模板（含公开项目） |
| `UPGRADE_SERVICE_TO_LATEST_TEMPLATE_VERSION` | 将服务升级到最新模板版本 |

### 7.2 React Hook 封装

[packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts#L37-L190)

```typescript
const useServiceTemplate = (currentProject, onServiceTemplateCreated) => {
  // 查询模板列表
  const { loadingServiceTemplates, serviceTemplates, reloadServiceTemplates } =
    useQuery(GET_SERVICE_TEMPLATES, { variables: { projectId: currentProject?.id } });

  // 创建模板
  const [createServiceTemplateInternal] = useMutation(CREATE_SERVICE_TEMPLATE);
  const createServiceTemplate = (data) => {
    createServiceTemplateInternal({ variables: { data } })
      .then(result => { reloadServiceTemplates(); ... });
  };

  // 从资源创建模板
  const createTemplateFromResource = (resourceId) => { ... };

  // 升级模板版本
  const upgradeServiceToLatestTemplateVersion = (resourceId) => { ... };

  // 通过模板查找关联资源
  const findResourcesByTemplate = (templateId) => { ... };

  return { serviceTemplates, createServiceTemplate, upgradeServiceToLatestTemplateVersion, ... };
};
```

### 7.3 代码引擎版本设置前端

[packages/amplication-client/src/Resource/codeGeneratorVersionSettings/CodeGeneratorVersion.tsx](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/Resource/codeGeneratorVersionSettings/CodeGeneratorVersion.tsx)

前端通过 `updateCodeGeneratorVersion` mutation 更新资源的代码引擎版本，对于 ServiceTemplate 类型资源，后端会同步写入 `TemplateCodeEngineVersion` Block。

---

## 八、完整调用链路总结图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              前端 (amplication-client)                        │
│  useServiceTemplate Hook → GraphQL Mutations/Queries                         │
│  CodeGeneratorVersion 设置 → updateCodeGeneratorVersion Mutation             │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         GraphQL Resolver 层                                   │
│  ServiceTemplateResolver / ResourceResolver / ProjectResolver                │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Service 业务层                                       │
│                                                                              │
│  ┌─────────────────────┐    ┌─────────────────────┐    ┌──────────────────┐  │
│  │ ProjectService      │    │ ResourceService     │    │ BlueprintService │  │
│  │  - createProject    │───▶│  - createProjectCfg │    │  - 引擎类型校验  │  │
│  └─────────────────────┘    │  - createFromTpl    │    └──────────────────┘  │
│                             │  - upgradeTpl      │                            │
│                             │  - updateCodeGenVer│                            │
│                             │  - 计费/属性校验    │                            │
│                             └──────────┬──────────┘                            │
│                                        │                                       │
│            ┌───────────────────────────┼───────────────────────────┐          │
│            ▼                           ▼                           ▼          │
│  ┌────────────────────┐   ┌────────────────────────┐  ┌────────────────────┐ │
│  │ ServiceTemplate    │   │ ResourceVersionService │  │ OutdatedVersion    │ │
│  │ Service            │   │  - 创建快照            │  │ AlertService       │ │
│  │  - 创建/升级模板    │   │  - 版本对比            │  │  - 触发告警        │ │
│  │  - 从模板创建资源  │   │  - semver校验          │  │  - 解决告警(Kafka) │ │
│  └─────────┬──────────┘   └────────────┬───────────┘  └────────────────────┘ │
│            │                           │                                    │
│            ▼                           ▼                                    │
│  ┌────────────────────────┐  ┌─────────────────────────────┐                │
│  │ ResourceTemplate       │  │ TemplateCodeEngineVersion   │                │
│  │ VersionService         │  │ Service                     │                │
│  │ (记录资源使用的模板版) │  │ (记录模板的代码引擎历史)    │                │
│  └───────────┬────────────┘  └──────────────┬──────────────┘                │
│              │                               │                                │
│              └───────────────┬───────────────┘                                │
│                              ▼                                                │
│                  ┌─────────────────────┐    ┌────────────────────────┐      │
│                  │ BlockService        │    │ CustomPropertyService  │      │
│                  │  - CRUD all Blocks  │    │  - JSON Schema 校验    │      │
│                  │  - 父子类型校验     │    │  - 属性规则构建        │      │
│                  └──────────┬──────────┘    └────────────────────────┘      │
│                             │                                                 │
│                             ▼                                                 │
│                  ┌─────────────────────┐                                     │
│                  │ PluginInstallation  │                                     │
│                  │ Service             │                                     │
│                  │  - 插件配置校验      │                                     │
│                  │  - mergeVersion...  │                                     │
│                  └──────────┬──────────┘                                     │
│                             │                                                 │
│                             ▼                                                 │
│                  ┌─────────────────────┐                                     │
│                  │ Prisma (DB Layer)   │                                     │
│                  │ Block / BlockVersion│                                     │
│                  │ Resource / ResourceVersion│                                │
│                  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键数据实体关系

```
Project (1) ──┬──> Resource (ProjectConfiguration) ──> Block (ProjectConfigurationSettings)
              │
              ├──> Resource (ServiceTemplate) ──┬──> Block (ServiceSettings)
              │                                   ├──> Block (PluginInstallation)[]
              │                                   ├──> Block (PluginOrder)
              │                                   ├──> Block (CodeEngineVersion / TemplateCodeEngineVersion)
              │                                   │     ├── codeGeneratorVersion
              │                                   │     └── codeGeneratorStrategy
              │                                   │
              │                                   └──> ResourceVersion (snapshot)
              │                                            ├── version (semver)
              │                                            ├──> EntityVersion[]
              │                                            └──> BlockVersion[]
              │
              └──> Resource (Service/Component) ──┬──> Block (ResourceTemplateVersion)
              │                                     │     ├── serviceTemplateId ────────┐
              │                                     │     └── version (模板版本号)        │
              │                                     │                                   │
              │                                     ├──> Block (ResourceSettings)        │
              │                                     ├──> Block (ServiceSettings)         │
              │                                     ├──> Block (PluginInstallation)[]    │
              │                                     └──> OutdatedVersionAlert[]          │
              │                                                                           │
              └───────────────────────────────────────────────────────────────────────────┘
                                                                 (serviceTemplateId 关联)
```

---

## 十、关键文件索引

| 模块 | 服务层 | Resolver | DTO/常量 |
|------|--------|----------|----------|
| 项目配置 | [packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts) | [packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.resolver.ts) | [packages/amplication-server/src/core/projectConfigurationSettings/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/dto) |
| 项目 | [packages/amplication-server/src/core/project/project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.service.ts) | [packages/amplication-server/src/core/project/project.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.resolver.ts) | [packages/amplication-server/src/core/project/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/dto) |
| 资源/模板 | [packages/amplication-server/src/core/resource/resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | [packages/amplication-server/src/core/resource/resource.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts) | [packages/amplication-server/src/core/resource/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/dto) |
| 服务模板 | [packages/amplication-server/src/core/resource/serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | [packages/amplication-server/src/core/resource/serviceTemplate.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.resolver.ts) | - |
| 资源版本 | [packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts) | [packages/amplication-server/src/core/resourceVersion/resourceVersion.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.resolver.ts) | [packages/amplication-server/src/core/resourceVersion/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/dto) |
| 资源模板版本 | [packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) | - | [packages/amplication-server/src/core/resourceTemplateVersion/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/dto) |
| 模板代码引擎版本 | [packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts) | - | [packages/amplication-server/src/core/templateCodeEngineVersion/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/dto) |
| Blueprint | [packages/amplication-server/src/core/blueprint/blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts) | [packages/amplication-server/src/core/blueprint/blueprint.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.resolver.ts) | [packages/amplication-server/src/core/blueprint/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/dto) |
| Block | [packages/amplication-server/src/core/block/block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.service.ts) | [packages/amplication-server/src/core/block/block.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.resolver.ts) | [packages/amplication-server/src/core/block/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/dto) |
| 自定义属性 | [packages/amplication-server/src/core/customProperty/customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts) | [packages/amplication-server/src/core/customProperty/customProperty.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.resolver.ts) | [packages/amplication-server/src/core/customProperty/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/dto) |
| 资源设置 | [packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts) | [packages/amplication-server/src/core/resourceSettings/resourceSettings.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.resolver.ts) | [packages/amplication-server/src/core/resourceSettings/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/dto) |
| 插件安装 | [packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | [packages/amplication-server/src/core/pluginInstallation/pluginInstallation.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.resolver.ts) | [packages/amplication-server/src/core/pluginInstallation/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/pluginInstallation/dto) |
| 版本过期告警 | [packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts) | - | [packages/amplication-server/src/core/outdatedVersionAlert/dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto) |
| BlockType 基类 | [packages/amplication-server/src/core/block/blockType.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/blockType.service.ts) | - | - |
| Block 类型枚举 | - | - | [packages/amplication-server/src/enums/EnumBlockType.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/enums/EnumBlockType.ts) |
| 前端 Hook | [packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts) | - | [packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts) |
| 前端代码引擎设置 | [packages/amplication-client/src/Resource/codeGeneratorVersionSettings/CodeGeneratorVersion.tsx](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/Resource/codeGeneratorVersionSettings/CodeGeneratorVersion.tsx) | - | [packages/amplication-client/src/Resource/codeGeneratorVersionSettings/queries.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/Resource/codeGeneratorVersionSettings/queries.ts) |