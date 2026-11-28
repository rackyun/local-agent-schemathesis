---
description: "Post-run test analysis report requirements and template for Schemathesis results"
connections: [case-volume, agent-prompts, ci-examples]
---

# Test Analysis Report

测试完成后，本地 Agent **必须**输出一份结构化分析报告。不要只丢原始日志。

## 必须包含的章节

1. **概览**
   - 运行时间、总用例数、通过/失败/跳过数量
   - 覆盖的操作数 / 总操作数
   - 使用的 max-examples、phases、是否启用 stateful

2. **失败分析**（按优先级排序）
   - 失败接口 + 方法
   - 失败类型（schema 违反、业务逻辑、5xx、超时、认证等）
   - 根因判断
   - 复现 curl
   - 建议修复方向

3. **Schema 改进建议**
   - 缺失的约束、错误响应、examples
   - 哪些改进能让 Schemathesis 自动覆盖更多场景

4. **自定义 Check 建议**
   - 哪些业务规则 schema 表达不了，建议写成 `@schemathesis.check`

5. **回归测试产出**
   - 已生成或建议生成的 pytest 回归用例列表

6. **下一步行动**
   - 优先级排序的待办（修 schema / 加 check / 提 examples / 过滤噪声）

7. **多接口场景**（运行场景时填写）
   - 记录场景名、每步通过 / 失败 / 跳过、变量来源和去向，以及清理结果。
   - 用状态码、断言或异常说明首个失败步骤。证据不足时将根因标记为待确认。

## 报告模板（可直接要求 Agent 填充）

```markdown
# Schemathesis 测试分析报告

**运行时间**：YYYY-MM-DD HH:MM  
**配置**：max-examples=XX, phases=..., stateful=on/off  
**结果**：通过 X / 失败 Y / 总计 Z

## 失败摘要

| 接口 | 方法 | 失败类型 | 严重程度 | 简要原因 |
|------|------|----------|----------|----------|
| /users/{id} | GET | 业务逻辑 | 高 | 返回了其他用户数据 |

## 详细失败

### 1. GET /users/{id} — 数据越权

- **复现 curl**：
  ```bash
  curl ...
  ```
- **根因**：...
- **建议**：添加自定义 check `check_user_isolation`

## Schema 改进建议

- ...

## 建议新增的自定义 Check

- ...

## 已生成的回归测试

- `tests/test_regression_user_isolation.py`

## 多接口场景（运行场景时填写）

**场景名**：create_then_get_order

示例数据。按确认草稿列出全部必测 case ID，包括没有运行结果的用例。

| Case ID | 必测分支 | 结果 | 证据 |
|---|---|---|---|
| order-normal | 正常建单查单 | 失败 | 建单响应缺少 id |
| order-max-quantity | 合法数量上限 | 通过 | 查到同一订单，清理成功 |
| order-invalid-quantity | 非法数量被拒 | 跳过 | pytest 记录跳过原因 |
| order-deleted-not-found | 删除后查不到 | 未执行 | 此次运行未选中该用例 |

通过表示业务断言和必要清理均通过。断言或清理失败时记为失败。pytest 明确跳过时写明原因。未选中、未生成或无运行结果时记为未执行，并注明原因。此表记录业务分支的执行结果，pytest 用例数不能用来计算服务端代码覆盖率。

**失败用例**：order-normal

| 步骤 | 结果 | 传参或提取 | 实际结果 |
|---|---|---|---|
| create_order | 失败 | 从响应的 `$.id` 提取 `order_id`，提取失败 | 201，响应缺少 id |
| get_order | 跳过 | 需要 `order_id` | 未发请求 |

- **失败根因**：建单响应缺少约定字段 `id`，查单无法构造路径。服务端为何漏字段待确认。
- **清理结果**：未取得资源标识，未执行删除。

## 下一步

1. ...
2. ...
```

## 给 Agent 的指令（可追加到现有 prompt）

```text
测试完成后，请严格按照 references/test-report.md 中的模板输出完整分析报告。
不要只返回原始日志。必须包含失败分析、schema 改进建议、自定义 check 建议和下一步行动。运行多接口场景时填写场景章节。
```
