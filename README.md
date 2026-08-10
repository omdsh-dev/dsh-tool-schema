# dsh-tool-schema

DSH JSON Schema 验证工具插件 —— 验证数据、列出失败路径、解释 schema 约束、安全应用 default。零网络、零动态代码执行。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 动机

Agent 需要验证任意 JSON 数据是否符合 schema（API 响应结构、插件 manifest、配置文件、会话事件），并定位失败路径。DSH 内置的 `defineTool` 参数 DSL 是受限的作者 DSL，不是面向任意用户 schema 的通用验证服务；`dsh-tool-json` 提供查询能力但不验证结构。

本插件提供独立的纯函数 JSON Schema 验证内核，支持常用关键字与本地 `$ref`，绝不静默忽略不支持的 schema 关键字。

## 用途

| action | 功能 |
|---|---|
| `validate` | 验证 instance，返回 verdict、path-qualified 错误（RFC 6901 JSON Pointer）与 schema 问题 |
| `paths` | 只返回失败路径与关键字摘要 |
| `explain` | 静态解释 schema 的约束树（有限节点列表，非自然语言长文） |
| `normalize` | 深拷贝数据并对缺失字段应用显式 `default` 后验证（不修改输入、不强制类型） |

支持：`type`/`enum`/`const`、对象（required/properties/additionalProperties/minProperties/maxProperties）、数组（items/minItems/maxItems/uniqueItems）、字符串（minLength/maxLength/pattern）、数值（minimum/maximum/multipleOf）、组合（allOf/anyOf/oneOf/not）、本地 `$ref`（`#/$defs/...`）。

安全边界：不支持的 schema 关键字绝不静默忽略（`unsupported-keyword` 报告）；不执行任何代码；pattern 验证有独立 worker 硬超时（ReDoS 防护）；数据/schema 大小、嵌套深度、节点数、错误数、`$ref` 链长均有上限。

> 详细设计（验证语义、错误格式、资源限制、测试计划）见 `DESIGN.md`，将在完成开发后随代码上传。
