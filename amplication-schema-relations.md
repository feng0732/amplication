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
| false | false | **One-to-One**（一对一） |
| false | true | **Many-to-One**（多对一，本端为"多"侧） |
| true | false | **One-to-Many**（一对多，本端为"一"侧） |
| true | true | **Many-to-Many**（多对多） |

判定工具函数位于 [field.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/field/field.ts#L34-L73)：

```typescript
export const isRelationField = (field: EntityField) =>
  field.dataType === EnumDataType.Lookup;

export const isOneToOneRelationField = (field: EntityLookupField) =>
  !field.properties.allowMultipleSelection &&
  !field.properties.relatedField.properties.allowMultipleSelection;

export const isToManyRelationField = (field: EntityLookupField) =>
  field.properties.allowMultipleSelection;
```

---

## 二、外键策略（Foreign Key Strategy）

### 2.1 外键持有方判定

在一对一关系中，只能有一侧持有外键列。Amplication 通过以下优先级判定哪一侧不生成外键（`isOneToOneWithoutForeignKey = true`）。

核心逻辑位于 [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261)：

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
| 非 One-to-One 关系 | 是（多对一中的"多"侧总是持有外键） |
| One-to-One + 指定了 fkHolder = 当前字段 ID | 是 |
| One-to-One + 指定了 fkHolder = 对方字段 ID | 否 |
| One-to-One + 未指定 fkHolder + 当前 permanentId < 对方 | 是 |
| One-to-One + 未指定 fkHolder + 当前 permanentId > 对方 | 否 |

### 2.2 外键字段名

外键列名由 `fkFieldName` 属性控制，默认规则：
- 若用户未显式指定：自动生成 `${fieldName}Id`
- 若用户显式指定：使用用户指定的名称

---

## 三、Prisma Schema 生成规则

Lookup 字段转换为 Prisma Schema 的核心逻辑位于 [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L280-L366)。

### 3.1 三种生成场景

#### 场景 A：Many-to-Many 或 无外键的 One-to-One

仅生成 Prisma 对象引用字段，不生成外键列：

```prisma
// 例：One-to-Many 的"一"侧 或 Many-to-Many
fieldName RelatedEntity @relation("RelationName")
```

条件：
- `isToManyRelationField(field)`（多对多/一对多的"多"侧反向），或
- `isOneToOneWithoutForeignKey === true`（一对一的无外键侧）

#### 场景 B：持有外键的 One-to-One

生成对象字段 + 标量外键字段 + `@unique` 约束：

```prisma
fieldName   RelatedEntity @relation("RelationName", fields: [fieldNameId], references: [id])
fieldNameId String        @unique // 一对一外键必须唯一
```

#### 场景 C：Many-to-One（持有外键的"多"侧）

生成对象字段 + 标量外键字段（无 `@unique`，因为多对一允许多条记录指向同一实体）：

```prisma
fieldName   RelatedEntity @relation("RelationName", fields: [fieldNameId], references: [id])
fieldNameId String
```

### 3.2 Prisma 关系命名

Prisma 的 `@relation("RelationName")` 名称由两端字段的唯一标识组合生成，确保双向关系正确配对。

---

## 四、关系对代码生成结构的影响

### 4.1 DTO 层生成

#### 4.1.1 嵌套输入 DTO 操作选项

定义于 [create-nested-input-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/nested-input-dto/create-nested-input-dto.ts#L20-L26)：

```typescript
export const NestedInputOperation = {
  Create: 1 << 0,           // 嵌套创建
  Connect: 1 << 1,          // 通过 ID 关联已有实体
  ConnectOrCreate: 1 << 2,  // 关联或创建
  Disconnect: 1 << 3,       // 解除关联（不删除目标实体）
  Set: 1 << 4,              // 替换整个关联集合
} as const;
```

**toOne vs toMany 的 DTO 差异：**

| 操作 | toOne（单值关联） | toMany（集合关联） |
|-----|------------------|------------------|
| CreateNestedInput | `CreateNestedOneWithout...Input` | `CreateNestedManyWithout...Input` |
| UpdateNestedInput | `UpdateNestedOneWithout...Input` | `UpdateManyWithout...Input` |
| 可用操作 | Connect / Disconnect / Create / ConnectOrCreate | Connect / Disconnect / Create / ConnectOrCreate / Set |

DTO 生成的默认策略见 [dto-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/dto-util.ts#L111-L137) 中的 `getDefaultDtosForRelatedEntity()`。

### 4.2 Service 层生成

Service 方法生成逻辑位于 [create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/service/create-service.ts)。

**toOne 关系（Parent 侧）：**
- `getParent()`：获取关联的父实体

**toMany 关系（Children 侧）：**
- `getChildren()`：分页获取子实体列表
- `findChild()`：按 ID 查找特定子实体
- `connectChild()` / `disconnectChild()`：关联/解除关联子实体
- `updateChild()`：更新子实体

API 端点生成由 [entity-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/entity-util.ts) 的 `getDefaultActionsForRelationField()` 控制。

### 4.3 ModuleAction 与 ModuleDto 自动同步

在创建 Lookup 字段时，除了创建字段本身，还会同步创建对应的权限和 DTO。核心流程位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2625-L2778) 的 `createField()`：

1. 创建本端 Lookup 字段
2. 自动创建反向关联字段（调用 `createRelatedField()`）
3. 为两端字段分别创建 ModuleAction（API 端点权限）
4. 为两端字段分别创建 ModuleDto（输入/输出 DTO 配置）

---

## 五、删除语义与级联行为

### 5.1 关联字段删除策略

定义于 [EnumRelatedFieldStrategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/dto/EnumRelatedFieldStrategy.ts)：

| 策略值 | 行为 |
|-------|------|
| `Delete` | 删除本端字段时，**级联删除**对端的关联字段 |
| `UpdateToScalar` | 删除本端字段时，将对端关联字段**转换为标量字段**（保留数据列） |

核心实现位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3119-L3257) 的 `deleteField()`：

```typescript
async deleteField(
  fieldId: string,
  relatedFieldStrategy: EnumRelatedFieldStrategy,
  ...
) {
  // ...
  if (isRelationField(field)) {
    const relatedField = await this.fieldService.findById(
      field.properties.relatedFieldId
    );
    if (relatedField) {
      switch (relatedFieldStrategy) {
        case EnumRelatedFieldStrategy.Delete:
          // 级联删除对端字段（包括其 ModuleAction 和 ModuleDto）
          await this.deleteRelatedField(field, ...);
          break;
        case EnumRelatedFieldStrategy.UpdateToScalar:
          // 将对端 Lookup 字段转换为标量字段（如 String）
          await this.updateRelatedFieldToScalar(field, ...);
          break;
      }
    }
  }
  // 最后删除本端字段
  await this.fieldService.delete(fieldId);
}
```

### 5.2 删除实体时的级联处理

位于 [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L941) 的 `deleteOneEntity()`：

1. 查询所有引用该实体的 Lookup 字段（通过 `relatedEntityId` 匹配）
2. 对每个引用字段执行级联策略（默认 `Delete`）
3. 删除该实体自身的所有字段、ModuleAction、ModuleDto
4. 最后删除实体记录

### 5.3 数据库层面的级联

**注意**：Amplication 生成的 Prisma Schema 默认**不包含** `onDelete: Cascade` 数据库级级联。删除语义主要在应用层（Service/Controller）控制，数据库外键默认使用 `ON DELETE RESTRICT` 或 Prisma 的默认行为。

如需数据库级级联，需要在生成后手动修改 Prisma Schema 或通过自定义代码实现。

---

## 六、Blueprint 级联构建

除了实体间的数据关系，Amplication 在 Blueprint 资源层面还有**级联构建**机制。

位于 [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/relation/relation.service.ts#L236-L270) 的 `getCascadingBuildableResourceIds()`：

```typescript
async getCascadingBuildableResourceIds(resourceId: string): Promise<string[]> {
  // 使用 BFS 广度优先搜索
  // 遍历所有 parentShouldBuildWithChild = true 的关系
  // 收集所有需要随当前资源一起构建的父资源 ID
}
```

**级联构建规则：**
- 当资源 A 修改触发构建时，若资源 B 与 A 存在 `parentShouldBuildWithChild = true` 的关系，则 B 也会被构建
- 支持多层级联（B 的父资源 C 也会被构建，以此类推）
- 使用 BFS 算法避免循环依赖

---

## 七、完整关系定义示例

### 7.1 示例：User ↔ Post（一对多）

**配置：**
- User 实体字段：`posts` (Lookup, allowMultipleSelection=true, relatedEntity=Post)
- Post 实体字段：`author` (Lookup, allowMultipleSelection=false, relatedEntity=User)

**生成的 Prisma Schema：**
```prisma
// User 模型（一"侧，无外键）
model User {
  id    String @id @default(cuid())
  posts Post[]
}

// Post 模型（"多"侧，持有外键）
model Post {
  id       String @id @default(cuid())
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}
```

**生成的 DTO 片段：**
```typescript
// Post 创建时的嵌套输入
class CreatePostInput {
  author?: CreateNestedOneWithoutPostsInput; // Connect/Create/ConnectOrCreate
}

// User 创建时的嵌套输入
class CreateUserInput {
  posts?: CreateNestedManyWithoutAuthorInput; // Connect/Create/ConnectOrCreate/Set
}
```

### 7.2 示例：User ↔ Profile（一对一，User 持有外键）

**配置：**
- User 实体字段：`profile` (Lookup, allowMultipleSelection=false, fkHolder=user.profile.fieldId)
- Profile 实体字段：`user` (Lookup, allowMultipleSelection=false)

**生成的 Prisma Schema：**
```prisma
// User 模型（持有外键侧）
model User {
  id        String  @id @default(cuid())
  profile   Profile @relation(fields: [profileId], references: [id])
  profileId String  @unique // 一对一外键必须唯一
}

// Profile 模型（无外键侧）
model Profile {
  id   String @id @default(cuid())
  user User?
}
```

---

## 八、关键文件速查表

| 文件 | 职责 |
|------|------|
| [lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/code-gen-types/src/schemas/lookup.json) | Lookup 字段属性 JSON Schema |
| [code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L98-L106) | `LookupResolvedProperties`、`EntityLookupField` 类型定义 |
| [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261) | 关系字段解析、外键持有方判定、fkFieldName 默认值 |
| [field.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/field/field.ts) | 关系类型判定工具函数 |
| [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) | Lookup → Prisma Schema 字段转换 |
| [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | 关系字段 CRUD、级联删除、双向字段同步 |
| [EnumRelatedFieldStrategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/entity/dto/EnumRelatedFieldStrategy.ts) | 删除策略枚举定义 |
| [create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/service/create-service.ts) | Service 层方法生成 |
| [create-nested-input-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/data-service-generator/src/server/resource/dto/nested-input-dto/create-nested-input-dto.ts) | 嵌套输入 DTO 生成 |
| [entity-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/entity-util.ts) | 关系字段默认 API Action |
| [dto-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/libs/util/dsg-utils/src/lib/dto-util.ts) | 关系字段默认 DTO 配置 |
| [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/107-amplication/packages/amplication-server/src/core/relation/relation.service.ts) | Blueprint 资源级联构建 |
