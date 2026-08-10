# dsh-tool-schema

DSH JSON Schema 验证工具插件 —— 验证数据、列出失败路径、解释 schema 约束、安全应用 default。零网络、零动态代码执行。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 动机

Agent 需要验证任意 JSON 数据是否符合 schema（API 响应结构、插件 manifest、配置文件、会话事件），并定位失败路径。DSH 内置的 `defineTool` 参数 DSL 是受限的作者 DSL，不是面向任意用户 schema 的通用验证服务；`dsh-tool-json` 提供查询能力但不验证结构。

本插件提供独立的纯函数 JSON Schema 验证内核，支持常用关键字与本地 `$ref`，**绝不静默忽略不支持的 schema 关键字**。

## 安全模型

1. **零动态执行**：验证内核是纯数据遍历，不构造 `RegExp`（pattern 在独立 worker 内执行）、不 `eval`、不访问网络、不读文件；
2. **不支持关键字绝不静默忽略**：报告 `unsupported-keyword` schema issue；`strictSchema=true`（默认）直接失败，`strictSchema=false` 验证已支持子集并返回 `valid:null` / `complete:false`；
3. **ReDoS 防线**：所有 `pattern` 校验在可终止的 worker 线程内共享 1,000ms 硬预算，超时 `terminate()` 并报错——灾难性回溯不能阻塞宿主进程；pattern ≤ 16 KiB、每 schema ≤ 100 个；
4. **原型污染防护**：所有对象访问用 `Object.hasOwn`，`__proto__` / `constructor` / `prototype` 只作为普通 JSON 键处理；
5. **资源上限**：data/schema 各 256 KiB、嵌套深度 64、schema 节点 10,000、遍历节点 100,000、错误 100（默认）/ 1,000（上限）、`$ref` 链 64、canonical 输出 1 MiB（超限截断并置 `truncated`）；
6. **`$ref` 安全性**：仅支持本地引用（`#` 与 `#/$defs/<token>`，RFC 6901 转义）；目标必须存在；环检测（schema-check 静态报告 + 验证期 `(schemaNode, instance)` 栈动态兜底）。

## 工具声明

注册 `schema` 工具（`@deepseek-ai/dsh-tool-schema`，row id `tool-schema`），统一输出 JSON 文本字符串。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `action` | string | ✅ | `validate` / `paths` / `explain` / `normalize` |
| `data` | json | | 待验证实例（validate/paths/normalize 必需；`null` 是合法数据） |
| `schema` | json | ✅ | JSON Schema（boolean 或 object；draft 2020-12 子集） |
| `strictSchema` | boolean | | 不支持关键字时失败。默认 `true` |
| `maxErrors` | integer | | 最大错误报告数。默认 100，范围 1..1,000 |

支持的关键字：`type`/`enum`/`const`、对象（required/properties/additionalProperties/minProperties/maxProperties）、数组（items/minItems/maxItems/uniqueItems）、字符串（minLength/maxLength/pattern）、数值（minimum/maximum/exclusive\*/multipleOf）、组合（allOf/anyOf/oneOf/not）、本地 `$ref`。

## Actions

| action | 功能 | 输出示例 |
|---|---|---|
| `validate` | verdict + RFC 6901 `instancePath`/`schemaPath` 错误（稳定排序）+ schema issues | `{"valid":true,"complete":true,"errors":[],"schemaIssues":[],"checkedNodes":3}` |
| `paths` | 只返回失败路径 + 关键字摘要 | `{"valid":false,"paths":[{"path":"/a","keywords":["type"]}],"errorCount":1}` |
| `explain` | 静态约束树节点序列（不执行验证） | `{"nodes":[{"kind":"type","value":"object"},...],"truncated":false}` |
| `normalize` | 深拷贝 + 仅对缺失字段应用显式 `default`，再完整验证；不修改输入、不强制类型 | `{"valid":true,"appliedDefaults":[{"path":"/b","value":1}],"warnings":[]}` |

## 示例

```
schema { action: "validate", data: { name: "x", age: 3 },
         schema: { type: "object", required: ["name"], properties: { name: { type: "string" }, age: { type: "integer", minimum: 0 } } } }
  → {"action":"validate","complete":true,"valid":true,"supportedSubsetValid":true,"errors":[],"schemaIssues":[],"checkedNodes":3,"truncated":false}

schema { action: "paths", data: { a: 1 }, schema: { type: "object", properties: { a: { type: "string" } } } }
  → {"action":"paths","valid":false,"paths":[{"path":"/a","keywords":["type"]}],"errorCount":1,"truncated":false}

schema { action: "normalize", data: { a: 1 }, schema: { type: "object", properties: { b: { type: "integer", default: 5 } } } }
  → {"action":"normalize","valid":true,"appliedDefaults":[{"path":"/b","value":5}],"warnings":[]}
```

> 详细设计（验证语义、错误格式、资源限制）见 `DESIGN.md`（随代码上传）。
