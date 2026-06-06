# Import/Export 配置迁移代码链路梳理

本文档从代码实现角度梳理 Amplication 项目配置、模板快照和兼容校验的完整链路。

---

## 一、核心架构概览

整个 Import/Export 配置迁移系统围绕 **Block** 抽象模型构建。所有配置项（项目配置、服务设置、模板版本、代码引擎版本等）都以 Block 的形式存储，通过 BlockService 统一管理。

### Block 类型枚举
参考 [EnumBlockType.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/enums/EnumBlockType.ts#L3-L19)

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
[ProjectConfigurationSettings.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/dto/ProjectConfigurationSettings.ts#L1-L14)

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
[project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.service.ts#L74-L118)

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
[resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L180-L206)

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
[projectConfigurationSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts#L17-L94)

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
| ServiceTemplateService | 模板的 CRUD、从模板创建资源、模板升级 | [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) |
| ResourceVersionService | 资源版本管理、版本差异对比 | [resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts) |
| ResourceTemplateVersionService | 记录资源使用的模板版本 | [resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) |
| TemplateCodeEngineVersionService | 记录模板使用的代码引擎版本历史 | [templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts) |

### 3.1 模板创建流程

#### 方式一：直接创建服务模板
[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L57-L89)

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
[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L95-L179)

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

[resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L40-L112)

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

### 3.3 从模板创建资源

[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L242-L371)

```typescript
async createResourceFromTemplate(args: CreateResourceFromTemplateArgs, user: User): Promise<Resource> {
  // 1. 获取可用模板列表（校验模板归属和可用性）
  const serviceTemplates = await this.availableServiceTemplatesForProject(...);

  // 2. 获取模板最新版本
  const templateVersion = await this.resourceVersionService.getLatest(template.id);

  // 3. 校验 Blueprint 状态
  const blueprint = await this.prisma.blueprint.findUnique({ where: { id: template.blueprintId } });
  if (!blueprint.enabled) {
    throw new AmplicationError("The selected template is based on a disabled blueprint.");
  }

  // 4. 根据 Blueprint 类型创建 Service 或 Component
  let newResource: Resource;
  if (resourceType === EnumResourceType.Component) {
    newResource = await this.internalCreateComponentFromTemplate(args, template, user);
  } else {
    newResource = await this.internalCreateServiceFromTemplate(args, template, user);
  }

  // 5. 记录资源使用的模板版本（关键：建立资源与模板快照的关联）
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    where: { id: newResource.id },
    data: { serviceTemplateId: template.id, version: templateVersion.version },
  }, user);

  // 6. 复制插件安装
  await this.copyPluginInstallations(args.data.serviceTemplate.id, newResource.id, user);

  return newResource;
}
```

### 3.4 模板版本升级

[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L498-L599)

```typescript
async upgradeServiceToLatestTemplateVersion(args: FindOneArgs, user: User): Promise<Resource> {
  // 1. 获取资源当前使用的模板版本
  const serviceTemplateVersion = await this.resourceService.getServiceTemplateSettings(resourceId, user);

  // 2. 获取模板最新版本
  const latestVersion = await this.resourceVersionService.getLatest(template.id);

  // 3. 对比两个版本间的差异
  const changes = await this.resourceVersionService.compareResourceVersions({
    where: {
      resource: { id: template.id },
      sourceVersion: serviceTemplateVersion.version,
      targetVersion: latestVersion.version,
    },
  });

  // 4. 合并变更：新增、删除、更新 Block
  const mergeOptions: BlockMergeOptions = { updatedManuallyCreatedBlocks: true };
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

  // 5. 更新资源的模板版本记录
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    where: { id: resourceId },
    data: { version: latestVersion.version, serviceTemplateId: serviceTemplateVersion.serviceTemplateId },
  }, user);

  // 6. 标记版本过期告警为已解决
  await this.outdatedVersionAlertService.resolvesServiceTemplateUpdated({ resourceId });
}
```

### 3.5 版本差异对比算法

[resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L251-L337)

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

### 3.6 资源模板版本记录

每个资源使用的模板版本以 `ResourceTemplateVersion` Block 形式存储：

[resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts#L19-L88)

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
[constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/constants.ts#L13-L20)

```typescript
export const DEFAULT_RESOURCE_TEMPLATE_VERSION = {
  blockType: EnumBlockType.ResourceTemplateVersion,
  description: "Resource Template Version",
  displayName: "Resource Template Version",
  serviceTemplateId: null,
  version: null,
};
```

### 3.7 模板代码引擎版本记录

[templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts#L14-L85)

```typescript
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

---

## 四、兼容校验（Compatibility Validation）

### 4.1 版本号校验（Semver）

[resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L171-L192)

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

### 4.2 Blueprint 引擎类型校验

[blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L29-L38)

```typescript
const VALID_TYPES_AND_GENERATORS: Partial<Record<EnumResourceType, (keyof typeof EnumCodeGenerator)[]>> = {
  [EnumResourceType.Component]: [EnumCodeGenerator.Blueprint],
  [EnumResourceType.Service]: [EnumCodeGenerator.NodeJs, EnumCodeGenerator.DotNet],
  [EnumResourceType.MessageBroker]: [EnumCodeGenerator.Blueprint],
};
```

[blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L164-L219)

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

### 4.3 自定义属性校验（JSON Schema）

[customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts#L296-L386)

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

### 4.4 资源设置属性校验

[resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts#L63-L102)

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

### 4.5 资源通用属性校验

[resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1524-L1552)

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

### 4.6 Block 父节点类型校验

[block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.service.ts#L83-L103)

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

### 4.7 代码生成器可用性校验

[resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L415-L450)

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

---

## 五、版本过期告警（Outdated Version Alert）

### 5.1 告警触发

当模板发布新版本时触发告警：

[outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L179-L246)

```typescript
async triggerAlertsForTemplateVersion(templateResourceId, outdatedVersion, latestVersion) {
  // 1. 查找所有使用该模板的服务
  const services = await this.resourceService.resources({
    where: { serviceTemplateId: templateResourceId, project: { workspace: { id: project.workspaceId } } },
  });

  // 2. 为每个服务创建 TemplateVersion 类型的告警
  for (const service of services) {
    const currentTemplateVersion = await this.resourceService.getServiceTemplateSettings(service.id, null);
    await this.create({
      data: {
        resource: { connect: { id: service.id } },
        type: EnumOutdatedVersionAlertType.TemplateVersion,
        outdatedVersion: currentTemplateVersion.version,
        latestVersion,
      },
    }, template.name);
  }
}
```

### 5.2 告警类型
参考 [EnumOutdatedVersionAlertType.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto/EnumOutdatedVersionAlertType.ts)

- `TemplateVersion` - 模板版本过期
- `PluginVersion` - 插件版本过期

---

## 六、前端交互链路

### 6.1 GraphQL 查询定义

[serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts#L1-L113)

| 查询/Mutation | 用途 |
|--------------|------|
| `GET_SERVICE_TEMPLATES` | 获取项目中的模板列表 |
| `CREATE_SERVICE_TEMPLATE` | 创建新模板 |
| `CREATE_TEMPLATE_FROM_RESOURCE` | 从现有资源创建模板 |
| `GET_AVAILABLE_TEMPLATES_FOR_PROJECT` | 获取项目可用模板（含公开项目） |
| `UPGRADE_SERVICE_TO_LATEST_TEMPLATE_VERSION` | 将服务升级到最新模板版本 |

### 6.2 React Hook 封装

[useServiceTemplate.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts#L37-L190)

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

---

## 七、完整调用链路总结图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              前端 (amplication-client)                        │
│  useServiceTemplate Hook → GraphQL Mutations/Queries                         │
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
│                             │  - 计费/属性校验    │                            │
│                             └──────────┬──────────┘                            │
│                                        │                                       │
│            ┌───────────────────────────┼───────────────────────────┐          │
│            ▼                           ▼                           ▼          │
│  ┌────────────────────┐   ┌────────────────────────┐  ┌────────────────────┐ │
│  │ ServiceTemplate    │   │ ResourceVersionService │  │ OutdatedVersion    │ │
│  │ Service            │   │  - 创建快照            │  │ AlertService       │ │
│  │  - 创建/升级模板    │   │  - 版本对比            │  │  - 触发告警        │ │
│  │  - 从模板创建资源  │   │  - semver校验          │  │  - 解决告警        │ │
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
│                  │ Prisma (DB Layer)   │                                     │
│                  │ Block / BlockVersion│                                     │
│                  │ Resource / ResourceVersion│                                │
│                  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键数据实体关系

```
Project (1) ──┬──> Resource (ProjectConfiguration) ──> Block (ProjectConfigurationSettings)
              │
              ├──> Resource (ServiceTemplate) ──┬──> Block (ServiceSettings/PluginInstallation...)
              │                                   ├──> Block (CodeEngineVersion)
              │                                   └──> ResourceVersion (snapshot)
              │                                            ├──> EntityVersion[]
              │                                            └──> BlockVersion[]
              │
              └──> Resource (Service/Component) ──┬──> Block (ResourceTemplateVersion)
                                                    │     ├── serviceTemplateId
                                                    │     └── version
                                                    ├──> Block (ResourceSettings)
                                                    ├──> Block (ServiceSettings)
                                                    └──> OutdatedVersionAlert[]
```

---

## 九、关键文件索引

| 模块 | 服务层 | Resolver | DTO |
|------|--------|----------|-----|
| 项目配置 | [projectConfigurationSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.service.ts) | [projectConfigurationSettings.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/projectConfigurationSettings.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/projectConfigurationSettings/dto) |
| 项目 | [project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.service.ts) | [project.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/project.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/project/dto) |
| 资源/模板 | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | [resource.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/dto) |
| 服务模板 | [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | [serviceTemplate.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resource/serviceTemplate.resolver.ts) | - |
| 资源版本 | [resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts) | [resourceVersion.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceVersion/dto) |
| 资源模板版本 | [resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) | - | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceTemplateVersion/dto) |
| 模板代码引擎版本 | [templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts) | - | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/dto) |
| Blueprint | [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts) | [blueprint.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/blueprint.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/blueprint/dto) |
| Block | [block.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.service.ts) | [block.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/block.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/block/dto) |
| 自定义属性 | [customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts) | [customProperty.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/customProperty.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/customProperty/dto) |
| 资源设置 | [resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts) | [resourceSettings.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.resolver.ts) | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/resourceSettings/dto) |
| 版本过期告警 | [outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts) | - | [dto/](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto) |
| 前端 Hook | [useServiceTemplate.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/useServiceTemplate.ts) | - | [serviceTemplateQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/53-amplication/packages/amplication-client/src/ServiceTemplate/hooks/serviceTemplateQueries.ts) |
