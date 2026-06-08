# Amplication Schema Relation 深度解析

## 一、关系类型与 Lookup 字段

Amplication 通过 `Lookup` 数据类型（`EnumDataType.Lookup`）统一表达实体间的所有关系类型。关系的具体类型由两端字段的 `allowMultipleSelection` 属性组合决定。

### 1.1 Lookup 字段核心属性

定义于 [lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/code-gen-types/src/schemas/lookup.json)：

| 属性 | 类型 | 说明 |
|------|------|------|
| `relatedEntityId` | string | 关联目标实体的 ID |
| `relatedFieldId` | string | 反向关联字段的 ID（双向关联时自动填充） |
| `allowMultipleSelection` | boolean | 是否允许选择多个关联实体 |
| `fkHolder` | string | **一对一专用**：指定持有外键的字段 `permanentId` |
| `fkFieldName` | string | 外键字段名，默认 `${fieldName}Id` |

### 1.2 关系类型判定矩阵

关系类型由两端 `allowMultipleSelection` 的布尔组合决定：

| 本端 allowMultipleSelection | 对端 allowMultipleSelection | 关系类型 |
|---------------------------|---------------------------|---------|
| false | false | **One-to-One**（严格一对一） |
| false | true | **Many-to-One**（多对一，本端为"多"侧） |
| true | false | **One-to-Many**（一对多，本端为"一"侧） |
| true | true | **Many-to-Many**（多对多） |

---

## 二、to-One 判定的代码实现与命名歧义

### ⚠️ 重要差异：`isOneToOneRelationField` 名不副实

**之前可能的误解**：以为 `isOneToOneRelationField` 会同时检查两端的 `allowMultipleSelection` 来判定严格的 One-to-One。

**代码实际情况**（[field.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/field/field.ts#L34-L42)）：

```typescript
export function isOneToOneRelationField(
  field: EntityField
): field is EntityLookupField {
  if (!isRelationField(field)) {
    return false;
  }
  const properties = field.properties as types.Lookup;
  return !properties.allowMultipleSelection;  // ← 只检查本端！不检查对端！
}

export function isToManyRelationField(
  field: EntityField
): field is EntityLookupField {
  return isRelationField(field) && !isOneToOneRelationField(field);
}
```

### 2.1 正确的语义理解

| 函数名 | 实际语义 | 包含的关系类型 |
|--------|---------|-------------|
| `isOneToOneRelationField(field)` | **to-One 判定**（本端是单值） | 严格 One-to-One + Many-to-One（"多"侧） |
| `isToManyRelationField(field)` | **to-Many 判定**（本端是集合） | One-to-Many（"一"侧） + Many-to-Many |

### 2.2 严格 One-to-One 判定仅用于外键策略

真正需要同时检查两端的严格 One-to-One 判定，只在 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts#L229-L241) 的外键持有方判定中内联实现：

```typescript
const isOneToOne =
  !fieldProperties.allowMultipleSelection &&
  !relatedFieldProperties.allowMultipleSelection;  // ← 同时检查两端
```

这个严格判定并未抽成独立的工具函数。

---

## 三、外键策略（Foreign Key Strategy）

### 3.1 外键持有方判定

在一对一关系中，只能有一侧持有外键列。Amplication 通过以下优先级判定哪一侧不生成外键（`isOneToOneWithoutForeignKey = true`）。

核心逻辑位于 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts#L229-L250)：

```typescript
const isOneToOne =
  !fieldProperties.allowMultipleSelection &&
  !relatedFieldProperties.allowMultipleSelection;

let isOneToOneWithoutForeignKey = true;

// 优先级1：用户显式指定 fkHolder
if (fieldProperties.fkHolder) {
  isOneToOneWithoutForeignKey =
    isOneToOne && field.permanentId !== fieldProperties.fkHolder;
} 
// 优先级2：比较 permanentId，较大者不持有外键
else {
  isOneToOneWithoutForeignKey =
    isOneToOne && field.permanentId > relatedField.permanentId;
}
```

**判定规则总结：**

| 条件 | 当前字段是否持有外键 |
|------|------------------|
| 非严格 One-to-One 关系 | to-One 侧（Many-to-One 的"多"端）总是持有外键 |
| 严格 One-to-One + 指定了 fkHolder = 当前字段 ID | 是 |
| 严格 One-to-One + 指定了 fkHolder = 对方字段 ID | 否 |
| 严格 One-to-One + 未指定 fkHolder + 当前 permanentId < 对方 | 是 |
| 严格 One-to-One + 未指定 fkHolder + 当前 permanentId > 对方 | 否 |

### 3.2 外键字段名

外键列名由 `fkFieldName` 属性控制，默认规则：
- 若用户未显式指定：自动生成 `${field.name}Id`
- 若用户显式指定：使用用户指定的名称

---

## 四、Prisma Schema 生成规则

Lookup 字段转换为 Prisma Schema 的核心逻辑位于 [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L280-L366)。

### 4.1 三种生成场景

#### 场景 A：to-Many 或 无外键的严格 One-to-One

仅生成 Prisma 对象引用字段，不生成外键列：

```prisma
fieldName RelatedEntity[]  // to-Many
// 或
fieldName RelatedEntity?   // 无外键的 One-to-One
```

条件：
- `isToManyRelationField(field)`（to-Many），或
- `isOneToOneWithoutForeignKey === true`（严格一对一的无外键侧）

#### 场景 B：持有外键的严格 One-to-One

生成对象字段 + 标量外键字段 + `@unique` 约束：

```prisma
fieldName   RelatedEntity @relation("RelationName", fields: [fieldNameId], references: [id])
fieldNameId String        @unique // 一对一外键必须唯一
```

#### 场景 C：Many-to-One（to-One 的"多"侧，持有外键）

生成对象字段 + 标量外键字段（无 `@unique`，因为多对一允许多条记录指向同一实体）：

```prisma
fieldName   RelatedEntity @relation("RelationName", fields: [fieldNameId], references: [id])
fieldNameId String
```

### 4.2 Prisma 关系命名

Prisma 的 `@relation("RelationName")` 名称由两端字段的唯一标识组合生成，确保双向关系正确配对。

---

## 五、嵌套 DTO 操作的实际生成逻辑

### ⚠️ 重要差异：枚举定义 ≠ 实际生成

**之前可能的误解**：以为 `Create`、`Connect`、`ConnectOrCreate`、`Disconnect`、`Set` 五种操作都会生成。

**代码实际情况**：枚举中虽然定义了 5 种，但**实际只生成了 3 种**，且 to-One 和 to-Many 的处理方式完全不同。

### 5.1 枚举定义（仅为语义声明）

位于 [create-nested-input-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/nested-input-dto/create-nested-input-dto.ts#L20-L26)：

```typescript
export enum NestedMutationOptions {
  "Create" = "create",
  "Connect" = "connect",
  "ConnectOrCreate" = "connectOrCreate",
  "Disconnect" = "disconnect",
  "Set" = "set",
}
```

### 5.2 to-Many 嵌套 DTO 的实际生成

核心逻辑位于 [create-nested-input-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/nested-input-dto/create-nested-input-dto.ts#L44-L97)：

```typescript
function createNestedManyProperties(...): namedTypes.ClassProperty[] {
  // ...
  const mutationOptionsObjectProperties: namedTypes.ClassProperty[] = [
    createProperty, // ← 始终包含 Connect
  ];
  // ↓ 只有 Update 场景的 toMany 才额外添加 Disconnect 和 Set
  if (dtoType === EntityDtoTypeEnum.RelationUpdateManyWithoutSourceInput) {
    mutationOptionsObjectProperties.push(disconnectProperty);
    mutationOptionsObjectProperties.push(setProperty);
  }
  return mutationOptionsObjectProperties;
}
```

**to-Many 实际生成矩阵：**

| DTO 类型 | 生成的操作 | 对应类名 |
|---------|----------|---------|
| `RelationCreateNestedManyWithoutSourceInput` | **仅 Connect** | `XxxCreateNestedManyWithoutYyyInput` |
| `RelationUpdateManyWithoutSourceInput` | Connect + Disconnect + Set | `XxxUpdateManyWithoutYyyInput` |

> ❌ **Create 和 ConnectOrCreate 在当前代码中从未实际生成！**

### 5.3 to-One 关系的 DTO 处理

**⚠️ 关键发现：to-One 关系不生成独立的嵌套 DTO 类！**

位于 [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L479-L485)：

```typescript
if (isQuery || isEntityInputExceptRelationInput(dtoType) || isNestedInput) {
  // to-One 的 CreateInput / UpdateInput 直接使用 WhereUniqueInput
  return [builders.tsTypeReference(createWhereUniqueInputID(prismaField.type))];
}
```

**to-One 的 DTO 行为：**

| DTO 类型 | 生成的类型 | 实际效果 |
|---------|----------|---------|
| `CreateInput` | `XxxWhereUniqueInput` | 只有 `{ id: string }`，即仅支持 Connect 通过 ID |
| `UpdateInput` | `XxxWhereUniqueInput` | 同上，仅支持 Connect 通过 ID |
| 实体 ObjectType | `Xxx` | 完整实体对象类型 |

> to-One 关系不支持 Disconnect、Create 等嵌套操作，只能通过 ID 关联或直接传 null。

### 5.4 to-Many vs to-One DTO 差异汇总

| 维度 | to-Many（isToManyRelationField） | to-One（isOneToOneRelationField） |
|-----|--------------------------------|--------------------------------|
| 是否生成独立嵌套 DTO 类 | ✅ 是 | ❌ 否，直接用 WhereUniqueInput |
| Create 场景 | `CreateNestedManyWithout` 类，仅含 `connect` | `WhereUniqueInput`，仅含 `id` |
| Update 场景 | `UpdateManyWithout` 类，含 `connect`/`disconnect`/`set` | `WhereUniqueInput`，仅含 `id` |
| 支持的嵌套操作 | Connect、Disconnect、Set | Connect（通过 ID）或 null |

---

## 六、字段删除策略的完整实现

删除策略枚举定义于 [EnumRelatedFieldStrategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/dto/EnumRelatedFieldStrategy.ts)：

```typescript
export enum EnumRelatedFieldStrategy {
  Delete = "Delete",
  UpdateToScalar = "UpdateToScalar",
}
```

### 6.1 公共清理函数 `deleteRelatedField()`

两种策略最终都会调用的对端清理函数，位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2849-L2898)：

```typescript
private async deleteRelatedField(
  permanentId: string, entityId: string, user: User
): Promise<void> {
  await this.useLocking(entityId, user, async (entity) => {
    const field = await this.getField({ where: { permanentId } });

    // 1. 删除对端 EntityField 记录
    const deletedField = await this.prisma.entityField.delete({
      where: { entityVersionId_permanentId: { permanentId, entityVersionId: field.entityVersionId } },
      include: { entityVersion: { include: { entity: true } } },
    });

    const moduleId = await this.moduleService.getDefaultModuleIdForEntity(entity.resourceId, entity.id);

    // 2. 删除对端关联的 ModuleAction（API 端点权限）
    await this.moduleActionService.deleteDefaultActionsForRelationField(
      deletedField, moduleId, user
    );

    // 3. 删除对端关联的 ModuleDto（DTO 配置）
    await this.moduleDtoService.deleteDefaultDtosForRelatedEntity(
      deletedField, deletedField.entityVersion.entity, moduleId, user
    );
  });
}
```

**`deleteRelatedField()` 执行的三项清理：**

| 步骤 | 操作 | 说明 |
|-----|------|------|
| 1 | `prisma.entityField.delete()` | 物理删除对端字段记录 |
| 2 | `moduleActionService.deleteDefaultActionsForRelationField()` | 删除对端该关系字段对应的所有 API 端点权限配置 |
| 3 | `moduleDtoService.deleteDefaultDtosForRelatedEntity()` | 删除对端该关系字段对应的嵌套 DTO 配置 |

---

### 6.2 策略一：Delete（级联删除关联字段 + 物理删除本端）

核心逻辑位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3149-L3256)。

**Delete 策略完整流程：**

```
deleteField(strategy=Delete)
        │
        ▼
  1. 对端清理（deleteRelatedField）
     ├─ 删除对端 EntityField
     ├─ 删除对端 ModuleAction
     └─ 删除对端 ModuleDto
        │
        ▼
  2. 本端物理删除（prisma.entityField.delete）
        │
        ▼
  3. 本端清理（仅当 dataType 是 Lookup/OptionSet 时）
     ├─ Lookup → deleteDefaultActionsForRelationField
     ├─ Lookup → deleteDefaultDtosForRelatedEntity
     └─ OptionSet → deleteDefaultDtoForEnumField
        │
        ▼
  返回已删除的字段对象
```

```typescript
// [entity.service.ts L3149-L3166]
if (field.dataType === EnumDataType.Lookup) {
  const properties = field.properties as unknown as types.Lookup;
  if (fieldStrategy === EnumRelatedFieldStrategy.Delete) {
    try {
      await this.deleteRelatedField(
        properties.relatedFieldId,   // 对端字段 ID
        properties.relatedEntityId,  // 对端实体 ID
        user
      );
    } catch (error) {
      // ⚠️ 容错：对端字段删除失败不阻塞本端删除
      this.logger.error("Continue with FieldDelete even though...", error);
    }
  }
}
// ...
// 本端物理删除 + 本端 ModuleAction/ModuleDto 清理
```

---

### 6.3 策略二：UpdateToScalar（转换为标量字段）

#### ⚠️ 关键发现：UpdateToScalar 也会触发对端清理！

**之前的误解**：以为 UpdateToScalar 只是把本端字段改成标量，不会影响对端。

**代码实际情况**：虽然 `deleteField` 的 UpdateToScalar 分支在调用 `updateField()` 后直接 `return`（跳过了 Delete 策略中本端物理删除和本端清理的代码），但 **`updateField()` 内部检测到 dataType 从 Lookup 变为非 Lookup 时，会自动触发 `shouldDeleteRelated` 逻辑，级联删除对端字段及其 ModuleAction、ModuleDto**。

#### 6.3.1 触发点：`updateField()` 中的 `shouldDeleteRelated`

位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2933-L2996)：

```typescript
// [entity.service.ts L2933-L2936]
// Delete related field in case field data type is changed from lookup
const shouldDeleteRelated =
  field.dataType === EnumDataType.Lookup &&          // 原始字段是 Lookup
  args.data.dataType !== EnumDataType.Lookup;        // 新 dataType 不是 Lookup → ✅ TRUE!

// ...

// [entity.service.ts L2987-L2996]
// In case related field should be deleted or changed, delete the existing related field
if (shouldDeleteRelated || shouldChangeRelated) {
  const properties = field.properties as unknown as types.Lookup;
  await this.deleteRelatedField(          // ← 自动调用对端清理！
    properties.relatedFieldId,
    properties.relatedEntityId,
    user
  );
}
```

因为 UpdateToScalar 传入的 `data.dataType` 是 `Json` 或 String（非 Lookup），所以 `shouldDeleteRelated` 恒为 `true`，进而自动触发对端的完整清理。

#### 6.3.2 UpdateToScalar 的字段转换规则

位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3167-L3195)：

| 原字段类型 | 转换后 dataType | 字段名变化 | displayName 变化 |
|----------|----------------|----------|----------------|
| to-Many Lookup | `Json` | 不变 | 不变 |
| to-One Lookup | 关联实体 ID 的标量类型（通常 String） | `${name}Id` | `${displayName} ID` |

```typescript
const data: EntityFieldUpdateInput = {
  dataType: field.dataType,
  name: allowMultipleSelection ? field.name : `${field.name}Id`,
  displayName: allowMultipleSelection ? field.displayName : `${field.displayName} ID`,
  properties: DATA_TYPE_TO_DEFAULT_PROPERTIES[field.dataType],
};
await this.updateField({ data, where: { id: args.where.id } }, user);
return;  // ← 跳过本端物理删除和本端 ModuleAction/ModuleDto 清理
```

#### 6.3.3 UpdateToScalar 完整流程图

```
deleteField(strategy=UpdateToScalar)
        │
        ▼
  构造 updateField 参数（dataType: Lookup → Json/String）
        │
        ▼
  调用 updateField(data, where)
        │
        ├─────────────────────────────────────────────┐
        │                                             │
        ▼                                             ▼
  updateField 入口                              shouldDeleteRelated = TRUE
        │                                    (Lookup → 非Lookup)
        │                                             │
        │                                             ▼
        │                                    deleteRelatedField() 对端清理
        │                                     ├─ 删除对端 EntityField
        │                                     ├─ 删除对端 ModuleAction
        │                                     └─ 删除对端 ModuleDto
        │                                             │
        ▼                                             ▼
  prisma.entityField.update()           对端字段、权限、DTO 均已清理
  ├─ dataType: Lookup → Json/String
  ├─ name: (to-One 加 Id 后缀)
  └─ properties: 默认标量属性
        │
        ▼
  return updatedField ──────── 回到 deleteField
        │
        ▼
  deleteField 直接 return;
  ⚠️ 跳过本端物理删除
  ⚠️ 跳过本端 ModuleAction 清理
  ⚠️ 跳过本端 ModuleDto 清理
```

#### 6.3.4 两种删除策略行为对比

| 行为 | Delete 策略 | UpdateToScalar 策略 |
|-----|------------|-------------------|
| 对端 EntityField | ✅ 删除 | ✅ 删除（通过 updateField 内部触发） |
| 对端 ModuleAction | ✅ 删除 | ✅ 删除（同上） |
| 对端 ModuleDto | ✅ 删除 | ✅ 删除（同上） |
| 本端 EntityField | ✅ 物理删除 | ⚠️ 转换为标量字段（保留记录） |
| 本端 ModuleAction | ✅ 删除 | ❌ **未清理**（return 跳过） |
| 本端 ModuleDto | ✅ 删除 | ❌ **未清理**（return 跳过） |
| 数据库列 | 外键列删除 | to-Many → Json 列；to-One → 标量列保留 |

> **潜在不一致点**：UpdateToScalar 策略下，本端的 ModuleAction 和 ModuleDto 没有被清理，仍残留在数据库中。这是因为 `deleteField` 的 UpdateToScalar 分支在调用 `updateField()` 后直接 `return`，跳过了 Delete 策略中 L3228-L3241 的本端清理代码。

---

## 七、实体删除：软删除机制

### ⚠️ 重要发现：实体删除不是物理删除，而是软删除

核心实现位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L941-L1024)：

```typescript
async deleteOneEntity(
  args: DeleteOneEntityArgs,
  user: User,
  fieldStrategy = EnumRelatedFieldStrategy.Delete
): Promise<Entity | null> {
  return await this.useLocking(args.where.id, user, async (entity) => {
    // 步骤1：找出所有引用该实体的 Lookup 字段并级联处理
    const relatedEntityFields = await this.prisma.entityField.findMany({
      where: {
        dataType: EnumDataType.Lookup,
        properties: { path: ["relatedEntityId"], equals: args.where.id },
        entityVersion: { versionNumber: CURRENT_VERSION_NUMBER },
      },
    });
    // ... 鉴权检查（auth 实体不可删除）...
    for (const relatedEntityField of relatedEntityFields) {
      await this.deleteField(
        { where: { id: relatedEntityField.id } },
        user,
        fieldStrategy  // 对每个引用字段应用相同的删除策略
      );
    }

    // 步骤2：删除该实体的默认 Module（含权限、DTO 配置）
    await this.moduleService.deleteDefaultModuleForEntity(...);

    // ⬇️ 步骤3：软删除（update，不是 delete！）
    return this.prisma.entity.update({
      where: args.where,
      data: {
        // 名称加前缀避免名称冲突，方便恢复
        name: prepareDeletedItemName(entity.name, entity.id),
        displayName: prepareDeletedItemName(entity.displayName, entity.id),
        pluralDisplayName: prepareDeletedItemName(entity.pluralDisplayName, entity.id),
        deletedAt: new Date(),  // ← 软删除标记
        versions: {
          update: {
            where: { entityId_versionNumber: { ... } },
            data: { deleted: true },  // ← 当前版本也标记删除
          },
        },
      },
    });
  });
}
```

### 7.1 软删除工具函数

位于 [softDelete.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/util/softDelete.ts)：

```typescript
export function prepareDeletedItemName(currentValue: string, id: string): string {
  return `__${id}_${currentValue}`;  // 例：__ent_abc123_User
}

export function revertDeletedItemName(currentValue: string, id: string): string {
  return currentValue.replace(`__${id}_`, "");
}
```

### 7.2 软删除的完整流程

```
用户触发 deleteOneEntity
        │
        ▼
  1. 查找所有 relatedEntityId = 本实体 ID 的 Lookup 字段
        │
        ▼
  2. 对每个引用字段执行 deleteField()（使用相同的 fieldStrategy）
        │
        ▼
  3. 删除该实体的默认 Module（ModuleAction + ModuleDto）
        │
        ▼
  4. prisma.entity.update() —— 软删除
     ├─ name → __${id}_${name}  （避免名称唯一约束冲突）
     ├─ displayName → __${id}_${displayName}
     ├─ pluralDisplayName → __${id}_${pluralDisplayName}
     ├─ deletedAt → new Date()   （软删除标记）
     └─ versions[CURRENT].deleted → true
```

### 7.3 查询时的软删除过滤

所有实体查询都会自动带上 `deletedAt: null` 条件（从测试用例可见，如 [entity.service.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.spec.ts#L633)）：

```typescript
prisma.entity.findFirst({
  where: {
    id: ...,
    deletedAt: null,  // ← 自动过滤已软删除的实体
  },
});
```

---

## 八、实体关系 vs Blueprint 级联构建：两种完全独立的机制

### ⚠️ 关键区分：完全不是同一套机制

| 维度 | 实体关系（Entity Relation） | Blueprint 级联构建 |
|------|--------------------------|------------------|
| **作用对象** | Entity（数据模型实体） | Resource（服务、项目资源） |
| **表达载体** | EntityField（Lookup 类型字段） | Relation Block + `ResourceRelationCache` 表 |
| **核心数据** | `relatedEntityId`、`relatedFieldId`、`allowMultipleSelection`、`fkHolder` | `relationKey`、`relatedResources[]`、`parentShouldBuildWithChild` |
| **影响范围** | 代码生成（Prisma Schema、DTO、Service、Controller） | 构建触发顺序 |
| **级联语义** | 删除字段/实体时的联动处理 | 子资源构建时父资源也跟着构建 |
| **核心代码** | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts)、[prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts) | [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/relation/relation.service.ts) |

### 8.1 Blueprint 级联构建机制

核心实现位于 [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/relation/relation.service.ts#L219-L270)，使用 BFS 广度优先搜索：

```typescript
async getCascadingBuildableResourceIds(resourceIds: string[]): Promise<string[]> {
  const visited = new Set<string>(resourceIds);
  let queue = [...resourceIds];

  while (queue.length > 0) {
    const currentBatch = queue;
    queue = [];
    const newParents = await this.getBuildableParents(currentBatch);
    // ↑ 查询 ResourceRelationCache 中
    //   parentShouldBuildWithChild = true 且 childResourceId 在 currentBatch 中的记录
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

**级联构建的触发条件：**
- Blueprint 中定义了关系 `parentShouldBuildWithChild: true`
- 子资源（child）触发了构建
- 自动将所有可达的父资源（parent）也加入构建队列

### 8.2 两者对比图

```
┌─────────────────────────────────────────────────────────────┐
│                    Amplication 平台                           │
├─────────────────────────────┬───────────────────────────────┤
│     实体关系（数据层）        │   Blueprint 关系（资源层）     │
├─────────────────────────────┼───────────────────────────────┤
│ Entity A ──Lookup──▶ Entity B│  Resource A ◀───Relation──▶ Resource B │
│  - 生成 Prisma @relation     │  - 仅控制构建顺序              │
│  - 生成嵌套 DTO              │  - parentShouldBuildWithChild  │
│  - 生成 Service 方法         │  - BFS 级联触发构建            │
│  - 删除级联（Delete/ToScalar）│  - 与数据模型完全无关          │
└─────────────────────────────┴───────────────────────────────┘
```

---

## 九、完整示例

### 9.1 示例：User ↔ Post（一对多）

**配置：**
- User 实体字段：`posts` (Lookup, allowMultipleSelection=true, relatedEntity=Post)
- Post 实体字段：`author` (Lookup, allowMultipleSelection=false, relatedEntity=User)

**判定：**
- `posts`: `isToManyRelationField` → to-Many
- `author`: `isOneToOneRelationField` → to-One（注意：函数名是 isOneToOne 但实际是 to-One）

**生成的 Prisma Schema：**
```prisma
model User {
  id    String @id @default(cuid())
  posts Post[]
}

model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}
```

**生成的 DTO：**
```typescript
// User CreateInput：to-Many → 独立嵌套 DTO，仅含 connect
class CreateUserInput {
  posts?: PostCreateNestedManyWithoutAuthorInput;
}
class PostCreateNestedManyWithoutAuthorInput {
  connect?: PostWhereUniqueInput[];  // ← 仅 Connect！
}

// Post CreateInput：to-One → 直接用 WhereUniqueInput
class CreatePostInput {
  author?: UserWhereUniqueInput;  // 即 { id?: string }
}
```

### 9.2 示例：User ↔ Profile（一对一，User 持有外键）

**配置：**
- User 实体字段：`profile` (Lookup, allowMultipleSelection=false, fkHolder=user.profile.fieldId)
- Profile 实体字段：`user` (Lookup, allowMultipleSelection=false)

**生成的 Prisma Schema：**
```prisma
model User {
  id        String  @id @default(cuid())
  profile   Profile @relation(fields: [profileId], references: [id])
  profileId String  @unique
}

model Profile {
  id   String @id @default(cuid())
  user User?
}
```

---

## 十、关键文件速查表

| 文件 | 职责 |
|------|------|
| [lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/code-gen-types/src/schemas/lookup.json) | Lookup 字段属性 JSON Schema |
| [code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L98-L106) | `LookupResolvedProperties`、`EntityLookupField` 类型定义 |
| [field.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/field/field.ts) | **to-One/to-Many 判定函数**（注意命名歧义） |
| [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261) | 严格 One-to-One 判定、外键持有方判定、fkFieldName 默认值 |
| [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) | Lookup → Prisma Schema 字段转换 |
| [create-nested-input-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/nested-input-dto/create-nested-input-dto.ts) | to-Many 嵌套 DTO 生成（仅 Connect / Connect+Disconnect+Set） |
| [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts#L374-L486) | DTO 类型映射核心（to-One 直接用 WhereUniqueInput） |
| [entity-dto-type-enum.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/entity-dto-type-enum.ts) | DTO 类型枚举 |
| [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | 关系字段 CRUD、级联删除、实体软删除 |
| [EnumRelatedFieldStrategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/dto/EnumRelatedFieldStrategy.ts) | 删除策略枚举（Delete / UpdateToScalar） |
| [softDelete.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/util/softDelete.ts) | 软删除名称前缀工具 |
| [create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/service/create-service.ts) | Service 层方法生成（区分 to-One / to-Many） |
| [entity-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/entity-util.ts) | 关系字段默认 API Action |
| [dto-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/dto-util.ts) | 关系字段默认 DTO 配置 |
| [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/relation/relation.service.ts) | **Blueprint 级联构建**（与实体关系完全独立） |

---

## 十一、之前描述 vs 代码实际情况 对比汇总

| 主题 | 之前描述 | 代码实际情况 | 结论 |
|-----|---------|------------|------|
| `isOneToOneRelationField` | 判定严格一对一（同时检查两端） | 只检查本端 `allowMultipleSelection`，实际是 **to-One** 判定（包含 One-to-One 和 Many-to-One"多"端） | ❌ 命名有歧义，函数名不准确 |
| 嵌套 DTO 操作 | Create/Connect/ConnectOrCreate/Disconnect/Set 五种齐全 | **to-Many**：Create 场景仅 Connect，Update 场景 Connect+Disconnect+Set；**to-One**：无独立 DTO，直接用 WhereUniqueInput | ❌ Create 和 ConnectOrCreate 从未实际生成 |
| to-One 的 DTO | 有独立的 `CreateNestedOneWithout...` 类 | 不生成独立嵌套 DTO，直接引用 `XxxWhereUniqueInput`（只能通过 ID 关联） | ❌ 与描述不一致 |
| 实体删除 | 物理删除 | 软删除（`deletedAt` + 名称加 `__${id}_` 前缀 + 当前版本标记 `deleted=true`） | ❌ 是软删除不是硬删除 |
| UpdateToScalar 对端影响 | 认为"不对对端字段做任何修改" | `updateField()` 内部的 `shouldDeleteRelated`（Lookup→非Lookup）自动触发 `deleteRelatedField()`，**对端 EntityField/ModuleAction/ModuleDto 全部被删除** | ❌ 描述完全错误 |
| UpdateToScalar 本端转换 | 笼统说"转换为标量字段" | to-Many→`Json`（名称不变）；to-One→ID 标量类型（字段名加 `Id` 后缀，displayName 加 ` ID` 后缀） | ⚠️ 描述不够精确 |
| UpdateToScalar 本端清理 | 未提及 | 本端 **ModuleAction 和 ModuleDto 未被清理**（`return` 跳过了清理代码），存在残留数据 | ❌ 潜在 Bug |
| Blueprint 级联构建 | 未区分，可能误认为与实体关系有关 | 完全独立的机制，作用于 Resource 构建顺序（BFS 遍历 `parentShouldBuildWithChild`），与数据模型完全无关 | ❌ 需要明确区分 |
