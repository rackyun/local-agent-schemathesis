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

## 下一步

1. ...
2. ...
```

## 给 Agent 的指令（可追加到现有 prompt）

```text
测试完成后，请严格按照 references/test-report.md 中的模板输出完整分析报告。
不要只返回原始日志。必须包含失败分析、schema 改进建议、自定义 check 建议和下一步行动。
```
