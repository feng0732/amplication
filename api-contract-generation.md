# GraphQL / API 契约生成机制剖析

本文从代码实现角度，梳理 Amplication 中「资源配置」如何一步步影响最终生成的 **GraphQL 类型（Types）**、**Resolver** 和 **访问控制（ACL）**。

---

## 一、整体流程总览

代码生成从 [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/create-data-service.ts) 的 `createDataService()` 函数作为入口。核心执行阶段如下：

```
DSGResourceData（输入）
    │
    ▼
prepareContext()            // 1. 规范化实体、解析 Lookup 字段、组装 Actions/DTOs 映射
    │
    ▼
createDTOs()                // 2. 生成所有 GraphQL/REST 共享的 DTO 类型
    │
    ├── createServer()      // 3a. 生成服务端（Resolver / Controller / Service / ACL...）
    │       │
    │       └── createResourcesModules()
    │             │
    │             ├── createResolverModules()   // Resolver
    │             ├── createControllerModules() // REST Controller
    │             ├── createServiceModules()    // Service 业务层
    │             └── createGrantsModule()      // 访问控制 grants.json
    │
    └── createAdminModules() // 3b. 生成 Admin UI
```

关键入口：

- [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/create-data-service.ts)
- [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts)
- [create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-server.ts)

---

## 二、资源配置数据结构

代码生成的输入封装在 `DSGResourceData` 中，定义于 [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts)：

```ts
class DSGResourceData {
  resourceType: EnumResourceType;       // Service / MessageBroker 等
  resourceInfo: AppInfo;                // 服务基本信息 + 设置
  entities: Entity[];                   // 实体定义（含字段、权限）
  roles: Role[];                        // 角色列表
  moduleContainers: ModuleContainer[];  // 每个实体对应一个模块容器
  moduleActions: ModuleAction[];        // 具体的 CRUD / 自定义动作
  moduleDtos: ModuleDto[];              // 自定义 DTO
  pluginInstallations: PluginInstallation[];
  // ...
}
```

### 2.1 服务设置（ServiceSettings / ServerSettings）

在 [code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/code-gen-types.ts) 和 [models.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts) 中：

```ts
type ServerSettings = {
  generateGraphQL: boolean;   // 是否生成 GraphQL API
  generateRestApi: boolean;   // 是否生成 REST API
  generateServer?: boolean;   // 是否生成服务端代码
  serverPath: string;         // 服务端代码输出路径
};
```

这三个开关直接决定后续文件是否生成：

| 开关 | 作用位置 | 效果 |
|------|----------|------|
| `generateGraphQL` | [create-resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-resource.ts#L89-L98) | 为 `false` 时跳过 `createResolverModules()` |
| `generateRestApi` | [create-resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-resource.ts#L61-L71) | 为 `false` 时跳过 `createControllerModules()` |
| `generateServer` | [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/create-data-service.ts#L73-L81) | 为 `false` 时整个服务端不生成 |

---

## 三、资源配置 → GraphQL 类型（DTOs）

### 3.1 DTO 生成入口

DTO 生成由 [create-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-dtos.ts) 中的 `createDTOs()` 函数驱动，每个 Entity 会生成以下 DTO：

| DTO 种类 | 生成函数 | GraphQL 用途 |
|----------|----------|--------------|
| `Entity` (ObjectType) | [create-entity-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-entity-dto.ts) | GraphQL schema 的输出类型 |
| `CreateInput` | [create-create-input.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-create-input.ts) | Mutation create 的输入 |
| `UpdateInput` | [create-update-input.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-update-input.ts) | Mutation update 的输入 |
| `WhereInput` | [create-where-input.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-where-input.ts) | Query 过滤条件 |
| `WhereUniqueInput` | [create-where-unique-input.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-where-unique-input.ts) | 单条查询条件 |
| `OrderByInput` | [order-by-input.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/order-by-input/order-by-input.ts) | 排序输入 |
| `CreateArgs` | [create-create-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/create/create-create-args.ts) | `@Args()` 类型（GraphQL） |
| `UpdateArgs` | [create-update-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/update/create-update-args.ts) | 更新 Args |
| `FindManyArgs` | [create-find-many-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/find-many/create-find-many-args.ts) | 列表查询 Args |
| `FindOneArgs` | [create-find-one-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/find-one/create-find-one-args.ts) | 单体查询 Args |
| `DeleteArgs` | [create-delete-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/delete/create-delete-args.ts) | 删除 Args |
| `CountArgs` | [create-count-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql/count/create-count-args.ts) | 计数 Args |

### 3.2 实体字段（EntityField）如何映射为 GraphQL 字段

每个字段最终会通过 [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts) 的 `createFieldClassProperty()` 处理，它会同时完成：

1. **TypeScript 类型标注**
2. **GraphQL `@Field()` 装饰器**（由 [create-graphql-field-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql-field-decorator/create-graphql-field-decorator.ts) 生成）
3. **Swagger `@ApiProperty()` 装饰器**
4. **Class-Validator 验证装饰器**（`@IsString`、`@MaxLength` 等）

#### 字段数据类型映射（Prisma Scalar → GraphQL 类型）

在 [create-graphql-field-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql-field-decorator/create-graphql-field-decorator.ts#L68-L150) 中：

| Prisma ScalarType | GraphQL 返回类型 | 代码位置 |
|-------------------|-----------------|----------|
| `Boolean` | `Boolean` | L95-L97 |
| `DateTime` | `Date` | L98-L100 |
| `Int` / `Float` | `Number` | L101-L106 |
| `Decimal` | `Float` | L107-L109 |
| `BigInt` | `GraphQLBigInt`（自定义标量） | L110-L112 |
| `String` | `String` | L113-L115 |
| `Json` | `GraphQLJSON`（自定义标量） | L116-L118 |
| Enum | 对应枚举名 | L119-L122 |
| Lookup (Relation) | `WhereUniqueInput` 或嵌套输入类型 | L123-L148 |

#### 字段属性（properties）对 DTO 的影响

字段的 `properties` JSON 对象（如 `maxLength`、`minimumValue`、`maximumValue`）会直接转化为装饰器：

- **文本字段**：`maxLength` → `@MaxLength(N)`（[create-field-class-property.ts L293-L300](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L293-L300)）
- **数值字段**：`minimumValue` / `maximumValue` → `@Min(N)` / `@Max(N)`（[create-field-class-property.ts L279-L292](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L279-L292)）
- **必填性**：`field.required` + `optional` 参数 → GraphQL `nullable: true/false`
- **密码字段**：在 Entity DTO（输出类型）中被**完全排除**（[create-entity-dto.ts L14](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-entity-dto.ts#L14) 使用 `isPasswordField()` 过滤）

#### Lookup（关联）字段的特殊处理

Lookup 字段在 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261) 的 `resolveLookupFields()` 中预先解析，填充 `relatedEntity`、`relatedField`、`isOneToOneWithoutForeignKey`、`fkFieldName` 等运行时属性。

对于 to-Many 关联，DTO 类型会根据上下文切换（[create-field-class-property.ts L409-L438](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L409-L438)）：

- `CreateInput` → `CreateNestedManyWithoutInput`
- `UpdateInput` → `UpdateManyWithoutInput`
- `WhereInput` → `EntityListRelationFilter`
- Entity ObjectType → 关联实体类型本身

### 3.3 自定义 DTO 与装饰器

除了默认生成的 DTO 外，`ModuleDto`（自定义 DTO）还会在 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L384-L486) 的 `prepareModuleActionsAndDtos()` 中根据被引用的 Action 自动添加 GraphQL 装饰器：

- 作为 Action 输入 → 添加 `@ArgsType()` 装饰器，嵌套属性添加 `@InputType()`
- 作为 Action 输出 → 添加 `@ObjectType()` 装饰器（递归到嵌套 DTO）

---

## 四、资源配置 → Resolver 生成

### 4.1 Resolver 模块结构

每个 Entity 生成两个文件：

1. **`{entity}.resolver.base.ts`**：自动生成的基类，包含所有默认 CRUD 方法和关联方法
2. **`{entity}.resolver.ts`**：用户可编辑的子类，继承 Base，用于扩展自定义逻辑

模板文件：

- [resolver.base.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/resolver.base.template.ts)
- [resolver.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/resolver.template.ts)

### 4.2 默认 CRUD Resolver 方法

基类模板预定义了 6 个标准 GraphQL 端点（[resolver.base.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/resolver.base.template.ts)）：

| 方法 | GraphQL 操作 | 对应 Action | 调用的 Service 方法 |
|------|-------------|-------------|---------------------|
| `META_QUERY` | `Query meta(name)` | `Search` | `service.count()` |
| `ENTITIES_QUERY` | `Query [Entity]` | `Search` | `service.findMany()` |
| `ENTITY_QUERY` | `Query Entity` (nullable) | `View` | `service.findOne()` |
| `CREATE_MUTATION` | `Mutation Entity` | `Create` | `service.create()` |
| `UPDATE_MUTATION` | `Mutation Entity` (nullable) | `Update` | `service.update()` |
| `DELETE_MUTATION` | `Mutation Entity` (nullable) | `Delete` | `service.delete()` |

方法名由 `entityActionsMap` 决定，该 Map 在 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L263-L382) 的 `prepareEntityActions()` 中构建。

### 4.3 ModuleAction / ModuleContainer 对 Resolver 的影响

每个 Entity 对应一个 `ModuleContainer`（模块容器），其下挂着若干 `ModuleAction`（动作）。在 `prepareEntityActions()` 中：

1. **默认动作**：如果 `ModuleContainer` 或数据库中不存在对应 Action，使用 `getDefaultActionsForEntity()` 生成默认（[prepare-context.ts L270](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L270)）
2. **关联字段动作**：对每个 Lookup 字段生成 `ParentGet` / `ChildrenFind` / `ChildrenConnect` / `ChildrenDisconnect` / `ChildrenUpdate` 等默认动作
3. **模块禁用（`enabled: false`）**：所有动作的 `enabled` 被强制置为 `false`（[prepare-context.ts L347-L369](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L347-L369)）

生成 Resolver Base 时会根据 `action.enabled` 动态删除方法（[create-resolver.ts L376-L381](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L376-L381)）：

```ts
Object.keys(entityActions.entityDefaultActions).forEach((key) => {
  const action: ModuleAction = entityActions.entityDefaultActions[key];
  if (action && !action.enabled) {
    removeClassMethodByName(classDeclaration, action.name);
  }
});
```

关联字段方法同理处理（[create-resolver.ts L350-L374](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L350-L374)）。

### 4.4 自定义 Action Resolver

`ModuleAction.actionType === EnumModuleActionType.Custom` 的动作会通过 [create-resolver-custom-actions.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver-custom-actions.ts) 以独立方法形式插入 Resolver Base 类，GraphQL 操作装饰器由 [create-graphql-operation-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-graphql-operation-decorator.ts) 生成：

- `action.gqlOperation === Query` → `@graphql.Query(() => 返回类型)`
- `action.gqlOperation === Mutation` → `@graphql.Mutation(() => 返回类型)`

返回类型由 `action.outputType`（`PropertyTypeDef`）通过 `convertTypeDefToGraphQLType()` 转换得到。

### 4.5 关联字段 Resolver 方法

- **ToOne 关联**（一对一）：模板 [to-one.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/to-one.template.ts)，方法名为 `get{FieldName}`，对应 `EnumEntityAction.View`
- **ToMany 关联**（一对多）：模板 [to-many.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/to-many.template.ts)，方法名为 `find{FieldName}`，对应 `EnumEntityAction.Search`

两者都在 [create-resolver.ts L434-L556](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L434-L556) 中生成并追加到 Resolver Base 类。

---

## 五、资源配置 → 访问控制（ACL）

Amplication 的访问控制分**两层**实现：

1. **请求拦截层**：通过 NestJS 装饰器（`@UseRoles`、拦截器）在 Resolver/Controller 方法上
2. **授权规则层**：通过 `accesscontrol` 库的 `grants.json` 配置，支持细粒度到字段级的权限

### 5.1 EntityPermission 数据结构

每个 Entity 含有一组 `EntityPermission`（[models.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts#L731-L739)）：

```ts
type EntityPermission = {
  action: EnumEntityAction;          // Create / Delete / Search / Update / View
  type: EnumEntityPermissionType;    // AllRoles / Disabled / Granular / Public
  permissionFields?: EntityPermissionField[];  // 字段级权限（Granular 时使用）
  permissionRoles?: EntityPermissionRole[];    // 角色级权限（Granular 时使用）
};
```

四种权限类型（`EnumEntityPermissionType`）：

| 类型 | 含义 | grants.json 生成逻辑 |
|------|------|---------------------|
| `Disabled` | 完全禁止 | 跳过，不生成任何 Grant |
| `Public` | 公开访问 | Grant 不写入（由 `@Public()` 装饰器处理） |
| `AllRoles` | 所有登录角色均可 | 为每个 Role 生成一条 Grant，attributes=`*` |
| `Granular` | 细粒度（指定角色 + 指定字段） | 为指定 Role 生成 Grant，通过负向匹配（`!fieldName`）剔除无权访问的字段 |

### 5.2 grants.json 生成逻辑

核心实现位于 [create-grants.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts) 的 `createGrants()` 函数。

动作映射关系（[create-grants.ts L146-L152](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts#L146-L152)）：

| `EnumEntityAction` | ACL Action |
|--------------------|-----------|
| `Create` | `create:any` |
| `Delete` | `delete:any` |
| `Search` | `read:any` |
| `Update` | `update:any` |
| `View` | `read:own` |

#### Granular 权限的字段级控制

当 `permissionFields` 存在时，DSG 会构建 `roleToFields` 映射（角色 → 可访问字段集合），然后通过**负向 glob** 实现：

```ts
// 允许所有字段，但排除 forbiddenFields
const attributes = createAttributes([
  "*",
  ...forbiddenFields.map(field => `!${field}`)
]);
// 例如："*,!password,!secretField"
```

详见 [create-grants.ts L97-L121](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts#L97-L121)。

### 5.3 Resolver 方法的装饰器注入

访问控制的**请求拦截层**由 [set-endpoint-permission.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/utils/set-endpoint-permission.ts) 的 `setEndpointPermissions()` 实现，该函数在每个 Resolver/Controller 方法上被调用（如 [create-resolver.ts L340-L342](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L340-L342)）。

其核心逻辑：如果某 Action 被配置为 `Public` 类型：

1. **Search / View**（读操作）：移除 `AclFilterResponseInterceptor`（不再过滤响应字段）
2. **Create / Update**（写操作）：移除 `AclValidateRequestInterceptor`（不再验证请求字段）
3. 移除 `@UseRoles()` 装饰器（不再强制登录）
4. 添加 `@Public()` 装饰器（标记为公开端点）

非 Public 的方法保留默认的 `@UseRoles()` + 两个 ACL 拦截器组合，由运行时的 `accesscontrol` + `grants.json` 做最终授权判断。

方法与 Action 的对应关系（[create-resolver.ts L307-L338](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L307-L338)）：

| Resolver 方法 | `EnumEntityAction` |
|--------------|--------------------|
| `CREATE_MUTATION` | `Create` |
| `ENTITIES_QUERY` | `Search` |
| `META_QUERY` | `Search` |
| `ENTITY_QUERY` | `View` |
| `UPDATE_MUTATION` | `Update` |
| `DELETE_MUTATION` | `Delete` |
| ToOne `getXxx` 方法 | `View`（关联实体） |
| ToMany `findXxx` 方法 | `Search`（关联实体） |

---

## 六、关键代码索引

| 关注点 | 核心文件 |
|--------|---------|
| DSG 主入口 | [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/create-data-service.ts) |
| 上下文准备（解析实体、组装 Actions） | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts) |
| 服务端代码生成总入口 | [create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-server.ts) |
| 实体模块生成（Service/Controller/Resolver） | [create-resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-resource.ts) |
| DTO 总生成入口 | [create-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-dtos.ts) |
| 字段属性 → TS/GraphQL 类型/装饰器 | [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts) |
| GraphQL `@Field()` 装饰器生成 | [create-graphql-field-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql-field-decorator/create-graphql-field-decorator.ts) |
| Entity ObjectType DTO | [create-entity-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-entity-dto.ts) |
| Resolver 生成 | [create-resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts) |
| Resolver 基类模板 | [resolver.base.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/resolver.base.template.ts) |
| 自定义 GraphQL 操作装饰器 | [create-graphql-operation-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-graphql-operation-decorator.ts) |
| ACL grants.json 生成 | [create-grants.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts) |
| 端点权限装饰器注入 | [set-endpoint-permission.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/utils/set-endpoint-permission.ts) |
| 代码生成输入类型 | [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts) |
| 核心类型（Entity/Field/Permission） | [code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/code-gen-types.ts)、[models.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts) |
