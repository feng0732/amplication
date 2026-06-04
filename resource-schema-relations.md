# 资源字段关系梳理

## 一、核心数据模型结构

### 1.1 三层模型架构

```
Resource (资源)
  └── Entity (实体)
        ├── EntityVersion (实体版本)
        │     ├── EntityField (实体字段)
        │     └── EntityPermission (实体权限)
        └── Module (模块)
              ├── ModuleAction (模块操作)
              └── ModuleDto (模块数据传输对象)
```

### 1.2 核心模型定义

#### Resource 模型
[Resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/models/Resource.ts)

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | String | PK, 非空 | 资源唯一标识 |
| name | String | 非空 | 资源名称 |
| resourceType | EnumResourceType | 非空 | 资源类型（Service/MessageBroker/Component 等） |
| blueprintId | String | 可选 | 关联蓝图 ID |
| projectId | String | 可选 | 所属项目 ID |
| entities | Entity[] | 关联 | 包含的实体列表 |

#### Entity 模型
[Entity.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/models/Entity.ts)

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | String | PK, 非空 | 实体唯一标识 |
| name | String | 非空 | 实体名称（代码用） |
| displayName | String | 非空 | 显示名称 |
| resourceId | String | FK, 非空 | 所属资源 ID |
| fields | EntityField[] | 关联 | 字段列表 |
| versions | EntityVersion[] | 关联 | 版本历史 |

#### EntityField 模型
[EntityField.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/models/EntityField.ts)

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | String | PK, 非空 | 字段唯一标识 |
| permanentId | String | 非空 | 永久 ID（跨版本保持） |
| name | String | 非空 | 字段名称（代码用） |
| dataType | EnumDataType | 非空 | 数据类型枚举 |
| properties | JsonValue | 可选 | 类型特定属性 |
| required | Boolean | 非空, 默认 false | 是否必填 |
| unique | Boolean | 非空, 默认 false | 是否唯一约束 |
| searchable | Boolean | 非空, 默认 false | 是否可搜索 |

---

## 二、字段类型与 Schema 约束

### 2.1 数据类型枚举 (EnumDataType)
[models.ts#L991-L1011](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/models.ts#L991-L1011)

```typescript
enum EnumDataType {
  Id,                    // 主键字段
  SingleLineText,        // 单行文本
  MultiLineText,         // 多行文本
  Email,                 // 邮箱
  WholeNumber,           // 整数
  DecimalNumber,         // 小数
  DateTime,              // 日期时间
  Boolean,               // 布尔
  Json,                  // JSON
  Lookup,                // 关联查找
  OptionSet,             // 单选枚举
  MultiSelectOptionSet,  // 多选枚举
  GeographicLocation,    // 地理位置
  File,                  // 文件
  CreatedAt,             // 创建时间（系统）
  UpdatedAt,             // 更新时间（系统）
  Username,              // 用户名
  Password,              // 密码
  Roles,                 // 角色集合
}
```

### 2.2 不可编辑字段类型
[field.ts#L8-L12](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/field/field.ts#L8-L12)

```typescript
const UNEDITABLE_FIELD_TYPES = new Set<EnumDataType>([
  EnumDataType.Id,
  EnumDataType.CreatedAt,
  EnumDataType.UpdatedAt,
]);
```

### 2.3 各类型 Schema 约束定义

所有字段类型的属性约束定义在 [libs/util/code-gen-types/src/schemas/](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/) 目录下。

#### 主键类型约束 (id.json)
[id.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/id.json)

```json
{
  "required": ["idType"],
  "properties": {
    "idType": {
      "type": "string",
      "enum": ["CUID", "UUID", "AUTO_INCREMENT", "AUTO_INCREMENT_BIG_INT"],
      "default": "CUID"
    }
  }
}
```

#### 关联字段约束 (lookup.json)
[lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/lookup.json)

```json
{
  "required": ["relatedEntityId", "relatedFieldId", "allowMultipleSelection"],
  "properties": {
    "relatedEntityId": { "type": "string" },        // 关联实体 ID
    "relatedFieldId": { "type": "string" },         // 反向关联字段 ID
    "allowMultipleSelection": { "type": "boolean" }, // 是否允许多选（一对多）
    "fkHolder": { "type": ["string", "null"] },     // 外键持有方 permanentId
    "fkFieldName": { "type": "string" }             // 外键字段名称
  }
}
```

#### 数值字段约束
[constants.ts#L194-L204](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts#L194-L204)

```typescript
// WholeNumber 默认属性
{
  databaseFieldType: "INT",  // 或 "BIG_INT"
  minimumValue: -999999999,
  maximumValue: 999999999,
}

// DecimalNumber 默认属性
{
  databaseFieldType: "FLOAT", // 或 "DECIMAL"
  minimumValue: -999999999,
  maximumValue: 999999999,
  precision: 2,
}
```

#### 文本字段约束
[constants.ts#L187-L192](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts#L187-L192)

```typescript
{
  maxLength: 1000, // SingleLineText / MultiLineText
}
```

#### 默认属性映射
[constants.ts#L184-L230](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts#L184-L230)

`DATA_TYPE_TO_DEFAULT_PROPERTIES` 定义了每种数据类型的默认属性值。

---

## 三、关联关系定义

### 3.1 Lookup 关联字段类型定义

**代码生成时解析后的 Lookup 属性：**
[code-gen-types.ts#L98-L102](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L98-L102)

```typescript
type LookupResolvedProperties = Lookup & {
  relatedEntity: Entity;              // 关联实体对象
  relatedField: EntityField;          // 反向关联字段对象
  isOneToOneWithoutForeignKey?: boolean; // 一对一关系中不持有外键的一方
};
```

### 3.2 关联类型判断

[field.ts#L34-L73](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/field/field.ts#L34-L73)

```typescript
// 是否关联字段
function isRelationField(field: EntityField): boolean {
  return field.dataType === EnumDataType.Lookup;
}

// 是否一对一关联
function isOneToOneRelationField(field: EntityField): boolean {
  const properties = field.properties as types.Lookup;
  return isRelationField(field) && !properties.allowMultipleSelection;
}

// 是否一对多关联
function isToManyRelationField(field: EntityField): boolean {
  return isRelationField(field) && !isOneToOneRelationField(field);
}
```

### 3.3 资源间关联 (Resource Relation)

[Relation.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/relation/dto/Relation.ts)

```typescript
class Relation extends IBlock {
  relationKey: string;           // 关联类型标识
  relatedResources: string[];    // 关联的资源 ID 列表
}
```

**蓝图中的关联定义：**
[models.ts#L243-L252](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/models.ts#L243-L252)

```typescript
type BlueprintRelation = {
  key: string;                          // 关联键
  name: string;                         // 关联名称
  relatedTo: string;                    // 关联的蓝图类型
  allowMultiple: boolean;               // 是否允许多个
  required: boolean;                    // 是否必填
  parentShouldBuildWithChild: boolean;  // 子资源构建时是否级联构建父资源
  limitSelectionToProject: boolean;     // 是否限制为同一项目内
};
```

---

## 四、关联传播逻辑

### 4.1 Lookup 字段双向关联自动创建

#### 创建关联字段流程
[entity.service.ts#L2669-L2696](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2669-L2696)

```typescript
async createField(args: CreateOneEntityFieldArgs, user: User): Promise<EntityField> {
  if (args.data.dataType === EnumDataType.Lookup) {
    // 1. 为反向关联字段生成 permanentId
    args.data.properties.relatedFieldId = cuid();
    
    // 2. 在关联实体上创建反向 Lookup 字段
    await this.createRelatedField(
      properties.relatedFieldId,           // 反向字段 permanentId
      args.relatedFieldName,                // 反向字段名称
      args.relatedFieldDisplayName,         // 反向字段显示名
      args.relatedFieldAllowMultipleSelection, // 反向字段是否允许多选
      properties.relatedEntityId,           // 关联实体 ID
      entity.id,                            // 当前实体 ID
      permanentId ?? fieldId,               // 当前字段 permanentId
      user,
      properties.fkHolder                   // 外键持有方
    );
  }
  
  // 3. 创建当前字段
  const newField = await this.prisma.entityField.create({...});
  
  // 4. 为关联字段创建默认操作和 DTO
  if (args.data.dataType === EnumDataType.Lookup) {
    await this.moduleActionService.createDefaultActionsForRelationField(...);
    await this.moduleDtoService.createDefaultDtosForRelatedEntity(...);
  }
}
```

#### 反向关联字段创建
[entity.service.ts#L2780-L2847](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2780-L2847)

```typescript
private async createRelatedField(
  id: string,                          // 反向字段 permanentId
  name: string,                        // 反向字段名称
  displayName: string,                 // 反向字段显示名
  relatedFieldAllowMultipleSelection: boolean, // 反向是否多选
  entityId: string,                    // 要创建字段的实体 ID
  relatedEntityId: string,             // 关联的实体 ID（即当前实体）
  relatedFieldId: string,              // 原字段 permanentId
  user: User,
  fkHolder?: string
): Promise<EntityField> {
  const newField = await this.prisma.entityField.create({
    data: {
      name,
      displayName,
      dataType: EnumDataType.Lookup,
      permanentId: id,
      properties: {
        allowMultipleSelection: relatedFieldAllowMultipleSelection,
        relatedEntityId,                // 指向原实体
        relatedFieldId,                 // 指向原字段
        fkHolder,
      },
      // ...
    },
  });
  
  // 同时为反向字段创建默认操作和 DTO
  await this.moduleActionService.createDefaultActionsForRelationField(...);
  await this.moduleDtoService.createDefaultDtosForRelatedEntity(...);
  
  return newField;
}
```

### 4.2 关联字段更新传播

#### 更新时的关联处理逻辑
[entity.service.ts#L2933-L3014](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2933-L3014)

```typescript
async updateField(args: UpdateOneEntityFieldArgs, user: User): Promise<EntityField> {
  // 判断需要执行的关联操作
  const shouldDeleteRelated = // 类型从 Lookup 改为其他类型
    field.dataType === EnumDataType.Lookup && args.data.dataType !== EnumDataType.Lookup;
  
  const shouldCreateRelated = // 类型从其他改为 Lookup
    args.data.dataType === EnumDataType.Lookup && field.dataType !== EnumDataType.Lookup;
  
  const shouldChangeRelated = // 关联实体 ID 变化
    args.data.properties?.relatedEntityId !== oldProperties.relatedEntityId;
  
  // 执行操作
  if (shouldDeleteRelated || shouldChangeRelated) {
    await this.deleteRelatedField(oldProperties.relatedFieldId, ...);
  }
  
  if (shouldCreateRelated || shouldChangeRelated) {
    await this.createRelatedField(newProperties.relatedFieldId, ...);
  }
  
  // 关联属性变化时更新操作
  if (shouldUpdateRelatedFieldActions) {
    await this.moduleActionService.updateDefaultActionsForRelationField(...);
  }
}
```

### 4.3 关联字段级联删除
[entity.service.ts#L2849-L2898](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2849-L2898)

删除 Lookup 字段时，同时删除：
1. 反向关联字段
2. 关联字段的默认操作（ModuleAction）
3. 关联字段的默认 DTO（ModuleDto）

### 4.4 一对一关系外键持有方判断
[prepare-context.ts#L229-L241](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts#L229-L241)

```typescript
const isOneToOne = !fieldProperties.allowMultipleSelection && 
                   !relatedFieldProperties.allowMultipleSelection;

let isOneToOneWithoutForeignKey = true;

if (fieldProperties.fkHolder) {
  // 显式指定外键持有方
  isOneToOneWithoutForeignKey = isOneToOne && field.permanentId !== fieldProperties.fkHolder;
} else {
  // 默认规则：permanentId 较大的一方不持有外键
  isOneToOneWithoutForeignKey = isOneToOne && field.permanentId > relatedField.permanentId;
}
```

### 4.5 资源级联构建
[relation.service.ts#L219-L270](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/relation/relation.service.ts#L219-L270)

```typescript
async getCascadingBuildableResourceIds(resourceIds: string[]): Promise<string[]> {
  const visited = new Set<string>(resourceIds);
  let queue = [...resourceIds];
  
  while (queue.length > 0) {
    const currentBatch = queue;
    queue = [];
    
    // 获取需要级联构建的父资源（parentShouldBuildWithChild = true）
    const newParents = await this.getBuildableParents(currentBatch);
    
    for (const parent of newParents) {
      if (!visited.has(parent.parentResourceId)) {
        visited.add(parent.parentResourceId);
        queue.push(parent.parentResourceId);
      }
    }
  }
  
  return Array.from(visited);
}
```

---

## 五、生成侧影响

### 5.1 关联字段解析 (prepare-context)
[prepare-context.ts#L185-L261](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261)

在代码生成前，会将 Lookup 字段的 ID 引用解析为完整对象：

```typescript
function resolveLookupFields(entities: Entity[]): Entity[] {
  // 1. 建立索引
  const entityIdToEntity: Record<string, Entity> = {};
  const fieldIdToField: Record<string, EntityField> = {};
  
  // 2. 解析每个 Lookup 字段
  return entities.map((entity) => ({
    ...entity,
    fields: entity.fields.map((field) => {
      if (field.dataType === EnumDataType.Lookup) {
        const properties: LookupResolvedProperties = {
          ...field.properties,
          relatedEntity: entityIdToEntity[relatedEntityId],
          relatedField: fieldIdToField[relatedFieldId],
          isOneToOneWithoutForeignKey,
          fkFieldName: fkFieldName || `${field.name}Id`,
        };
        return { ...field, properties };
      }
      return field;
    }),
  }));
}
```

### 5.2 Prisma Schema 生成影响
[create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts)

#### 字段类型映射
| 字段类型 | Prisma 类型 | 特殊处理 |
|---------|------------|---------|
| Id (CUID/UUID) | String | @id @default(cuid()) / @default(uuid()) |
| Id (AUTO_INCREMENT) | Int / BigInt | @id @default(autoincrement()) |
| SingleLineText | String | 支持 @unique |
| WholeNumber | Int / BigInt | 根据 databaseFieldType |
| DecimalNumber | Float / Decimal | 根据 databaseFieldType |
| Lookup (持有外键) | 实体类型 + 外键字段 | 生成关联字段和外键字段 |
| Lookup (不持有外键) | 实体类型 | 仅生成关联字段 |
| OptionSet | Enum | 生成枚举定义 |
| MultiSelectOptionSet | Enum[] | 生成枚举数组 |

#### Lookup 字段生成逻辑
[create-prisma-schema-fields.ts#L280-L366](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L280-L366)

```typescript
case EnumDataType.Lookup: {
  const { relatedEntity, allowMultipleSelection, isOneToOneWithoutForeignKey } = properties;
  
  if (allowMultipleSelection || isOneToOneWithoutForeignKey) {
    // 一对多或一对一不持有外键方：仅生成对象字段
    return [createObjectField(name, relatedEntity.name, !isOneToOneWithoutForeignKey, allowMultipleSelection)];
  }
  
  // 一对一持有外键方或多对一：生成对象字段 + 外键字段
  return [
    // 对象字段（关联）
    createObjectField(name, relatedEntity.name, false, field.required, relationName, [scalarRelationFieldName], ["id"]),
    // 外键字段（scalar）
    createScalarField(scalarRelationFieldName, idTypeToPrismaScalarType[idType], false, field.required, ...),
  ];
}
```

### 5.3 DTO 生成影响
[create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts)

#### 不同 DTO 类型的字段处理

| DTO 类型 | Lookup 字段类型 | 生成类型 | 装饰器 |
|---------|----------------|---------|-------|
| Entity (对象类型) | 所有 | 关联实体类型 | @Field() @Type() |
| CreateInput | 一对一 | WhereUniqueInput | @ValidateNested() @Type() |
| CreateInput | 一对多 | CreateNestedManyWithoutInput | @ValidateNested() @Type() |
| UpdateInput | 一对一 | WhereUniqueInput | @ValidateNested() @Type() |
| UpdateInput | 一对多 | UpdateManyWithoutInput | @ValidateNested() @Type() |
| WhereInput | 一对一 | WhereUniqueInput | @Type() |
| WhereInput | 一对多 | EntityListRelationFilter | @Type() |
| WhereUniqueInput | 一对一 | WhereUniqueInput | @Type() |

#### 校验装饰器
[create-field-class-property.ts#L225-L330](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L225-L330)

- 数值字段：`@Min()` `@Max()`（根据 minimumValue/maximumValue）
- 文本字段：`@MaxLength()`（根据 maxLength）
- 枚举字段：`@IsEnum()`
- 关联字段：`@ValidateNested()` `@Type()`
- 可选字段：`@IsOptional()`

### 5.4 GraphQL API 生成影响

#### 默认操作生成
[entity-util.ts#L27-L97](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/lib/entity-util.ts#L27-L97)

每个实体生成以下 GraphQL 操作：
```
Query:
  - entityName(id): Entity          # 获取单个
  - entityNames(where, orderBy, skip, take): [Entity!]!  # 查询列表
  - entityNamesCount(where): Int!   # 计数
  - _entityNamesMeta: Meta          # 元数据

Mutation:
  - createEntityName(data): Entity  # 创建
  - updateEntityName(where, data): Entity  # 更新
  - deleteEntityName(where): Entity  # 删除
```

#### 关联字段操作生成
[entity-util.ts#L100-L168](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/lib/entity-util.ts#L100-L168)

**一对多关系 (isToMany = true)：**
```
# 关联操作（以 Order -> OrderItem 为例）
Query:
  - orderItems(id): [OrderItem!]!   # 查询订单的订单项

Mutation:
  - connectOrderItems(id, data): Order  # 关联多个订单项
  - disconnectOrderItems(id, data): Order  # 取消关联多个订单项
  - updateOrderItems(id, data): Order    # 更新多个订单项
```

**一对一关系 (isToMany = false)：**
```
# 关联操作（以 User -> Profile 为例）
Query:
  - profile(id): Profile   # 获取用户的个人资料
```

### 5.5 REST API 生成影响

#### 实体 CRUD 路由
[entity-util.ts#L27-L97](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/lib/entity-util.ts#L27-L97)

```
GET    /api/entity-names              # 列表
GET    /api/entity-names/:id          # 单个
POST   /api/entity-names              # 创建
PATCH  /api/entity-names/:id          # 更新
DELETE /api/entity-names/:id          # 删除
GET    /api/entity-names/:id/meta     # 元数据
```

#### 关联字段路由
**一对多：**
```
GET    /api/entity-names/:id/related-field     # 查询关联
POST   /api/entity-names/:id/related-field     # 关联多个
PATCH  /api/entity-names/:id/related-field     # 更新关联
DELETE /api/entity-names/:id/related-field     # 取消关联
```

**一对一：**
```
GET    /api/entity-names/:id/related-field     # 获取关联
```

### 5.6 外键字段命名规则
[prepare-context.ts#L248-L250](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts#L248-L250)

```typescript
fkFieldName: !isEmpty(trim(fkFieldName))
  ? fkFieldName              // 用户指定
  : `${field.name}Id`,       // 默认：字段名 + Id
```

### 5.7 关联名称生成规则
[create-prisma-schema-fields.ts#L538-L582](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L538-L582)

当两个实体间存在多个关联时，需要生成关联名称以区分：

1. 优先匹配实体名与字段名的关系
2. 字段名唯一时使用字段名
3. 都不唯一时组合实体名和字段名

---

## 六、字段校验流程

### 6.1 创建/更新字段时的校验
[entity.service.ts#L2667](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2667)

```typescript
this.validateFieldMutationArgs(args, entity);   // 参数校验
await this.validateFieldData(data, entity, enforceValidation);  // 数据校验
```

### 6.2 命名规范校验
[entity.service.ts#L147-L149](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L147-L149)

```typescript
const NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/;
```

### 6.3 JSON Schema 校验
使用 JSON Schema 校验各数据类型的 `properties` 字段：
- 从 `@amplication/code-gen-types` 获取对应类型的 schema
- 通过 `JsonSchemaValidationService` 执行校验
- 必填字段、类型、枚举值等约束

---

## 七、关键代码参考

| 模块 | 文件路径 | 主要职责 |
|------|---------|---------|
| 实体服务 | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | 实体/字段 CRUD、关联处理 |
| 关联服务 | [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/relation/relation.service.ts) | 资源关联、级联构建 |
| 字段工具 | [field.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/field/field.ts) | 字段类型判断 |
| 实体工具 | [entity-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/lib/entity-util.ts) | 默认操作生成 |
| Prisma 生成 | [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) | Prisma 字段生成 |
| DTO 生成 | [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts) | DTO 属性生成 |
| 上下文准备 | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts) | 关联字段解析 |
| 常量定义 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts) | 默认属性、系统字段 |
