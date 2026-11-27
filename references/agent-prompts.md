---
description: "Ready-to-use prompts for instructing a local agent to generate Schemathesis checks, regression tests, and schema improvements"
connections: [hooks-examples, ci-examples]
---

# Agent Prompts for Schemathesis

## 1. Generate a custom check

```text
你是 API 测试专家。请根据下面的业务规则，为 Schemathesis 编写一个自定义 check。

业务规则：
[在这里写规则，例如：GET /users/{user_id} 必须返回与路径参数相同的 id，禁止越权访问]

要求：
- 使用 @schemathesis.check 装饰器
- 函数签名必须是 (ctx, response, case)
- 只在相关接口和状态码下检查
- 失败时抛出 AssertionError，信息要清晰
- 代码放在 hooks.py 中
- 在 schemathesis.toml 的顶层用 hooks = ["./hooks.py"] 加载。全局选项 --config-file 必须放在 run 前面，示例命令如下。
  uvx schemathesis@4.34.3 --config-file schemathesis.toml run openapi.yaml --url <base-url>

请直接输出完整可运行的 Python 代码。
```

## 2. Turn a failure into a regression test

```text
下面是一次 Schemathesis 失败的信息（包含 curl 或响应）：

[优先粘贴 --report=pytest 生成的复现代码，也可提供失败详情 / curl / 响应片段]

请完成：
1. 分析根因（是 schema 问题、业务逻辑问题，还是数据问题）
2. 已有 pytest 复现代码时，补充修复后的预期断言；否则生成可独立运行的 pytest 回归测试（使用 requests 或 httpx）
3. 测试名要能看出是哪个接口、什么场景
4. 断言要明确
5. 如果需要，同时给出改进 OpenAPI 的建议
```

## 3. Improve OpenAPI schema

```text
请审查下面的 OpenAPI 文件，找出会影响 Schemathesis 测试质量的问题：

[粘贴 openapi.yaml 或关键路径]

重点检查：
- 缺少 required / min / max / pattern / format
- 缺少 4xx 错误响应定义
- 缺少 examples
- 路径参数与响应不一致
- 没有 operationId

请给出：
1. 问题列表（按优先级）
2. 修改后的完整片段（YAML）
3. 哪些问题修好后，Schemathesis 能自动覆盖更多边界情况
```

## 4. Generate hooks.py + schemathesis.toml

```text
请为以下 API 生成一套可直接使用的 Schemathesis 配置：

API 描述：
[简单描述接口、认证方式、关键业务规则]

输出内容：
1. schemathesis.toml（包含认证、max-examples、phases）
2. hooks.py（至少包含 1-2 个业务规则 check）
3. 运行命令示例
4. 建议的 PR 浅测 vs Nightly 深测参数
```

## 5. Persistent agent instructions (for AGENTS.md / CLAUDE.md)

```text
你是本项目的 API 测试助手，使用 Schemathesis + 本地 Agent 工作流。

原则：
- 优先让 Schemathesis 从 OpenAPI 自动生成大量用例
- 业务规则必须写成 @schemathesis.check，不要手写大量重复测试
- 每次失败都要产出：curl 复现 + 回归测试 + Schema 改进建议
- PR 用浅测（max-examples=50），Nightly 用深测（max-examples=300+ 并开启 stateful）

当我要求生成测试时，默认输出：
1. hooks.py 代码
2. 加载命令
3. 对应的 schemathesis.toml 片段
```
