---
description: "Recommended Schemathesis case generation volumes by scenario (max-examples, time budget, coverage goals)"
connections: [ci-examples, agent-prompts]
---

# Case Generation Volume Recommendations

> **重要说明**：`--max-examples`（或 `generation.max-examples`）是**按每个操作（operation）计算的**，不是整个 API 的总用例数。
>
> - 一个 operation = 一个路径 + 一个 HTTP 方法（如 `GET /users/{id}`、`POST /orders` 各算一个）
> - `--max-examples=100` 为 fuzzing 阶段的每个操作设置 100 个 Hypothesis 样例的预算
> - fuzzing 用例量级 ≈ 操作数 × max-examples。其他阶段和失败缩减的请求另计。

下面是按场景的建议值（均为**每个操作**的数量）。

## 快速参考

| 场景 | 每个操作 max-examples | 预计耗时（中等规模 API） | 说明 |
|------|-----------------------|--------------------------|------|
| 本地快速冒烟 | 20–50 | < 1 分钟 | 验证链路是否通、schema 是否基本可用 |
| PR / 预合并检查 | 50–100 | 2–5 分钟 | 平衡速度与覆盖，适合 CI 门禁 |
| 夜间 / 深度测试 | 200–500 | 15–60 分钟 | 配合 stateful + fuzzing，追求更高覆盖 |
| 安全 / 边界专项 | 500–1000+ | 视接口而定 | 对关键接口单独提高，其他保持较低 |
| 新接口首次接入 | 100–200 | — | 先跑一轮，再根据失败调整 |

### 总量估算示例

| API 规模 | 操作数 | 每个操作 examples | 大致总用例数 |
|----------|--------|-------------------|--------------|
| 小型 | 15 | 50 | ~750 |
| 中型 | 60 | 50 | ~3000 |
| 大型 | 150 | 30（过滤后） | 视过滤范围而定 |

## 按 API 规模调整

- **小型 API**（< 20 个操作）：PR 可用 100，Nightly 可用 400–600
- **中型 API**（20–100 个操作）：PR 保持 50–80，Nightly 200–300
- **大型 API**（> 100 个操作）：
  - 优先用路径/标签过滤，只测变更相关接口
  - 关键路径单独提高 max-examples
  - 其余接口保持较低值，避免 CI 过长

## 实用原则

1. **时间优先于数量**：先确定 CI 可接受的时间，再反推 max-examples。
2. **渐进增加**：从 50 开始，失败稳定后再逐步提高。
3. **分阶段**：
   - 第一阶段：只开 fuzzing，较低 examples
   - 第二阶段：开启 stateful
   - 第三阶段：对高风险接口提高 examples
4. **配合其他手段**：生成量再大，也无法替代好的 schema 约束和自定义 check。业务规则应写在 hooks 里，而不是单纯堆 examples。

## 配置示例

```toml
# schemathesis.toml
generation.max-examples = 100          # 默认（每个操作）

[[operations]]
include-name = "GET /users/{id}"
generation.max-examples = 300          # 关键接口单独提高（每个操作）
```

```bash
# 使用配置，保留按操作设置的预算
uvx schemathesis@4.34.3 --config-file schemathesis.toml run openapi.yaml \
  --url http://localhost:8080 --phases=fuzzing
```

`include-name` 按 `HTTP 方法 + 路径模板` 匹配单个操作。需要统一覆盖各操作的预算时，在 `run` 参数中传入 `--max-examples=50`。

## 本机验证（4.34.3）

对照随包提供的 `schemathesis/config/schema.json`，确认顶层 `generation.max-examples`、`[[operations]]` 和操作内覆盖均受支持。按上面的结构保存 `verify.toml`，将默认值设为 2，`GET /users/{id}` 设为 3。

```bash
uvx schemathesis@4.34.3 --config-file verify.toml run openapi.json \
  --url "$BASE_URL" -H "Authorization: Bearer local-demo" \
  --mode=positive --phases=fuzzing
```

针对两个本地操作实跑，CLI 输出 `5 generated, 5 passed`。分别读取生效配置，确认 `GET /health` 使用默认值 2，`GET /users/{id}` 使用覆盖值 3。

## 与 CI 双轨的对应关系

- **PR**：每个操作 `max-examples=50`（或 80），`--phases=examples,coverage,fuzzing`，目标 < 5 分钟
- **Nightly**：每个操作 `max-examples=300+`，`--phases=examples,coverage,fuzzing,stateful`，可接受更长时间
