---
description: "Recommended Schemathesis case generation volumes by scenario (max-examples, time budget, coverage goals)"
connections: [ci-examples, agent-prompts]
---

# Case Generation Volume Recommendations

Schemathesis 的 `--max-examples`（或配置里的 `generation.max-examples`）控制每个操作生成的用例数量。下面是按场景的建议值。

## 快速参考

| 场景 | max-examples | 预计耗时（中等规模 API） | 说明 |
|------|--------------|--------------------------|------|
| 本地快速冒烟 | 20–50 | < 1 分钟 | 验证链路是否通、schema 是否基本可用 |
| PR / 预合并检查 | 50–100 | 2–5 分钟 | 平衡速度与覆盖，适合 CI 门禁 |
| 夜间 / 深度测试 | 200–500 | 15–60 分钟 | 配合 stateful + fuzzing，追求更高覆盖 |
| 安全 / 边界专项 | 500–1000+ | 视接口而定 | 对关键接口单独提高，其他保持较低 |
| 新接口首次接入 | 100–200 | — | 先跑一轮，再根据失败调整 |

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
generation.max-examples = 100          # 默认

[[operations]]
include-path = "/users/"
generation.max-examples = 300          # 关键接口单独提高
```

```bash
# 命令行覆盖
schemathesis run openapi.yaml --url http://localhost:8080 --max-examples=50
```

## 与 CI 双轨的对应关系

- **PR**：`max-examples=50`（或 80），`--phases=fuzzing`，目标 < 5 分钟
- **Nightly**：`max-examples=300+`，`--phases=fuzzing,stateful`，可接受更长时间
