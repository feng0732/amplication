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
| properties | JsonValue | 可选 | 资源自定义属性（蓝图自定义属性） |
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

## 二、字段类型与 JSON Schema 约束

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

### 2.3 各字段类型完整 JSON Schema 定义

所有字段类型的属性约束定义在 [libs/util/code-gen-types/src/schemas/](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/) 目录下。

#### 2.3.1 主键类型 (Id)
[id.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/id.json)

```json
{
  "required": ["idType"],
  "additionalProperties": false,
  "properties": {
    "idType": {
      "type": "string",
      "enum": ["CUID", "UUID", "AUTO_INCREMENT", "AUTO_INCREMENT_BIG_INT"],
      "default": "CUID"
    }
  }
}
```

#### 2.3.2 单行文本 (SingleLineText)
[singleLineText.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/singleLineText.json)

```json
{
  "required": ["maxLength"],
  "additionalProperties": false,
  "properties": {
    "maxLength": {
      "type": "integer",
      "minimum": 1,
      "default": 256
    }
  }
}
```

#### 2.3.3 多行文本 (MultiLineText)
[multiLineText.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/multiLineText.json)

```json
{
  "required": ["maxLength"],
  "additionalProperties": false,
  "properties": {
    "maxLength": {
      "type": "integer",
      "minimum": 1,
      "default": 256
    }
  }
}
```

#### 2.3.4 邮箱 (Email)
[email.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/email.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.5 整数 (WholeNumber)
[wholeNumber.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/wholeNumber.json)

```json
{
  "required": ["minimumValue", "maximumValue"],
  "additionalProperties": false,
  "properties": {
    "databaseFieldType": {
      "type": "string",
      "enum": ["INT", "BIG_INT"],
      "default": "INT"
    },
    "minimumValue": {
      "type": "integer",
      "default": 0
    },
    "maximumValue": {
      "type": "integer",
      "default": 99999999999
    }
  }
}
```

#### 2.3.6 小数 (DecimalNumber)
[decimalNumber.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/decimalNumber.json)

```json
{
  "required": ["minimumValue", "maximumValue", "precision"],
  "additionalProperties": false,
  "properties": {
    "databaseFieldType": {
      "type": "string",
      "enum": ["DECIMAL", "FLOAT"],
      "default": "DECIMAL"
    },
    "minimumValue": {
      "type": "integer",
      "default": 0
    },
    "maximumValue": {
      "type": "integer",
      "default": 99999999999
    },
    "precision": {
      "type": "integer",
      "minimum": 0,
      "maximum": 8,
      "default": 8
    }
  }
}
```

#### 2.3.7 日期时间 (DateTime)
[dateTime.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/dateTime.json)

```json
{
  "required": ["timeZone", "dateOnly"],
  "additionalProperties": false,
  "properties": {
    "timeZone": {
      "type": "string",
      "enum": ["localTime", "serverTime"],
      "default": "localTime"
    },
    "dateOnly": {
      "type": "boolean",
      "default": false
    }
  }
}
```

#### 2.3.8 布尔 (Boolean)
[boolean.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/boolean.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.9 JSON
[json.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/json.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.10 关联查找 (Lookup)
[lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/lookup.json)

```json
{
  "required": ["relatedEntityId", "relatedFieldId", "allowMultipleSelection"],
  "additionalProperties": false,
  "properties": {
    "relatedEntityId": { "type": "string" },
    "relatedFieldId": { "type": "string" },
    "allowMultipleSelection": { "type": "boolean" },
    "fkHolder": { "type": ["string", "null"] },
    "fkFieldName": { "type": "string" }
  }
}
```

#### 2.3.11 单选枚举 (OptionSet)
[optionSet.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/optionSet.json)

```json
{
  "required": ["options"],
  "additionalProperties": false,
  "properties": {
    "options": {
      "type": "array",
      "uniqueItems": true,
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["label", "value"],
        "additionalProperties": false,
        "properties": {
          "label": { "type": "string" },
          "value": {
            "type": "string",
            "pattern": "^(?![0-9])[a-zA-Z0-9$_]+$"
          }
        }
      }
    }
  }
}
```

#### 2.3.12 多选枚举 (MultiSelectOptionSet)
[multiSelectOptionSet.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/multiSelectOptionSet.json)

```json
{
  "required": ["options"],
  "additionalProperties": false,
  "properties": {
    "options": {
      "type": "array",
      "uniqueItems": true,
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["label", "value"],
        "additionalProperties": false,
        "properties": {
          "label": { "type": "string" },
          "value": {
            "type": "string",
            "pattern": "^(?![0-9])[a-zA-Z0-9$_]+$"
          }
        }
      }
    }
  }
}
```

#### 2.3.13 地理位置 (GeographicLocation)
[geographicLocation.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/geographicLocation.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.14 文件 (File)
[file.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/file.json)

```json
{
  "additionalProperties": false,
  "properties": {
    "maxFileSize": {
      "type": "number",
      "minimum": 0
    },
    "allowedMimeTypes": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "string" }
    },
    "containerPath": {
      "type": "string",
      "default": "/"
    }
  }
}
```

#### 2.3.15 创建时间 (CreatedAt)
[createdAt.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/createdAt.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.16 更新时间 (UpdatedAt)
[updatedAt.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/updatedAt.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

#### 2.3.17 用户名 (Username)
[username.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/username.json)

```json
{
  "additionalProperties": false,
  "properties": {
    "maxLength": {
      "type": "integer",
      "minimum": 1,
      "default": 256
    }
  }
}
```

#### 2.3.18 密码 (Password)
[password.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/password.json)

```json
{
  "additionalProperties": false,
  "properties": {
    "maxLength": {
      "type": "integer",
      "minimum": 1,
      "default": 256
    }
  }
}
```

#### 2.3.19 角色集合 (Roles)
[roles.json](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/schemas/roles.json)

```json
{
  "additionalProperties": false,
  "properties": {}
}
```

### 2.4 默认属性映射
[constants.ts#L184-L230](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts#L184-L230)

`DATA_TYPE_TO_DEFAULT_PROPERTIES` 定义了每种数据类型的默认属性值。

---

## 三、资源 Properties 与蓝图自定义属性校验

### 3.1 自定义属性类型 (EnumCustomPropertyType)
[customProperty.service.ts#L19](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/customProperty/dto/EnumCustomPropertyType.ts)

```typescript
enum EnumCustomPropertyType {
  Text,         // 文本
  Link,         // 链接
  Select,       // 单选
  MultiSelect,  // 多选
}
```

### 3.2 customProperties 查询的 blueprintId 过滤逻辑（核心）
[customProperty.service.ts#L36-L60](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts#L36-L60)

```typescript
async customProperties(args: CustomPropertyFindManyArgs): Promise<CustomProperty[]> {
  args.where = args.where || {};

  // 关键：未指定 blueprintId 或 blueprint 时，默认只返回全局属性
  if (
    args.where.blueprintId === undefined &&
    args.where.blueprint === undefined
  ) {
    args.where.blueprintId = null;  // 强制过滤：只返回 blueprintId = null 的全局属性
  }

  const properties = await this.prisma.customProperty.findMany({
    ...args,
    where: {
      ...args.where,
      deletedAt: null,
    },
  });
  // ...
}
```

**过滤规则总结：**

| 查询条件 | 实际过滤的 blueprintId | 返回的属性类型 |
|---------|----------------------|-------------|
| 未指定 blueprintId 和 blueprint | `null` | 全局属性 |
| 指定 `blueprintId: null` | `null` | 全局属性 |
| 指定 `blueprintId: "xxx"` | `"xxx"` | 指定蓝图的属性 |

### 3.3 两套独立的属性校验体系

#### 3.3.1 全局属性校验 - `validateResourceProperties`
[resource.service.ts#L1524-L1552](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1524-L1552)

```typescript
async validateResourceProperties(
  values: Record<string, unknown>,
  user: User
): Promise<void> {
  if (!values || Object.keys(values).length === 0) {
    return;
  }

  // 关键：查询时未指定 blueprintId → 根据 customProperties 默认逻辑，
  // 只会返回 blueprintId = null 的全局属性
  const customProperties = await this.customPropertyService.customProperties({
    where: {
      workspace: { id: user.workspace.id },
      enabled: true,
      // 没有 blueprintId 过滤条件 → 默认为 blueprintId = null
    },
  });

  const validationResults =
    await this.customPropertyService.validateCustomProperties(
      customProperties,  // 只包含全局属性
      values
    );

  if (!validationResults.isValid) {
    throw new AmplicationError(
      `Validation failed for resource properties: ${validationResults.errorText}`
    );
  }
}
```

**调用时机**：仅在 `updateResource` 时调用
[resource.service.ts#L1568-L1571](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1568-L1571)

```typescript
async updateResource(args: UpdateOneResourceArgs, user: User): Promise<Resource | null> {
  // ...
  await this.validateResourceProperties(
    args.data.properties as Record<string, unknown>,
    user
  );
  // ...
}
```

**存储位置**：`Resource.properties` 字段

**注意**：`createResource` 时**不调用**此校验。

#### 3.3.2 蓝图属性校验 - `validateResourceSettingsProperties`
[resourceSettings.service.ts#L63-L102](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts#L63-L102)

```typescript
async validateResourceSettingsProperties(
  resourceId: string,
  properties: Record<string, unknown>
): Promise<void> {
  if (!properties || Object.keys(properties).length === 0) {
    return;
  }

  const resource = await this.resourceService.resource({
    where: { id: resourceId },
  });

  if (!resource) {
    throw new Error(`Resource not found with id ${resourceId}`);
  }

  if (!resource.blueprintId) {
    throw new Error(`Blueprint not found for resource with id ${resourceId}`);
  }

  // 关键：通过 blueprintService.properties 查询，显式指定 blueprintId
  const blueprintProperties = await this.blueprintService.properties({
    where: {
      id: resource.blueprintId,  // 显式指定蓝图 ID
    },
  });

  const validationResults =
    await this.customPropertyService.validateCustomProperties(
      blueprintProperties,  // 只包含该蓝图的属性
      properties
    );

  if (!validationResults.isValid) {
    throw new AmplicationError(
      `Validation failed for resource settings properties: ${validationResults.errorText}`
    );
  }
}
```

**blueprintService.properties 实现**：
[blueprint.service.ts#L391-L397](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L391-L397)

```typescript
async properties(args: FindOneArgs): Promise<CustomProperty[]> {
  return this.customPropertyService.customProperties({
    where: {
      blueprintId: args.where.id,  // 显式指定 blueprintId，绕过默认的 null 过滤
    },
  });
}
```

**调用时机**：在 `updateResourceSettings` 时调用
[resourceSettings.service.ts#L108-L111](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts#L108-L111)

```typescript
async updateResourceSettings(args: UpdateResourceSettingsArgs, user: User): Promise<ResourceSettings> {
  await this.validateResourceSettingsProperties(
    args.where.id,
    args.data.properties as Record<string, unknown>
  );
  // ...
}
```

**存储位置**：`ResourceSettings.properties` 字段（独立的 Block 存储）

### 3.4 全局属性 vs 蓝图属性校验边界

| 维度 | 全局属性校验 | 蓝图属性校验 |
|------|------------|-----------|
| 校验方法 | `ResourceService.validateResourceProperties` | `ResourceSettingsService.validateResourceSettingsProperties` |
| 存储位置 | `Resource.properties` | `ResourceSettings.properties` |
| 查询时 blueprintId 过滤 | 未指定 → 默认 `null` | 显式指定 `resource.blueprintId` |
| 校验的属性类型 | `blueprintId = null` 的全局属性 | `blueprintId = resource.blueprintId` 的蓝图属性 |
| 调用时机 | `updateResource`（更新资源基本信息） | `updateResourceSettings`（更新资源设置） |
| createResource 时是否校验 | ❌ 否 | ❌ 否 |
| 资源无 blueprintId 时 | ✅ 正常工作 | ❌ 抛出错误 |
| additionalProperties | ✅ `false`（禁止额外属性） | ✅ `false`（禁止额外属性） |

### 3.5 自定义属性校验 Schema 生成
[customProperty.service.ts#L308-L386](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts#L308-L386)

```typescript
getValidationSchema(customProperties: CustomProperty[]): JSONSchema {
  const properties: Record<string, JSONSchema> = {};
  const required: string[] = [];
  const errorMessage = { properties: {} };

  for (const customProperty of customProperties) {
    const key = customProperty.key;
    const schema: JSONSchema = {
      title: customProperty.name,
      type: "string",
    };

    // Select 类型：枚举值校验
    if (customProperty.type === EnumCustomPropertyType.Select) {
      schema.enum = customProperty.options.map((option) => option.value);
      if (!customProperty.required) {
        schema.enum = [null, ...schema.enum];
        schema.type = ["string", "null"];
      }
    }

    // MultiSelect 类型：数组枚举校验
    if (customProperty.type === EnumCustomPropertyType.MultiSelect) {
      schema.type = "array";
      schema.items = {
        type: "string",
        enum: customProperty.options.map((option) => option.value),
      };
      if (!customProperty.required) {
        schema.type = ["array", "null"];
        schema.items.enum = [null, ...schema.items.enum];
        schema.items.type = ["string", "null"];
      }
    }

    // Text/Link 类型
    if (
      customProperty.type === EnumCustomPropertyType.Text ||
      customProperty.type === EnumCustomPropertyType.Link
    ) {
      if (!customProperty.required) {
        schema.type = ["string", "null"];
      }
    }

    // 必填校验
    if (customProperty.required) {
      required.push(key);
      schema.isNotEmpty = true;
      errorMessage.properties[key] = `${customProperty.name} is required`;
      
      if (customProperty.type === EnumCustomPropertyType.MultiSelect) {
        schema.minItems = 1;
        errorMessage.properties[key] = `At least one ${customProperty.name} is required`;
      }
    }

    // 正则校验规则
    if (customProperty.validationRule) {
      schema.pattern = customProperty.validationRule;
      errorMessage.properties[key] = customProperty.validationMessage;
    }

    properties[key] = schema;
  }

  return {
    additionalProperties: false,  // 禁止额外属性
    type: "object",
    required,
    errorMessage,
    properties,
  };
}
```

### 3.6 JSON Schema 校验服务
[jsonSchemaValidation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/services/jsonSchemaValidation.service.ts)

```typescript
class JsonSchemaValidationService {
  public async validateSchema(schema: object, data: any): Promise<SchemaValidationResult> {
    const ajv: Ajv.Ajv = new Ajv({ allErrors: true });

    // 自定义关键字：非空字符串校验
    ajv.addKeyword("isNotEmpty", {
      type: "string",
      validate: function (schema, data) {
        return typeof data === "string" && data.trim() !== "";
      },
      errors: true,
    });
    
    ajvErrors(ajv);
    const isValid = ajv.validate(schema, data);
    
    if (!isValid) {
      return new SchemaValidationResult(false, ajv.errorsText());
    }
    return new SchemaValidationResult(true);
  }
}
```

---

## 四、关联关系定义

### 4.1 Lookup 关联字段类型定义

**代码生成时解析后的 Lookup 属性：**
[code-gen-types.ts#L98-L102](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/code-gen-types/src/code-gen-types.ts#L98-L102)

```typescript
type LookupResolvedProperties = Lookup & {
  relatedEntity: Entity;              // 关联实体对象
  relatedField: EntityField;          // 反向关联字段对象
  isOneToOneWithoutForeignKey?: boolean; // 一对一关系中不持有外键的一方
};
```

### 4.2 关联类型判断

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

### 4.3 资源间关联 (Resource Relation)

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

## 五、关联传播逻辑（详细）

### 5.1 Lookup 字段双向关联自动创建

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

### 5.2 Lookup 字段更新传播（详细逻辑）
[entity.service.ts#L2900-L3117](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2900-L3117)

#### 更新条件判断
```typescript
// 条件1：类型从 Lookup 改为其他类型 → 需要删除反向关联
const shouldDeleteRelated =
  field.dataType === EnumDataType.Lookup &&
  args.data.dataType !== EnumDataType.Lookup;

// 条件2：类型从其他改为 Lookup → 需要创建反向关联
const shouldCreateRelated =
  args.data.dataType === EnumDataType.Lookup &&
  field.dataType !== EnumDataType.Lookup;

// 条件3：关联实体 ID 变化 → 需要重建反向关联（先删后建）
const shouldChangeRelated =
  !shouldCreateRelated &&
  !shouldDeleteRelated &&
  args.data.properties?.relatedEntityId !==
    (field.properties as unknown as types.Lookup)?.relatedEntityId;

// 条件4：字段名称/显示名/多选属性变化 → 需要更新操作名称
const shouldUpdateRelatedFieldActions =
  args.data.dataType === EnumDataType.Lookup &&
  field.dataType === EnumDataType.Lookup &&
  (field.name !== args.data.name ||
    field.displayName !== args.data.displayName ||
    (field.properties as unknown as types.Lookup).allowMultipleSelection !==
      (args.data.properties as unknown as types.Lookup).allowMultipleSelection);

// 条件5：关系类型变化（一对一 ↔ 一对多）
const relationTypeChanged =
  args.data.dataType === EnumDataType.Lookup &&
  field.dataType === EnumDataType.Lookup &&
  (args.data.properties as unknown as types.Lookup)?.allowMultipleSelection !==
    (field.properties as unknown as types.Lookup)?.allowMultipleSelection;
```

#### 更新执行流程
```typescript
return await this.useLocking(entityId, user, async (entity) => {
  // 1. 参数校验
  this.validateFieldMutationArgs(args, entity);

  // 2. 需要创建/变更关联时，生成新的反向字段 permanentId
  if (shouldCreateRelated || shouldChangeRelated) {
    args.data.properties.relatedFieldId = cuid();
  }

  // 3. 数据校验
  await this.validateFieldData(args.data, entity, enforceValidation);

  // 4. 删除旧的反向关联（需要删除或变更时）
  if (shouldDeleteRelated || shouldChangeRelated) {
    const properties = field.properties as unknown as types.Lookup;
    await this.deleteRelatedField(
      properties.relatedFieldId,
      properties.relatedEntityId,
      user
    );
  }

  // 5. 创建新的反向关联（需要创建或变更时）
  if (shouldCreateRelated || shouldChangeRelated) {
    const properties = args.data.properties as unknown as types.Lookup;
    await this.createRelatedField(
      properties.relatedFieldId,
      args.relatedFieldName,
      args.relatedFieldDisplayName,
      args.relatedFieldAllowMultipleSelection,
      properties.relatedEntityId,
      entity.id,
      field.permanentId,
      user,
      properties.fkHolder
    );
  }

  // 6. 更新当前字段（移除客户端传入的相关字段参数）
  const updatedField = await this.prisma.entityField.update(
    omit(args, [
      "relatedFieldName",
      "relatedFieldDisplayName",
      "relatedFieldAllowMultipleSelection",
    ])
  );

  // 7. 更新关联字段的操作名称
  if (shouldUpdateRelatedFieldActions || relationTypeChanged) {
    const moduleId = await this.moduleService.getDefaultModuleIdForEntity(...);
    await this.moduleActionService.updateDefaultActionsForRelationField(...);
  }

  // 8. 枚举类型变化处理
  if (args.data.dataType === OptionSet || args.data.dataType === MultiSelectOptionSet) {
    // 更新枚举 DTO
    await this.moduleDtoService.updateDefaultDtoForEnumField(...);
  } else if (原类型是枚举类型) {
    // 删除枚举 DTO
    await this.moduleDtoService.deleteDefaultDtoForEnumField(...);
  }

  // 9. 同步 fkHolder 到反向字段
  if (field.dataType === Lookup && updateFieldProperties?.fkHolder !== null) {
    const relatedField = await this.getField({ where: { permanentId: ... } });
    relatedFieldProps.fkHolder = updateFieldProperties.fkHolder;
    await this.prisma.entityField.update({
      where: { id: relatedField.id },
      data: { properties: relatedFieldProps },
    });
  }

  return updatedField;
});
```

### 5.3 Lookup 字段删除传播（详细逻辑）
[entity.service.ts#L3119-L3257](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3119-L3257)

#### 删除策略枚举
```typescript
enum EnumRelatedFieldStrategy {
  Delete,        // 删除反向字段（默认）
  UpdateToScalar, // 将反向字段转为标量字段
}
```

#### 删除执行流程
```typescript
async deleteField(
  args: DeleteEntityFieldArgs,
  user: User,
  fieldStrategy = EnumRelatedFieldStrategy.Delete
): Promise<EntityField | null> {
  const field = await this.getField({...});

  // 系统字段不可删除
  if (isSystemDataType(field.dataType as EnumDataType)) {
    throw new ConflictException(...);
  }

  return await this.useLocking(entityId, user, async (entity) => {
    if (field.dataType === EnumDataType.Lookup) {
      const properties = field.properties as unknown as types.Lookup;
      
      // 策略1：删除反向字段（默认）
      if (fieldStrategy === EnumRelatedFieldStrategy.Delete) {
        try {
          await this.deleteRelatedField(
            properties.relatedFieldId,
            properties.relatedEntityId,
            user
          );
        } catch (error) {
          // 容错：反向字段删除失败仍继续删除当前字段
          this.logger.error(
            "Continue with FieldDelete even though the related field could not be deleted ",
            error
          );
        }
      } 
      // 策略2：将反向字段转为标量字段
      else if (fieldStrategy === EnumRelatedFieldStrategy.UpdateToScalar) {
        const allowMultipleSelection = properties.allowMultipleSelection;
        
        // 多选 → JSON，单选 → 关联实体主键类型
        field.dataType = allowMultipleSelection
          ? EnumDataType.Json
          : await this.getRelatedFieldScalarTypeByRelatedEntityIdType(
              properties.relatedEntityId
            );

        // 字段名转换：fieldName → fieldNameId（单选用）
        const data: EntityFieldUpdateInput = {
          dataType: field.dataType,
          name: allowMultipleSelection ? field.name : `${field.name}Id`,
          displayName: allowMultipleSelection
            ? field.displayName
            : `${field.displayName} ID`,
          properties: DATA_TYPE_TO_DEFAULT_PROPERTIES[field.dataType],
        };

        // 执行更新（转为标量字段）
        await this.updateField({ data, where: { id: args.where.id } }, user);
        return;
      }
    }

    // 删除当前字段
    const deletedField = await this.prisma.entityField
      .delete({ where: { id: args.where.id } })
      .catch((error) => {
        if (error.code === "P2025") {
          // 字段已不存在，容错继续
          return null;
        }
        throw new AmplicationError(error);
      });

    // 清理关联的操作和 DTO
    if (deletedField) {
      const moduleId = await this.moduleService.getDefaultModuleIdForEntity(...);
      
      if (deletedField.dataType === EnumDataType.Lookup) {
        await this.moduleActionService.deleteDefaultActionsForRelationField(...);
        await this.moduleDtoService.deleteDefaultDtosForRelatedEntity(...);
      }
      
      if (deletedField.dataType === EnumDataType.OptionSet || 
          deletedField.dataType === EnumDataType.MultiSelectOptionSet) {
        await this.moduleDtoService.deleteDefaultDtoForEnumField(...);
      }
    }

    return deletedField;
  });
}
```

#### deleteRelatedField 内部实现
[entity.service.ts#L2849-L2898](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2849-L2898)

```typescript
private async deleteRelatedField(
  permanentId: string,
  entityId: string,
  user: User
): Promise<void> {
  await this.useLocking(entityId, user, async (entity) => {
    // 1. 获取待删除字段
    const field = await this.getField({ where: { permanentId } });

    // 2. 删除反向字段
    const deletedField = await this.prisma.entityField.delete({
      where: {
        entityVersionId_permanentId: {
          permanentId,
          entityVersionId: field.entityVersionId,
        },
      },
      include: { entityVersion: { include: { entity: true } } },
    });

    // 3. 删除关联的操作和 DTO
    const moduleId = await this.moduleService.getDefaultModuleIdForEntity(...);
    
    await this.moduleActionService.deleteDefaultActionsForRelationField(
      deletedField,
      moduleId,
      user
    );

    await this.moduleDtoService.deleteDefaultDtosForRelatedEntity(
      deletedField,
      deletedField.entityVersion.entity,
      moduleId,
      user
    );
  });
}
```

### 5.4 一对一关系外键持有方判断
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

### 5.5 资源级联构建
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

## 六、从模板导入到生成上下文的数据流向

### 6.1 模板导入流程
[serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts)

#### 6.1.1 从模板创建资源
```typescript
async createResourceFromTemplate(args, user): Promise<Resource> {
  // 1. 验证模板可用性
  const template = await this.availableServiceTemplatesForProject(...);
  
  // 2. 获取最新模板版本
  const templateVersion = await this.resourceVersionService.getLatest(template.id);
  
  // 3. 根据蓝图类型创建资源
  let newResource: Resource;
  if (resourceType === EnumResourceType.Component) {
    newResource = await this.internalCreateComponentFromTemplate(...);
  } else {
    newResource = await this.internalCreateServiceFromTemplate(...);
  }
  
  // 4. 记录模板版本关联
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    serviceTemplateId: template.id,
    version: templateVersion.version,
  });
  
  // 5. 复制插件安装
  await this.copyPluginInstallations(template.id, newResource.id, user);
  
  // 6. 可选：构建后自动提交
  if (args.data.buildAfterCreation) {
    await this.projectService.commit(...);
  }
  
  return newResource;
}
```

#### 6.1.2 服务模板创建（替换变量）
```typescript
private async internalCreateServiceFromTemplate(args, template, user) {
  const serviceSettings = await this.serviceSettingsService.getServiceSettingsValues(...);
  
  // 替换模板中的 {{SERVICE_NAME}} 变量
  const kebabCaseServiceName = kebabCase(args.data.name);
  
  serviceSettings.adminUISettings.adminUIPath = 
    serviceSettings.adminUISettings.adminUIPath.replace(
      "{{SERVICE_NAME}}",
      kebabCaseServiceName
    );
  
  serviceSettings.serverSettings.serverPath = 
    serviceSettings.serverSettings.serverPath.replace(
      "{{SERVICE_NAME}}",
      kebabCaseServiceName
    );
  
  // 创建新服务
  return await this.resourceService.createService(...);
}
```

#### 6.1.3 插件安装复制
```typescript
async copyPluginInstallations(sourceResourceId, targetResourceId, user) {
  const plugins = await this.pluginInstallationService.getOrderedPluginInstallations(
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

    await this.pluginInstallationService.create({ data: createInput }, user);
  }
}
```

### 6.2 模板升级流程
```typescript
async upgradeServiceToLatestTemplateVersion(args, user) {
  // 1. 获取当前模板版本
  const serviceTemplateVersion = await this.resourceService.getServiceTemplateSettings(...);
  
  // 2. 获取最新版本
  const latestVersion = await this.resourceVersionService.getLatest(template.id);
  
  // 3. 版本比较
  const changes = await this.resourceVersionService.compareResourceVersions({
    sourceVersion: serviceTemplateVersion.version,
    targetVersion: latestVersion.version,
  });
  
  // 4. 合并变更
  const mergeOptions: BlockMergeOptions = {
    updatedManuallyCreatedBlocks: true,
  };
  
  // 新增块
  const createdPromises = changes.createdBlocks.map(async (blockVersion) => {
    return this.handleMergeCreatedBlock(resourceId, blockVersion, user, mergeOptions);
  });
  
  // 删除块
  const deletedPromises = changes.deletedBlocks.map(...);
  
  // 更新块
  const updatedPromises = changes.updatedBlocks.forEach(...);
  
  await Promise.all([createdPromises, deletedPromises, updatedPromises]);
  
  // 5. 更新版本记录
  await this.resourceTemplateVersionService.updateResourceTemplateVersion(...);
}
```

### 6.3 DSG 数据构建流程
[build.service.ts#L1384-L1521](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/build/build.service.ts#L1384-L1521)

#### 6.3.1 getDSGResourceData 数据收集
```typescript
async getDSGResourceData(resource, buildId, buildVersion, user, rootGeneration = true) {
  const resourceId = resource.id;
  
  // 1. 插件安装（启用的）
  const orderedPlugins = (
    await this.pluginInstallationService.getOrderedPluginInstallations(resourceId)
  ).filter((plugin) => plugin.enabled);
  
  // 2. 模块操作和 DTO
  const moduleActions = await this.moduleActionService.findMany(...);
  const moduleDtos = await this.moduleDtoService.findMany(...);
  const modules = await this.moduleService.findMany(...);
  
  // 3. 资源关联
  const relations = await this.resourceService.getRelations(resourceId);
  
  // 4. 资源设置和服务设置
  const resourceSettings = await this.resourceSettingsService.getResourceSettingsBlock(...);
  const serviceSettings = resource.resourceType === EnumResourceType.Service
    ? await this.serviceSettingsService.getServiceSettingsValues(...)
    : undefined;
  
  // 5. 关联资源数据（递归收集）
  let otherResources = undefined;
  if (rootGeneration) {
    const resources = await this.resourceService.getRelatedResourcesRecursive(resourceId);
    
    // Service 类型额外包含项目内其他资源
    if (resource.resourceType === EnumResourceType.Service) {
      const projectResources = await this.resourceService.resources(...);
      resources = [...resources, ...projectResources.filter(...)];
    }
    
    otherResources = await Promise.all(
      resources
        .filter(({ id }) => id !== resourceId)
        .map((resource) =>
          this.getDSGResourceData(resource, buildId, buildVersion, user, false)
        )
    );
  }
  
  // 6. 构建 DSGResourceData
  const dsgResourceData: CodeGenTypes.DSGResourceData = {
    entities: rootGeneration ? await this.getOrderedEntities(buildId) : [],
    roles: await this.getResourceRoles(resourceId),
    pluginInstallations: orderedPlugins,
    resourceSettings,
    relations,
    moduleContainers: modules,
    moduleActions,
    moduleDtos,
    resourceType: resource.resourceType,
    topics: await this.topicService.findMany(...),
    serviceTopics: await this.serviceTopicsService.findMany(...),
    buildId,
    resourceInfo: {
      properties: resource.properties,  // 资源自定义属性
      name: resource.name,
      description: resource.description,
      version: buildVersion,
      id: resourceId,
      url,
      settings: serviceSettings,
      codeGeneratorVersionOptions: {...},
      codeGeneratorName: resource.codeGeneratorName,
    },
    otherResources,
  };

  return omitDeep(dsgResourceData, DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE);
}
```

#### 6.3.2 数据保存与消息传递
[build.service.ts#L568-L618](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

```typescript
private async generate(logger, build, user) {
  return this.actionService.run(build.actionId, GENERATE_STEP_NAME, ..., async (step) => {
    const { resourceId, id: buildId, version: buildVersion } = build;

    // 1. 收集生成数据
    const resource = await this.resourceService.resource({ where: { id: resourceId } });
    const dsgResourceData = await this.getDSGResourceData(resource, buildId, buildVersion, user);

    // 2. 保存到共享存储（避免 Kafka 消息过大）
    await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);

    // 3. 发送轻量级消息到 Kafka
    const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
      key: null,
      value: { resourceId, buildId },
    };

    await this.kafkaProducerService.emitMessage(
      KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
      codeGenerationEvent
    );

    return null;
  });
}
```

### 6.4 生成上下文准备 (prepare-context)
[prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts)

```typescript
export async function prepareContext(dSGResourceData, internalLogger, pluginInstallationPath) {
  const {
    pluginInstallations,
    entities,
    roles,
    resourceInfo: appInfo,
    otherResources,
    moduleActions,
    moduleContainers,
    moduleDtos,
  } = dSGResourceData;

  // 1. 注册插件
  const plugins = await registerPlugins(pluginInstallations, pluginInstallationPath);

  // 2. 实体名称复数化
  const entitiesWithPluralName = prepareEntityPluralName(entities);

  // 3. 解析 Lookup 字段（ID → 对象）
  const normalizedEntities = resolveLookupFields(entitiesWithPluralName);

  // 4. 服务主题名称处理
  const serviceTopicsWithName = prepareServiceTopics(dSGResourceData);

  // 5. 填充上下文
  const context = DsgContext.getInstance;
  context.appInfo = appInfo;
  context.roles = roles;
  context.entities = normalizedEntities;
  context.serviceTopics = serviceTopicsWithName;
  context.otherResources = otherResources;
  context.pluginInstallations = pluginInstallations;
  context.moduleContainers = moduleContainers;
  context.moduleDtos = moduleDtos;

  // 6. 模块操作和 DTO 映射
  context.moduleActionsAndDtoMap = prepareModuleActionsAndDtos(...);

  // 7. 实体操作映射
  context.entityActionsMap = prepareEntityActions(...);

  // 8. 特性检测
  context.generateGrpc = shouldGenerateGrpc(context.pluginInstallations);
  context.hasDecimalFields = normalizedEntities.some(...);
}
```

### 6.5 Lookup 字段解析（ID 到对象）
[prepare-context.ts#L185-L261](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts#L185-L261)

```typescript
function resolveLookupFields(entities: Entity[]): Entity[] {
  // 1. 建立索引
  const entityIdToEntity: Record<string, Entity> = {};
  const fieldIdToField: Record<string, EntityField> = {};

  entities.forEach((entity) => {
    entityIdToEntity[entity.id] = entity;
    entity.fields.forEach((field) => {
      fieldIdToField[field.permanentId] = field;
    });
  });

  // 2. 解析每个 Lookup 字段
  return entities.map((entity) => ({
    ...entity,
    fields: entity.fields.map((field) => {
      if (field.dataType === EnumDataType.Lookup) {
        const fieldProperties = field.properties as types.Lookup;
        const {
          relatedEntityId,
          relatedFieldId,
          fkHolder,
          fkFieldName,
          allowMultipleSelection,
        } = fieldProperties;

        const relatedEntity = entityIdToEntity[relatedEntityId];
        const relatedField = fieldIdToField[relatedFieldId];
        const relatedFieldProperties = relatedField?.properties as types.Lookup;

        // 一对一关系外键持有方判断
        const isOneToOne = !allowMultipleSelection && 
                          !relatedFieldProperties?.allowMultipleSelection;
        
        let isOneToOneWithoutForeignKey = true;
        if (fkHolder) {
          isOneToOneWithoutForeignKey = isOneToOne && field.permanentId !== fkHolder;
        } else {
          isOneToOneWithoutForeignKey = isOneToOne && field.permanentId > relatedField?.permanentId;
        }

        const properties: LookupResolvedProperties = {
          ...field.properties,
          relatedEntity,
          relatedField,
          isOneToOneWithoutForeignKey,
          fkFieldName: !isEmpty(trim(fkFieldName))
            ? fkFieldName
            : `${field.name}Id`,
        };

        return { ...field, properties };
      }
      return field;
    }),
  }));
}
```

---

## 七、生成侧影响

### 7.1 Prisma Schema 生成影响
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

### 7.2 DTO 生成影响
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

### 7.3 GraphQL API 生成影响

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

### 7.4 外键字段命名规则
[prepare-context.ts#L248-L250](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts#L248-L250)

```typescript
fkFieldName: !isEmpty(trim(fkFieldName))
  ? fkFieldName              // 用户指定
  : `${field.name}Id`,       // 默认：字段名 + Id
```

### 7.5 关联名称生成规则
[create-prisma-schema-fields.ts#L538-L582](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L538-L582)

当两个实体间存在多个关联时，需要生成关联名称以区分：

1. 优先匹配实体名与字段名的关系
2. 字段名唯一时使用字段名
3. 都不唯一时组合实体名和字段名

---

## 八、字段校验流程

### 8.1 创建/更新字段时的校验
[entity.service.ts#L2667](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L2667)

```typescript
this.validateFieldMutationArgs(args, entity);   // 参数校验
await this.validateFieldData(data, entity, enforceValidation);  // 数据校验
```

### 8.2 命名规范校验
[entity.service.ts#L147-L149](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L147-L149)

```typescript
const NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/;
```

### 8.3 JSON Schema 校验
使用 JSON Schema 校验各数据类型的 `properties` 字段：
- 从 `@amplication/code-gen-types` 获取对应类型的 schema
- 通过 `JsonSchemaValidationService` 执行校验
- 必填字段、类型、枚举值等约束

---

## 九、关键代码参考

| 模块 | 文件路径 | 主要职责 |
|------|---------|---------|
| 实体服务 | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | 实体/字段 CRUD、关联处理 |
| 关联服务 | [relation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/relation/relation.service.ts) | 资源关联、级联构建 |
| 构建服务 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/build/build.service.ts) | 构建流程、DSG 数据收集 |
| 服务模板 | [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | 模板导入、升级 |
| 自定义属性 | [customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts) | 自定义属性校验 |
| 字段工具 | [field.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/field/field.ts) | 字段类型判断 |
| 实体工具 | [entity-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/libs/util/dsg-utils/src/lib/entity-util.ts) | 默认操作生成 |
| Prisma 生成 | [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) | Prisma 字段生成 |
| DTO 生成 | [create-field-class-property.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/server/resource/dto/create-field-class-property.ts) | DTO 属性生成 |
| 上下文准备 | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/data-service-generator/src/prepare-context.ts) | 关联字段解析 |
| Schema 校验 | [jsonSchemaValidation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/services/jsonSchemaValidation.service.ts) | JSON Schema 校验 |
| 常量定义 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/22-amplication/packages/amplication-server/src/core/entity/constants.ts) | 默认属性、系统字段 |
