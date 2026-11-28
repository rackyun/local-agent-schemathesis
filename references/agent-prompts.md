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

## 6. Generate a multi-operation scenario

```text
输入是用户用自然语言描述的 API 链路，例如“建单后，用返回的 id 查单，并确认查到的是同一笔订单”。结合实际 OpenAPI 列出相关操作。

1. 盘点逻辑，再起草。根据 OpenAPI 和用户已确认的业务规则，逐项检查正常链路、已知规则的边界内外、每条拒绝规则、允许及禁止的状态迁移、跨步骤数据传递与失败后的状态。契约明确适用时，加入可区分的重复操作、回滚和条件组合。不适用写理由，信息不足单列「待确认」并询问，勿猜测状态码或业务规则。
   按 references/scenario-patterns.md 起草 scenarios/<名>.scenario.yaml。每项必测逻辑写稳定逻辑 ID、规则出处、触发、预期、具体断言和关联 case ID；逻辑与用例可多对多。可独立判定的条件各用逻辑 ID。同一逻辑若需要多个见证，明确列出所有必要的用例与断言。草稿同时保留操作顺序、请求数据、变量来源和去向及清理。相同逻辑和预期的重复用例或随机值不增加覆盖；不设用例上限，条件组合按契约选择，无需枚举笛卡尔积。检查计划是否覆盖 100% 已确认必测逻辑，每项都须有可执行用例及绑定断言。补齐缺口或列出阻塞及其 ID。请用户确认当前草稿，确认前不生成 pytest；草稿修改后重新确认。

2. 确认后生成 tests/test_scenario_<名>.py，保留逻辑 ID 到具体断言及 pytest case ID 的映射。相同步骤和断言仅输入不同的用例参数化，执行路径不同的分支独立编写。隔离各用例数据，使用 requests 显式传递变量并断言状态码与业务字段。拒绝后不继续执行依赖成功的步骤；断言失败后仍清理本次创建且尚未删除的资源，不重复删除，单独记录清理。鉴权从环境读取，提供单独运行命令。核对实际生成的用例和断言，列出缺口。

3. 生成后和运行后均按 references/test-report.md 输出四项指标：计划、已生成、已执行和已通过。每项以满足条件的逻辑 ID 去重计数，除以确认草稿中固定的已确认必测逻辑总数；分母为零写 N/A。逐项列出缺口 ID、待确认和不适用。仅当所有必要见证的断言实际求值才计为已执行，失败也计执行；全部必要断言通过才计为通过。用例启动、跳过或前置失败导致断言未到达均不算该断言已执行。记录实际通过、失败及证据、跳过、未执行及原因，便于发现部分执行。存在待确认或遗漏时不可报总体达标。指标衡量可观察的业务逻辑，不能用于推算服务端代码覆盖率。
```
