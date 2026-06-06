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

### 4.6 Resolver 方法装饰器的真实状态（与 Controller 对比）

这是容易理解错误的关键点：**Resolver 和 Controller 的初始装饰器完全不同**。

| 装饰器 / 拦截器 | Resolver Base 模板初始状态 | Controller Base 模板初始状态 |
|-----------------|--------------------------|----------------------------|
| `@graphql.Query` / `@graphql.Mutation` | ✅ 存在于 Resolver 模板 | ❌ 不存在 |
| `@common.Get` / `@common.Post` 等 REST 装饰器 | ❌ 不存在 | ✅ 存在于 Controller 模板 |
| `@nestAccessControl.UseRoles({...})` | ❌ **不存在** | ❌ 模板中不存在（但生成后通过运行时或其他链路注入） |
| `@common.UseInterceptors(AclFilterResponseInterceptor)` | ❌ **不存在** | ❌ 模板中不存在 |
| `@common.UseInterceptors(AclValidateRequestInterceptor)` | ❌ **不存在** | ❌ 模板中不存在 |
| `@Public()` | ❌ 不存在，仅当 Action 为 Public 时由 `setEndpointPermissions()` 动态添加 | ❌ 不存在，仅当 Action 为 Public 时动态添加 |

**结论**：GraphQL Resolver 的 6 个 CRUD 方法和关联方法在代码生成后，**默认只有 GraphQL 原生装饰器**（`@Query`/`@Mutation`/`@ResolveField`/`@Args`/`@Parent`），没有任何 ACL 相关的 `@UseRoles` 或拦截器装饰器。`setEndpointPermissions()` 对 Resolver 的**唯一实际作用**是在权限类型为 Public 时追加 `@Public()` 装饰器——其他"移除拦截器/移除 UseRoles"操作均为空操作（因为本来就不存在这些装饰器）。

测试快照验证（以 Customer 实体为例）：Resolver Base 的 `customers()`、`customer()`、`createCustomer()`、`updateCustomer()`、`deleteCustomer()` 五个方法上只有 `@graphql.Query` / `@graphql.Mutation`，没有任何 ACL 装饰器；只有关联方法 `findOrders()` 上出现了 `@Public()`（因为关联实体 Order 的 Search 权限是 Public）。

---

## 五、资源配置 → 访问控制（ACL）

Amplication 的访问控制分**两层**实现：

1. **端点标记层**：通过 NestJS 装饰器（`@Public()` 标记公开端点、`@UseRoles` 声明角色权限要求、ACL 拦截器执行校验）
2. **授权规则层**：通过 `accesscontrol` 库的 `grants.json` 配置，支持细粒度到字段级的权限

> **注意**：ACL 装饰器（`@UseRoles` + 两个拦截器）在 GraphQL Resolver 生成代码中**不出现**，仅在 REST Controller 生成代码中出现。GraphQL 端点的运行时授权依赖其他机制（如全局 Guard、模块级配置），或依赖后续版本的补充。

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

### 5.3 端点权限装饰器注入（setEndpointPermissions 精确行为）

访问控制的端点标记由 [set-endpoint-permission.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/utils/set-endpoint-permission.ts) 的 `setEndpointPermissions()` 实现。该函数在以下位置被调用：

- Resolver Base 的 6 个 CRUD 方法：[create-resolver.ts L340-L342](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L340-L342)
- Resolver 的 ToOne / ToMany 关联方法：[create-resolver.ts L487-L492](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L487-L492) 和 [create-resolver.ts L548-L553](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L548-L553)
- Controller Base 的 5 个 CRUD 方法：[create-controller.ts L320-L322](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/controller/create-controller.ts#L320-L322)
- Controller 的 ToMany 关联方法：[create-controller.ts L478-L480](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/controller/create-controller.ts#L478-L480)

#### 函数实际执行逻辑（逐行解读）

```ts
function setEndpointPermissions(classDeclaration, methodId, action, entity) {
  // 1. 找方法
  const classMethod = getClassMethodById(classDeclaration, methodId);
  if (!classMethod) return;

  // 2. 查权限：不是 Public → 直接 return，什么都不做
  if (!isPublicEntity(entity, action)) return;

  // 3. Search / View 读操作 → 移除响应过滤拦截器
  if (action === EnumEntityAction.Search || action === EnumEntityAction.View) {
    removeIdentifierFromUseInterceptorDecorator(
      classMethod, ACL_FILTER_RESPONSE_INTERCEPTOR_NAME
    );
  }

  // 4. Create / Update 写操作 → 移除请求校验拦截器
  if (action === EnumEntityAction.Create || action === EnumEntityAction.Update) {
    removeIdentifierFromUseInterceptorDecorator(
      classMethod, ACL_VALIDATE_REQUEST_INTERCEPTOR_NAME
    );
  }

  // 5. 移除 @UseRoles
  removeDecoratorByName(classMethod, USE_ROLES_DECORATOR_NAME);

  // 6. 再次移除 @UseRoles（防御性重复）
  removeDecoratorByName(classMethod, USE_ROLES_DECORATOR_NAME);

  // 7. 追加 @Public()
  classMethod.decorators?.unshift(createPublicDecorator());
}
```

#### 对 Resolver 的实际效果 vs 对 Controller 的实际效果

| 步骤 | Resolver（实际效果） | Controller（实际效果） |
|------|---------------------|----------------------|
| 3. 移除 `AclFilterResponseInterceptor` | **空操作**（模板里没有这个装饰器） | 如果模板有则移除 |
| 4. 移除 `AclValidateRequestInterceptor` | **空操作**（模板里没有这个装饰器） | 如果模板有则移除 |
| 5. 移除 `@UseRoles` | **空操作**（模板里没有这个装饰器） | 如果模板有则移除 |
| 6. 再次移除 `@UseRoles` | **空操作** | — |
| 7. 添加 `@Public()` | ✅ **唯一实际生效** | ✅ 实际生效 |

**关键结论**：`setEndpointPermissions()` 对 GraphQL Resolver 的唯一作用是——当某 Action 权限类型为 Public 时，在对应方法上追加 `@Public()` 装饰器。对于非 Public 的 Resolver 方法，函数在第 2 步就直接 `return`，**不做任何修改**。

#### Resolver 方法与 Action 的对应关系

[create-resolver.ts L307-L338](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L307-L338) 中定义：

| Resolver 方法 | `EnumEntityAction` |
|--------------|--------------------|
| `CREATE_MUTATION` | `Create` |
| `ENTITIES_QUERY` | `Search` |
| `META_QUERY` | `Search` |
| `ENTITY_QUERY` | `View` |
| `UPDATE_MUTATION` | `Update` |
| `DELETE_MUTATION` | `Delete` |
| ToOne `getXxx` 方法 | `View`（关联实体的权限） |
| ToMany `findXxx` 方法 | `Search`（关联实体的权限） |

---

## 六、插件扩展点（Plugin Hooks）

Amplication 的代码生成引擎在每一个关键生成环节都暴露了 **Before/After** 双阶段插件钩子，允许第三方插件在默认行为执行前后介入、修改输入参数或替换输出模块。

### 6.1 插件系统架构

插件系统的核心组件：

| 组件 | 位置 | 作用 |
|------|------|------|
| `pluginWrapper` | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts) | 每个生成函数的执行包装器，负责调度 Before/After 钩子 |
| `registerPlugins` | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/register-plugin.ts) | 动态加载插件 npm 包，组装成 `PluginMap` |
| `EventNames` | [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugins.types.ts#L77-L124) | 全部 47 个可挂钩事件名的枚举 |
| `Events` 类型 | [plugin-events.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugin-events.types.ts) | 每个事件对应的参数类型约束 |
| `AmplicationPlugin` | [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugins.types.ts#L126-L129) | 插件必须实现的接口（含 `register()` 方法） |

#### 插件执行流水线

`pluginWrapper` 的执行顺序（[plugin-wrapper.ts L59-L99](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L99)）：

```
1. context.utils.skipDefaultBehavior = false
2. ── beforePlugins pipe ──→ 依次执行所有 Before 钩子，eventParams 可被逐步修改
3. ── defaultBehavior ──→    调用原始 DSG 生成函数（可被 skipDefaultBehavior 跳过）
4. ── afterPlugins pipe ──→ 依次执行所有 After 钩子，可修改/新增/替换输出 ModuleMap
5. ── 写入 context.modules
```

关键机制：
- **Before 钩子链式传递**：使用 `beforeEventsPipe` 通过 reduce 将每个插件的返回值作为下一个插件的输入（[plugin-wrapper.ts L17-L23](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L17-L23)）
- **跳过默认行为**：Before 钩子可设置 `context.utils.skipDefaultBehavior = true`，此时默认函数完全不执行（[plugin-wrapper.ts L40-L51](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L40-L51)）
- **After 钩子合并**：After 钩子可返回新的 `ModuleMap`，最终 `for...of` 遍历 upsert 到 `context.modules`（[plugin-wrapper.ts L95-L97](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L95-L97)）

### 6.2 与 GraphQL 契约直接相关的插件事件

以下是直接影响 GraphQL 类型、Resolver 和访问控制的关键插件点。

#### GraphQL 类型层（DTOs）

| EventName | 参数类型 | 可介入时机 | 影响范围 |
|-----------|---------|------------|---------|
| `CreateDTOs` | `CreateDTOsParams`（仅含 `dtos` + `dtoNameToPath`） | Before + After | 所有 Entity 相关 DTO（Entity/CreateInput/WhereInput 等），可修改 AST 节点、新增 DTO、重命名路径 |
| `CreatePrismaSchema` | `CreatePrismaSchemaParams` | Before + After | Prisma Schema，会间接影响生成的所有 DTO 类型 |

> **重要纠正**：`CreateDTOsParams` 的**真实参数只有两个字段**（[plugin-events-params.types.ts L353-L356](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugin-events-params.types.ts#L353-L356)）：
> - `dtos: DTOs` —— 已生成的 DTO AST 节点集合
> - `dtoNameToPath: Record<string, string>` —— DTO 类名到输出文件路径的映射
>
> **参数中没有 Entity！** 插件无法在此事件上直接修改原始实体配置，只能操作已生成的 AST 节点。

##### DTOs 数据结构详解

DTOs 是一个三层嵌套的 AST 节点集合（[code-gen-types.ts L194-L216](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L194-L216)）：

```
DTOs {
  [entityName: string]: EntityEnumDTOs & EntityDTOs
}
├── EntityDTOs
│   ├── entity: NamedClassDeclaration           // @ObjectType() —— 主实体输出类型
│   ├── createInput: NamedClassDeclaration      // @InputType() —— 创建输入
│   ├── updateInput: NamedClassDeclaration      // @InputType() —— 更新输入
│   ├── whereInput: NamedClassDeclaration       // @InputType() —— 过滤条件
│   ├── whereUniqueInput: NamedClassDeclaration // @InputType() —— 唯一键条件
│   ├── deleteArgs: NamedClassDeclaration       // @ArgsType() —— 删除参数
│   ├── countArgs: NamedClassDeclaration        // @ArgsType() —— 计数参数
│   ├── findManyArgs: NamedClassDeclaration     // @ArgsType() —— 列表查询参数
│   ├── findOneArgs: NamedClassDeclaration      // @ArgsType() —— 单体查询参数
│   ├── createArgs?: NamedClassDeclaration      // @ArgsType() —— 创建参数
│   ├── updateArgs?: NamedClassDeclaration      // @ArgsType() —— 更新参数
│   ├── orderByInput: NamedClassDeclaration     // @InputType() —— 排序输入
│   └── listRelationFilter: NamedClassDeclaration // @InputType() —— 关联过滤
└── EntityEnumDTOs
    └── [enumName: string]: TSEnumDeclaration   // TS 枚举（Entity 字段枚举）
```

每个 `NamedClassDeclaration` 是 `ast-types` 的 TS AST 节点，包含：
- `id: Identifier` —— 类名
- `decorators: Decorator[]` —— 类装饰器（`@ObjectType`、`@InputType` 等）
- `body: ClassBody` —— 类体，内含 `ClassProperty` 节点（每个字段一个）

##### CreateDTOs Before / After 钩子的真实介入方式

`CreateDTOs` 事件由 [create-dtos.ts L36-L44](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-dtos.ts#L36-L44) 中的 `createDTOModules()` 触发。

对照 [plugin-wrapper.ts L59-L99](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L99) 的实际执行流程：

```
createDTOs(entities)           // 1. 先根据 Entity[] 生成 DTOs（AST 节点集合）
getDTONameToPath(dtos)         // 2. 生成 dtoName → 文件路径映射
         │
         ▼
原始 args = { dtos, dtoNameToPath }
         │
         ▼
beforeEventsPipe(原始 args)    // ◀─ Before 钩子接收原始 args，可返回修改后对象
         │
         ▼
updatedEventParams             // 3. Before 钩子链式处理后的最终参数（可能是全新对象）
         │
         ▼
defaultBehavior(updatedEventParams)
  └── createDTOModulesInternal()  // 4. 用 updatedEventParams 中的 AST 编译成 Module{path, code}
         │
         ▼
defaultBehaviorModules = ModuleMap   // 5. 已生成的文件集合（存的是 code 字符串，不是 AST）
         │
         ▼
afterEventsPipe(
  context,
  原始 args,        // ◀─ 关键：After 钩子收到的是原始 args，不是 updatedEventParams
  defaultBehaviorModules
)
```

| 钩子 | 接收的 eventParams | 可操作对象 | 能做什么 | 不能做什么 / 边界 |
|------|------------------|-----------|---------|------------------|
| **Before** | 原始 `{ dtos, dtoNameToPath }`，返回值传递给下一个 Before 钩子，最终传给 defaultBehavior | `dtos`（AST 节点集合）、`dtoNameToPath`（路径映射） | 修改已有 ClassProperty 的装饰器/类型；往已有 ClassDeclaration 追加/删除 ClassProperty；往 `dtos` 中新增自定义 NamedClassDeclaration；返回全新的 dtos 对象替换参数 | 访问原始 Entity（参数中没有）；直接操作最终文件 |
| **After** | **原始 `{ dtos, dtoNameToPath }`**（plugin-wrapper.ts L88 传的是原始 `args`，不是 `updatedEventParams`） | **主要是** `ModuleMap`（文件集合） | 修改已有文件代码；新增/替换/删除文件模块；通过 `moduleMap.replaceModulesCode()` 批量修改代码字符串 | **关键边界**：虽然 After 钩子能**读到** `eventParams.dtos`，但修改它**不会自动改变已生成的 ModuleMap**——因为 `defaultBehavior` 已经用 updatedEventParams 把 AST 编译成了字符串存入 ModuleMap，两者之间没有引用关系。要让 DTO AST 的修改生效，必须**手动重新编译 AST 为 code 字符串并调用 `moduleMap.set()` 覆盖**，或改用 Before 钩子修改。 |

##### DTO AST 与 ModuleMap 的生效时机边界（重要）

这是插件开发中最容易踩坑的点，必须明确区分：

| 阶段 | 数据形态 | 存储形式 | 修改是否自动生效 |
|------|---------|---------|---------------|
| Before 钩子期间 | `dtos` 中的 AST 节点（NamedClassDeclaration / ClassProperty / Decorator 等） | 内存中的 `ast-types` AST 对象引用 | ✅ 自动生效——Before 钩子的返回值会直接传给 `defaultBehavior`，后续编译会用到修改后的 AST |
| defaultBehavior 执行期间 | AST → 字符串的编译过程 | 调用 `createDTOModule()` / `createEnumDTOModule()` 将每个 AST 节点通过 TS printer 渲染为 `code: string`，封装成 Module | — |
| After 钩子期间 | `ModuleMap` 中的 Module | `{ path: string, code: string }` 的纯数据结构，字符串里是完整的 TS 文件代码 | ✅ 通过 `moduleMap.set()` / `moduleMap.replaceModulesCode()` 修改自动生效 |
| After 钩子期间（陷阱） | `eventParams.dtos` 中的 AST 节点 | 原始内存引用 | ❌ **不自动生效**——AST 和 ModuleMap 之间已断开引用。即使原地修改了某个 ClassProperty，已生成的 `code` 字符串不会随之变化 |

> **正确做法示例**：
> - 想修改某个字段的装饰器 → 在 **Before** 钩子中操作 `dtos.Customer.entity.body.body[i].decorators.push(...)`
> - 想替换某个已生成 DTO 的文件内容 → 在 **After** 钩子中调用 `moduleMap.replaceModulesCode((path, code) => path.includes('customer.dto') ? newCode : code)`
> - 想新增一个完全自定义的 GraphQL 类型 → 要么在 **Before** 钩子中往 `dtos` push 新 AST + 更新 `dtoNameToPath`（让 defaultBehavior 编译它），要么在 **After** 钩子中直接构造 Module 对象并 `moduleMap.set(module)` 注入

##### ModuleMap 数据结构与操作方式

`ModuleMap` 是 `FileMap<string>` 的子类（[code-gen-types.ts L151-L178](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L151-L178)），本质是 `path → Module` 的有序 Map。每个 Module 包含：

- `path: string` —— 输出文件的相对路径（如 `"src/customer/base/customer.resolver.base.ts"`）
- `code: string` —— 文件源代码（已由 AST printer 渲染的字符串）

After 钩子中对 ModuleMap 的典型操作：

| API | 用途 |
|-----|------|
| `moduleMap.set(module)` | 新增或覆盖一个文件模块 |
| `moduleMap.modules()` | 获取所有模块数组，用于遍历修改 |
| `moduleMap.replaceModulesCode((path, code) => newCode)` | 批量替换所有模块的代码字符串 |
| `moduleMap.replaceModulesPath((path) => newPath)` | 批量重命名所有模块的输出路径 |
| `moduleMap.get(path)` | 按路径获取单个模块 |

#### Resolver 层

| EventName | 参数类型 | 可介入时机 | 影响范围 |
|-----------|---------|------------|---------|
| `CreateEntityResolverBase` | `CreateEntityResolverBaseParams` | Before + After | Resolver 基类（默认 CRUD + 关联方法），可增删方法、修改装饰器、改返回类型 |
| `CreateEntityResolver` | `CreateEntityResolverParams` | Before + After | Resolver 可编辑子类（继承基类），可添加自定义方法 |
| `CreateEntityResolverToManyRelationMethods` | `CreateEntityResolverToManyRelationMethodsParams` | Before + After | 一对多关联的 find/connect/disconnect 方法 |
| `CreateEntityResolverToOneRelationMethods` | `CreateEntityResolverToOneRelationMethodsParams` | Before + After | 一对一关联的 get 方法 |
| `CreateEntityService` / `CreateEntityServiceBase` | 对应 Params | Before + After | Service 层（Resolver 调用的后端逻辑），修改 Service 会联动 Resolver 的调用签名 |

#### 插件参数类型定义与实际传递的不一致（重要）

以下是代码审查发现的**参数类型不一致**问题，插件开发者需特别注意：

##### `CreateEntityResolverBaseParams` —— 多传了 `entityDTO`

| 字段 | 类型定义 ([plugin-events-params.types.ts L321-L337](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugin-events-params.types.ts#L321-L337)) | 实际传递 ([create-resolver.ts L173-L190](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L173-L190)) | 实际接收函数签名 ([create-resolver.ts L253-L269](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts#L253-L269)) |
|------|---------------------|------------------|-------------------|
| `template` | ✅ | ✅ | ✅ |
| `entityName` | ✅ | ✅ | ✅ |
| `entityType` | ✅ | ✅ | ✅ |
| `entityServiceModule` | ✅ | ✅ | ✅ |
| `entity` | ✅ | ✅ | ✅ |
| **`entityDTO`** | ❌ **类型定义中没有** | ✅ **实际被传入** | ❌ **函数不接收** |
| `serviceId` | ✅ | ✅ | ✅ |
| `resolverBaseId` | ✅ | ✅ | ✅ |
| `createArgs` | ✅ | ✅ | ✅ |
| `updateArgs` | ✅ | ✅ | ✅ |
| `createMutationId` | ✅ | ✅ | ✅ |
| `updateMutationId` | ✅ | ✅ | ✅ |
| `templateMapping` | ✅ | ✅ | ✅ |
| `moduleContainers` | ✅ | ✅ | ✅ |
| `entityActions` | ✅ | ✅ | ✅ |
| `dtoNameToPath` | ✅ | ✅ | ✅ |

**影响**：`entityDTO` 字段通过 `as CreateEntityResolverBaseParams` 类型断言被偷偷传入 `pluginWrapper`，Before 钩子的 `eventParams` 中会包含此字段，但 TS 类型系统不感知。插件如果依赖该字段，需自行断言。

##### 其他验证一致的事件

| 事件 | 类型定义 vs 实际传递 |
|------|---------------------|
| `CreateEntityResolver` | ✅ 完全一致 |
| `CreateDTOs` | ✅ 完全一致 |
| `CreateEntityControllerBase` | ✅ 完全一致（对照 [create-controller.ts L176-L190](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/controller/create-controller.ts#L176-L190)） |

#### 访问控制层

| EventName | 参数类型 | 可介入时机 | 影响范围 |
|-----------|---------|------------|---------|
| `CreateServerAuth` | `CreateServerAuthParams` | Before + After | JWT/Auth 模块生成，影响登录态校验 |
| `CreateSeed` | `CreateSeedParams` | Before + After | 种子数据（含默认用户角色），间接影响 ACL 的可用角色 |

#### 全局层

| EventName | 参数类型 | 可介入时机 | 影响范围 |
|-----------|---------|------------|---------|
| `CreateServerAppModule` | `CreateServerAppModuleParams` | Before + After | AppModule 的 imports，可注入新的 GraphQL Module |
| `CreateServer` | `CreateServerParams` | Before + After | 整个服务端生成总入口，可完全替换行为 |
| `LoadStaticFiles` | `LoadStaticFilesParams` | Before + After | 静态资源文件，可注入额外的拦截器或公共类型 |

### 6.3 插件对 GraphQL 契约的典型影响方式

插件通过介入这些事件可以（以下方式均结合代码实际参数验证）：

1. **新增自定义 GraphQL 类型**：在 `CreateDTOs` 的 **Before** 钩子中往 `dtos[entityName]` 对象上追加新的 `NamedClassDeclaration` AST 节点，并同步更新 `dtoNameToPath` 映射（这样 `createDTOModulesInternal` 才会把它编译成文件）。或者在 **After** 钩子中通过 `moduleMap.set()` 直接注入新的 Module 文件。
2. **替换 Resolver 方法**：在 `CreateEntityResolverBase` 的 After 钩子中用 TS AST 遍历 `classDeclaration.body.body`，找到目标方法后替换 `ClassMethod` 节点
3. **为 DTO 字段添加装饰器**：在 `CreateDTOs` 的 **Before** 钩子中操作 `dtos` 中的 AST——找到目标 `NamedClassDeclaration.body.body` 里的 `ClassProperty`，直接 push 新的 Decorator 节点（如 GraphQL `@Field`、验证器 `@IsOptional`、或 `@ApiProperty`）。注意：CreateDTOs 参数中没有 Entity，不能通过修改 Entity 来间接影响字段。
4. **为 Resolver 追加 ACL 装饰器**：由于 DSG 默认不在 Resolver 方法上生成 `@UseRoles` 和 ACL 拦截器（参见第四章 4.6 节），插件可在 `CreateEntityResolverBase` 的 After 钩子中通过 AST 手动为方法添加 `@UseInterceptors(AclFilterResponseInterceptor)`、`@UseRoles({...})` 等装饰器（需同时确保 import 语句被追加）
5. **新增 Resolver 方法**：在 After 钩子中 push 新的 `ClassMethod`（带 `@Query`/`@Mutation` 装饰器）到 Resolver 基类
6. **完全替换默认行为**：在任意事件的 Before 钩子中设置 `context.utils.skipDefaultBehavior = true`，然后由插件自行生成 ModuleMap 返回给 After 钩子处理（常用于完全替换某个 Entity 的 DTO 或 Resolver 生成逻辑）

---

## 七、自定义 DTO 与自定义模块接入流程

除了基于 Entity 的标准 CRUD，Amplication 还支持完全自定义的「模块」（`ModuleContainer` + `ModuleAction` + `ModuleDto`），用于实现业务自定义的 GraphQL Query/Mutation。

### 7.1 数据结构总览

三个核心类型定义于 [models.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts)：

| 类型 | 关键字段 | 作用 |
|------|---------|------|
| `ModuleContainer` | `name`、`enabled`、`entityId?` | 模块容器；`entityId === undefined` 即为自定义模块 |
| `ModuleAction` | `name`、`enabled`、`actionType`、`gqlOperation`、`inputType`、`outputType`、`restVerb`、`restInputSource` | 模块内的动作（GraphQL Query/Mutation），含输入输出类型定义 |
| `ModuleDto` | `name`、`dtoType`、`properties?`、`members?`、`decorators?`、`enabled` | 自定义 DTO；`dtoType` 分 `Custom`（类）和 `CustomEnum`（枚举） |
| `ModuleDtoProperty` | `name`、`isOptional`、`isArray`、`propertyTypes[]` | 自定义 DTO 的属性，`propertyTypes` 是 `PropertyTypeDef[]` 支持联合类型 |

`PropertyTypeDef` 支持 6 种类型源（`EnumModuleDtoPropertyType`）：

```
Primitive  → 基础类型（String/Number/Boolean/Date/Json）
Dto        → 引用另一个 ModuleDto（嵌套对象）
Entity     → 引用系统 Entity（使用其 ObjectType）
Enum       → 引用系统 Entity 的字段枚举
CustomEnum → 引用另一个自定义 Enum DTO
NotFound   → 占位
```

### 7.2 在 prepare-context 中的组装

在 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts#L384-L486) 的 `prepareModuleActionsAndDtos()` 中完成两件事：

#### （1）按 ModuleContainer 维度分组

将扁平的 `moduleActions` 和 `moduleDtos` 组装成 `moduleActionsAndDtoMap`（以 `moduleContainer.name` 为 key）。

#### （2）根据 Action 引用自动为 DTO 注入 GraphQL 装饰器

遍历每个 `ModuleAction`，根据其 `inputType` 和 `outputType` 递归地给被引用的 `ModuleDto` 追加 `decorators`：

- **作为 Action 的输入类型** → 顶层 DTO 添加 `ArgsType` 装饰器，其嵌套属性 DTO 添加 `InputType` 装饰器（递归）
- **作为 Action 的输出类型** → 顶层 DTO 添加 `ObjectType` 装饰器，其嵌套属性 DTO 也添加 `ObjectType` 装饰器（递归）

装饰器去重通过 `EnumModuleDtoDecoratorType` 枚举判断（`ArgsType`/`InputType`/`ObjectType`）。

### 7.3 自定义 DTO 的代码生成

自定义 DTO 由 [create-custom-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/custom-types/create-custom-dtos.ts) 的 `createCustomDtos()` 生成，过滤条件是：

```ts
dto.dtoType === EnumModuleDtoType.Custom || dto.dtoType === EnumModuleDtoType.CustomEnum
```

#### 类 DTO（Custom）

由 `createDto()` 生成（[create-custom-dtos.ts L97-L141](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/custom-types/create-custom-dtos.ts#L97-L141)），每个 `ModuleDto.decorators` 会转成以下 GraphQL 装饰器：

| 装饰器类型 | 生成结果 |
|-----------|---------|
| `ArgsType` | `@ArgsType()` |
| `InputType` | `@InputType("${name}Input")` |
| `ObjectType` | `@ObjectType("${name}Object")` |

每个属性由 `createProperty()` 生成（[create-custom-dtos.ts L163-L208](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/custom-types/create-custom-dtos.ts#L163-L208)），同时添加：

- **GraphQL 端**：`createGraphQLFieldDecorator(property)`（仅当 `generateGraphQL` 为 true）
- **REST 端**：`createApiPropertyDecorator(property)`（仅当 `generateRestApi` 为 true）
- **类型转译**：`createTypeDecorator(property)`（用于 `class-transformer` 的嵌套对象反序列化）

#### 枚举 DTO（CustomEnum）

由 `createEnumDTO()` 生成 TS 原生枚举（`builders.tsEnumDeclaration`），每个成员是 `builders.tsEnumMember(identifier, stringLiteral(value))`。

### 7.4 自定义模块的 Resolver 生成

自定义模块（`moduleContainer.entityId === undefined`）由 [create-custom-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/create-custom-module.ts) 的 `createCustomModulesModules()` 处理，生成 4 类文件：

| 文件类型 | 生成函数 | 是否受开关控制 |
|---------|---------|--------------|
| Service | `createServiceModules()` | 始终生成 |
| Controller (REST) | `createCustomModuleControllerModules()` | 仅当 `generateRestApi` |
| Resolver (GraphQL) | `createCustomModuleResolverModules()` | 仅当 `generateGraphQL` |
| Module (NestJS Module) | `createCustomModule()` | 始终生成 |

#### 自定义 Resolver 详解

模板 [custom-resolver/resolver.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/resolver.template.ts) 是一个空壳（仅含构造函数注入 service），所有方法由 `createResolverCustomActionMethods()` 动态注入（[create-resolver-custom-actions.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/create-resolver-custom-actions.ts)）。

每个 `ModuleAction` 生成一个方法：

```ts
// 伪代码
@graphql.Query(/* 或 @graphql.Mutation */)
async {actionName}(@Args() args: {inputType}): Promise<{outputType}> {
  return this.service.{actionName}(args);
}
```

关键细节：
- 装饰器由 [create-graphql-operation-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/create-graphql-operation-decorator.ts) 根据 `action.gqlOperation`（Query/Mutation）生成
- `action.inputType.type !== Dto`（即原始类型）时，`@Args()` 不传递参数名；否则使用标准嵌套 Args 形式
- **自定义 Resolver 默认不生成 ACL 装饰器**（与 Entity Resolver 不同）——没有 `@UseRoles`、没有 `AclFilterResponseInterceptor`、没有 `AclValidateRequestInterceptor`，这是目前自定义模块的一个重要边界

---

## 八、公开权限（Public）与字段级过滤的边界

### 8.1 ACL 运行时的两层拦截器

最终运行时的权限控制由两个 NestJS Interceptor 完成，它们位于生成的服务端代码中（以 `data-service-generator-catalog` 为例）：

#### AclFilterResponseInterceptor —— 响应过滤（读操作）

代码：[aclFilterResponse.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/interceptors/aclFilterResponse.interceptor.ts)

执行流程：
1. 从 Reflector 读取当前方法挂载的 `roles` 元数据（由 `@UseRoles` 写入，包含 `role`、`action`、`possession`、`resource`）
2. 通过 `rolesBuilder.permission(...)` 从 `grants.json` 查询权限对象
3. 对响应数据调用 `permission.filter(data)` 进行字段级过滤
4. 数组响应 → 逐个元素过滤；单个对象 → 直接过滤

`permission.filter()` 是 `accesscontrol` 库的原生方法，会根据 attributes glob（如 `"*,!password,!secret"`）删除对象上不被允许的字段。

#### AclValidateRequestInterceptor —— 请求校验（写操作）

代码：[aclValidateRequest.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/interceptors/aclValidateRequest.interceptor.ts)

执行流程：
1. 同样读取 Reflector 中的 `roles` 元数据
2. 根据上下文类型获取输入数据：
   - HTTP (REST)：`context.switchToHttp().getRequest().body`
   - GraphQL：`context.getArgByIndex(1).data`（第二个参数是 `@Args()` 解析后的对象）
3. 调用 `abacUtil.getInvalidAttributes(permission, inputData)` 找出无权字段
4. 若有 `invalidAttributes.length > 0`，抛出 `ForbiddenException("Insufficient privileges to complete the operation")`

字段比对工具 [abac.util.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/auth/abac.util.ts) 的核心逻辑：

```ts
const filteredData = permission.filter(structuredClone(data));
return Object.keys(data).filter((key) => !(key in filteredData));
```

> **注意**：`structuredClone` 是必要的，因为 GraphQL 请求传入的对象原型为 `null`，而 `accesscontrol` 库的 `filter()` 对无原型对象处理异常。

### 8.2 Public 权限的精确边界（Resolver vs Controller 分别讨论）

`setEndpointPermissions()` 对 Public 权限的处理（[set-endpoint-permission.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/utils/set-endpoint-permission.ts#L16-L54)）必须**区分 GraphQL Resolver 和 REST Controller** 讨论，因为两者模板的初始装饰器状态完全不同。

#### Resolver（GraphQL）上的实际行为

由于 Resolver Base 模板里**初始没有任何 ACL 装饰器**（`@UseRoles`、两个拦截器均不存在），`setEndpointPermissions()` 对 Resolver 的执行效果是：

| Action | 代码分支 | 实际效果 |
|--------|---------|---------|
| Search | Search/View → 移除 FilterInterceptor | **空操作**（模板里没有） + 移除 @UseRoles（空操作）×2 + 添加 @Public() → **净效果：只追加 @Public()** |
| View | Search/View → 移除 FilterInterceptor | 同上 → **净效果：只追加 @Public()** |
| Create | Create/Update → 移除 ValidateInterceptor | **空操作**（模板里没有） + 移除 @UseRoles（空操作）×2 + 添加 @Public() → **净效果：只追加 @Public()** |
| Update | Create/Update → 移除 ValidateInterceptor | 同上 → **净效果：只追加 @Public()** |
| **Delete** | **两个 if 都不匹配** → 不移除任何拦截器 + 移除 @UseRoles（空操作）×2 + 添加 @Public() → **净效果：只追加 @Public()** |

**结论**：对于 GraphQL Resolver，**所有 5 种 Action（含 Delete）被设为 Public 时的最终效果完全一致**——在方法上追加 `@Public()` 装饰器，其余步骤均为空操作。不存在"Delete 拦截器残留导致 TypeError"的问题，因为 Resolver 里本来就没有这些拦截器。

#### Controller（REST）上的实际行为

对于 REST Controller，如果运行时模板或其他生成链路为方法注入了 `@UseInterceptors` 和 `@UseRoles`（with-auth-jwt 测试快照确认 Controller 上确实存在这些装饰器），则执行效果如下：

| Action | 代码分支 | 实际效果 |
|--------|---------|---------|
| Search / View | Search/View → 移除 FilterInterceptor | 移除 `AclFilterResponseInterceptor` + 移除 `@UseRoles` + 添加 `@Public()` |
| Create / Update | Create/Update → 移除 ValidateInterceptor | 移除 `AclValidateRequestInterceptor` + 移除 `@UseRoles` + 添加 `@Public()` |
| **Delete** | **两个 if 都不匹配** → **不移除任何拦截器** + 移除 `@UseRoles` + 添加 `@Public()` | **关键边界：Delete Action 在 Controller 上仍保留两个 ACL 拦截器**，但 `@UseRoles` 已被移除 |

**Controller 上 Delete 的潜在边界问题**：若 Delete 被设为 Public，运行时拦截器（`AclFilterResponseInterceptor` 和 `AclValidateRequestInterceptor`）仍会执行。由于 `@UseRoles` 已被移除，拦截器内部通过 `Reflector.get("roles", handler)` 读取到的 `permissionsRoles` 可能为 `undefined`，访问 `permissionsRoles.role` 会抛出 `TypeError: Cannot read properties of undefined`。这是 Controller 端的一个潜在 bug。

#### 三条关键边界总结

**边界一：Resolver 的 Public = 完全开放（类型层保障范围内）**

GraphQL Resolver 的 6 个 CRUD 方法和关联方法被设为 Public 时，只追加 `@Public()`。由于模板里本就没有 ACL 拦截器和 `@UseRoles`，Public 之后的方法完全不受 grants.json 的字段级过滤约束——即 Public = 返回 Entity DTO 中定义的全部字段。

**边界二：Public 读操作绕过字段级过滤（Controller 端）**

Controller 的 Search/View 被设为 Public 时，`AclFilterResponseInterceptor` 被显式移除，即使 grants.json 中对某些角色有字段限制，公开请求也会直接返回完整 Entity DTO。

**边界三：密码字段的双重保险**

密码字段已经在 Entity DTO（输出类型）层面被 `isPasswordField()` 完全排除（[create-entity-dto.ts L14](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-entity-dto.ts#L14)），即使 Public 权限开放读取，密码字段也不会出现在 GraphQL Schema 和 REST 响应中，是"类型层 + ACL 层"双重保障。

### 8.3 Granular 字段级权限的精确算法

在 `createGrants()` 中处理 Granular 权限时，字段级控制通过 `roleToFields` 和负向 glob 匹配实现（[create-grants.ts L55-L121](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts#L55-L121)）：

#### 步骤一：收集每个角色有权访问的字段

```
permissionFields.forEach(field →
  field.permissionRoles.forEach(role →
    roleToFields[role.resourceRole.name].add(field.field.name)
  )
)
```

`EntityPermissionField` 数据结构（[models.ts L741-L749](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts#L741-L749)）：
```
EntityPermissionField
├── field: EntityField              // 指向具体字段
├── permissionRoles: EntityPermissionRole[]  // 哪些角色可以访问该字段
└── fieldPermanentId / permissionId // 关联键
```

#### 步骤二：对每个角色做差集

对于当前 Entity 的 `allEntityFields`，不在 `roleToFields[role]` 集合中的字段即为 `forbiddenFields`。

#### 步骤三：生成负向 attributes

```ts
createAttributes(["*", ...forbiddenFields.map(f => `!${f}`)])
```

最终写入 grants.json 的效果：

```json
{
  "admin": {
    "User": {
      "read:any": ["*"],
      "update:any": ["*", "!password", "!secretField"]
    }
  },
  "viewer": {
    "User": {
      "read:any": ["*", "!password", "!email"],
      "update:any": []
    }
  }
}
```

**边界细节**：
- 如果某个角色的 `forbiddenFields` 包含了 Entity 的全部字段 → attributes 为 `["*", "!field1", "!field2", ...]`，实际效果等于禁止访问
- 密码字段等敏感字段即使未在 Granular 权限中显式排除，也不会出现在输出 DTO 中（类型层保障）
- 关联字段（Lookup）的粒度权限仅作用于"外键字段本身"，不级联影响关联 Entity 的内部字段

---

## 九、关键代码索引

### 9.1 核心生成流程

| 关注点 | 核心文件 |
|--------|---------|
| DSG 主入口 | [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/create-data-service.ts) |
| 上下文准备（解析实体、组装 Actions） | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/prepare-context.ts) |
| 服务端代码生成总入口 | [create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-server.ts) |
| 实体模块生成（Service/Controller/Resolver） | [create-resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-resource.ts) |
| 核心数据结构（Entity/Field/Module/Permission 等） | [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts)、[code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/code-gen-types.ts)、[models.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/models.ts) |

### 9.2 GraphQL 类型 / DTO 生成

| 关注点 | 核心文件 |
|--------|---------|
| DTO 总生成入口 | [create-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/create-dtos.ts) |
| 字段属性 → TS/GraphQL 类型/装饰器 | [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts) |
| GraphQL `@Field()` 装饰器生成 | [create-graphql-field-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/graphql-field-decorator/create-graphql-field-decorator.ts) |
| Entity ObjectType DTO | [create-entity-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/create-entity-dto.ts) |
| 自定义 DTO（Custom / CustomEnum） | [create-custom-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/dto/custom-types/create-custom-dtos.ts) |

### 9.3 Resolver 生成

| 关注点 | 核心文件 |
|--------|---------|
| Entity Resolver 生成 | [create-resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts) |
| Entity Resolver 基类模板 | [resolver.base.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/resolver.base.template.ts) |
| Entity Resolver 自定义动作 | [create-resolver-custom-actions.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver-custom-actions.ts) |
| GraphQL Query/Mutation 装饰器生成 | [create-graphql-operation-decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/resource/resolver/create-graphql-operation-decorator.ts) |
| 自定义模块 Resolver 生成 | [custom-resolver/create-resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/create-resolver.ts) |
| 自定义模块 Resolver 模板 | [custom-resolver/resolver.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/resolver.template.ts) |
| 自定义模块 Resolver 动作方法 | [custom-resolver/create-resolver-custom-actions.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/create-resolver-custom-actions.ts) |

### 9.4 访问控制（ACL）

| 关注点 | 核心文件 |
|--------|---------|
| grants.json 生成 | [create-grants.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/create-grants.ts) |
| 端点权限装饰器注入（Public / UseRoles） | [set-endpoint-permission.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/utils/set-endpoint-permission.ts) |
| 响应字段级过滤拦截器（运行时） | [aclFilterResponse.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/interceptors/aclFilterResponse.interceptor.ts) |
| 请求字段级校验拦截器（运行时） | [aclValidateRequest.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/interceptors/aclValidateRequest.interceptor.ts) |
| ABAC 字段级权限比对工具（运行时） | [abac.util.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator-catalog/src/auth/abac.util.ts) |

### 9.5 插件扩展点

| 关注点 | 核心文件 |
|--------|---------|
| 插件执行包装器（Before/After 调度） | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/plugin-wrapper.ts) |
| 插件加载与注册 | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/register-plugin.ts) |
| EventNames 枚举（全部 47 个事件） | [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugins.types.ts) |
| Events 类型映射（事件 → 参数类型） | [plugin-events.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugin-events.types.ts) |
| 各事件详细参数类型 | [plugin-events-params.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/libs/util/code-gen-types/src/plugin-events-params.types.ts) |

### 9.6 自定义模块

| 关注点 | 核心文件 |
|--------|---------|
| 自定义模块总生成入口 | [create-custom-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/create-custom-module.ts) |
| 自定义 Service 生成 | [custom-service/create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-service/create-service.ts) |
| 自定义 Controller 生成 | [custom-controller/create-controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-controller/create-controller.ts) |
| 自定义 NestJS Module 生成 | [custom-module/create-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/44-amplication/packages/data-service-generator/src/server/custom-module/custom-module/create-module.ts) |
