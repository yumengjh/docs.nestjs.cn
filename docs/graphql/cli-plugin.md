## CLI 插件

> **警告** 本章仅适用于代码优先（code first）方法。

TypeScript 的元数据反射系统存在若干限制，例如无法确定类包含哪些属性，或者识别某个属性是可选的还是必需的。不过，其中部分限制可以在编译时得到解决。Nest 提供了一个插件来增强 TypeScript 编译过程，从而减少所需的样板代码量。

> **提示** 此插件为**可选项**。如果你愿意，可以手动声明所有装饰器，或者仅在需要的地方声明特定装饰器。

#### 概述

GraphQL 插件将自动：

- 为所有输入对象、对象类型及参数类的属性添加 `@Field` 注解，除非使用了 `@HideField`
- 根据问号设置 `nullable` 属性（例如 `name?: string` 会设置为 `nullable: true`）
- 根据类型设置 `type` 属性（同时支持数组类型）
- 根据注释生成属性描述（当 `introspectComments` 设置为 `true` 时）

请注意，您的文件名**必须包含**以下后缀之一才能被插件分析：`['.input.ts', '.args.ts', '.entity.ts', '.model.ts']`（例如 `author.entity.ts`）。如果使用不同后缀，您可以通过指定 `typeFileNameSuffix` 选项来调整插件行为（见下文）。

根据目前所学，您需要重复大量代码来让包知道您的类型应如何在 GraphQL 中声明。例如，您可以如下定义一个简单的 `Author` 类：

```typescript title="authors/models/author.model.ts"
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

虽然对于中型项目来说这不是大问题，但一旦拥有大量类时，代码会变得冗长且难以维护。

通过启用 GraphQL 插件，上述类定义可以简化为：

```typescript title="authors/models/author.model.ts"
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;
  firstName?: string;
  lastName?: string;
  posts: Post[];
}
}
```

该插件会基于**抽象语法树**动态添加适当的装饰器。因此，您无需再为散落在代码各处的 `@Field` 装饰器而烦恼。

> info **注意** 插件会自动生成所有缺失的 GraphQL 属性，但如需覆盖它们，只需通过 `@Field()` 显式设置即可。

#### 注释内省

启用注释自省功能后，CLI 插件将根据注释自动生成字段描述。

例如，给定一个示例属性 `roles`：

```typescript
/**
 * A list of user's roles
 */
@Field(() => [String], {
  description: `A list of user's roles`
})
roles: string[];
```

您必须复制描述值。启用 `introspectComments` 后，CLI 插件可提取这些注释并自动为属性提供描述。现在，上述字段可简化为以下声明方式：

```typescript
/**
 * A list of user's roles
 */
roles: string[];
```

#### 使用 CLI 插件

要启用该插件，请打开 `nest-cli.json` 文件（如果使用 [Nest CLI](/cli/overview)）并添加以下 `plugins` 配置：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql"]
  }
}
```

您可以使用 `options` 属性来自定义插件的行为。

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/graphql",
        "options": {
          "typeFileNameSuffix": [".input.ts", ".args.ts"],
          "introspectComments": true
        }
      }
    ]
  }
}

```

`options` 属性必须满足以下接口：

```typescript
export interface PluginOptions {
  typeFileNameSuffix?: string[];
  introspectComments?: boolean;
}
```

| 选项               | 默认                                                   | 描述                                          |
| ------------------ | ------------------------------------------------------ | --------------------------------------------- |
| typeFileNameSuffix | \['.input.ts', '.args.ts', '.entity.ts', '.model.ts'\] | GraphQL 类型文件后缀                          |
| introspectComments | false                                                  | 如果设置为 true，插件将根据注释为属性生成描述 |

如果您不使用 CLI 而是使用自定义的 `webpack` 配置，可以将此插件与 `ts-loader` 结合使用：

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/graphql/plugin').before({}, program)]
}),
```

#### SWC 构建器

对于标准设置（非 monorepo），要在 SWC 构建器中使用 CLI 插件，您需要启用类型检查，具体方法如[此处](/recipes/swc#类型检查)所述。

```bash
$ nest start -b swc --type-check
```

对于 monorepo 设置，请按照[此处](/recipes/swc#monorepo-和-cli-插件)的说明操作。

```bash
$ npx ts-node src/generate-metadata.ts
```
# OR npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

现在，序列化的元数据文件必须通过 `GraphQLModule` 方法加载，如下所示：

```typescript
import metadata from './metadata'; // <-- file auto-generated by the "PluginMetadataGenerator"
```

GraphQLModule.forRoot<...>({
  ..., // other options
  metadata,
}),
```

#### 与 `ts-jest` 集成（端到端测试）

当启用此插件运行端到端测试时，可能会遇到模式编译问题。例如，最常见的错误之一是：

```json
Object type <name> must define one or more fields.
```

这是因为 `jest` 配置中没有在任何地方导入 `@nestjs/graphql/plugin` 插件。

要解决此问题，请在端到端测试目录中创建以下文件：

```javascript
const transformer = require('@nestjs/graphql/plugin');

module.exports.name = 'nestjs-graphql-transformer';
// you should change the version number anytime you change the configuration below - otherwise, jest will not detect changes
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/graphql/plugin options (can be empty)
    },
    cs.program // "cs.tsCompiler.program" for older versions of Jest (<= v27)
  );
};
```

完成上述设置后，在你的 `jest` 配置文件中导入 AST 转换器。默认情况下（在初始应用程序中），端到端测试配置文件位于 `test` 文件夹下，名为 `jest-e2e.json`。

```json
{
  ... // other configuration
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<path to the file created above>"]
      }
    }
  }
}
```

如果你使用 `jest@^29`，请采用下面的代码片段，因为之前的实现方式已被弃用。

```json
{
  ... // other configuration
  "transform": {
    "^.+\\.(t|j)s$": [
      "ts-jest",
      {
        "astTransformers": {
          "before": ["<path to the file created above>"]
        }
      }
    ]
  }
}
```
