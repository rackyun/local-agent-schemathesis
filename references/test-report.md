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

从确认草稿冻结去重后的已确认必测逻辑 ID 集合，令其数量为 N。待确认和不适用分别列出，不计入 N；确认范围有变更时另记基线。无论生成或执行时遗漏多少项，分母都用同一个 N。每列覆盖率 = 满足该列条件的逻辑 ID 数 ÷ N × 100%；N 为 0 时写 N/A。

- **计划**要求该项的每个必要见证都有可执行用例和绑定断言。
- **已生成**要求每个必要见证都能定位到生成的 pytest 用例与具体断言。
- **已执行**要求每个必要见证的断言都实际求值，断言失败也算已执行。跳过、未选、漏生成或因前置失败未到达的断言均不计。若只到达部分见证，记录已到达及其结果，并列出余下缺口，该逻辑项暂不计入已执行。
- **已通过**要求每个必要见证的断言都实际通过。任一必要断言失败，保留失败证据，即使其他见证通过或未到达也一样。

生成后先填四列，尚未运行时执行和通过数为 0（N 为 0 时百分比仍为 N/A）。运行后据实更新。每列列出缺口 ID，待确认项给出问题，不适用项给出理由。只要有待确认、遗漏、执行缺口或必要断言失败，就不能报总体达标。N 为 0 时也不报满覆盖。这是可观察的业务逻辑覆盖；pytest 条数不能用来计算服务端代码覆盖率。

以下为八项 [模拟契约切片](scenario-patterns.md#覆盖矩阵) 的虚构运行记录。草稿八项均有计划，生成时漏了末项断言。本次模拟假设还存在待确认的恢复规则。

| 指标 | 数量及百分比 | 缺口逻辑 ID |
|---|---|---|
| 计划 | 8/8（100%） | 无 |
| 已生成 | 7/8（87.5%） | lookup-after-delete |
| 已执行 | 4/8（50%） | lookup-found、lookup-same-id、reject-zero、lookup-after-delete |
| 已通过 | 3/8（37.5%） | create-id、lookup-found、lookup-same-id、reject-zero、lookup-after-delete |

| 逻辑 ID | Case ID | 断言结果 | 证据或原因 |
|---|---|---|---|
| create-normal | order-normal | 通过 | POST 返回 201 |
| create-id | order-normal | 失败 | 201 响应缺少 ID |
| lookup-found | order-normal | 未执行 | 因 ID 断言失败未发 GET |
| lookup-same-id | order-normal | 未执行 | 因 ID 断言失败未发 GET |
| accept-max | order-max-quantity | 通过 | 数量 10 的 POST 返回 201 |
| reject-zero | order-invalid-quantity | 跳过 | pytest 标记模拟拒绝夹具不可用 |
| delete-created | order-deleted-not-found | 通过 | DELETE 返回 204 |
| lookup-after-delete | order-deleted-not-found | 未执行 | 生成阶段漏掉删除后查询断言 |

- **待确认**：已删除订单是否允许恢复，需提供契约。**不适用**：此模拟切片无事务回滚契约。
- **总体判定**：未达标，存在待确认、生成及执行缺口。
- **用例结果及清理**：逐个 case ID 保留通过、失败、跳过或未执行及原因。只有该用例的业务断言和必要清理都通过，才可标记用例通过。清理失败单独记录，且应阻止报告总体达标；已有逻辑断言的执行证据及结果仍保留。此例中，正常建单用例失败且未取得清理所需 ID，数量上限用例通过且清理成功，非法数量用例跳过，删除用例已运行并完成主动删除，但因漏断言不可认定其业务验证通过。

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
