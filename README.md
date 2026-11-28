# local-agent-schemathesis

用 Schemathesis 从 OpenAPI 自动生成 API 测试，再用本地 Agent（如 Claude Code、Cursor 或 Continue）完善接口规范、编写业务断言、分析失败，并将用户描述的业务链路转成 pytest 测试。

| 测试目标 | 用法 |
| --- | --- |
| 检查接口是否符合 OpenAPI | Schemathesis 运行规范中的示例，并生成边界用例和正负例。 |
| 根据接口依赖自动串联请求 | OpenAPI 有可用的 `links`（接口间参数映射）时，使用 Schemathesis 的 `stateful` 阶段。 |
| 验证指定的多接口业务链路 | 用户用自然语言描述场景，确认 YAML 草稿后，由 Agent 生成独立的 pytest 测试。 |

## 快速开始

1. 准备 `uv`、待测项目的 OpenAPI 文件和一个已启动的测试环境。在下面的命令中换成实际的文件路径和服务地址。Schemathesis 会发出真实请求，请使用可写入测试数据的环境。
2. 在待测项目中运行下面的命令。需要 Bearer 鉴权时，先在环境中设置测试令牌 `API_TOKEN`，再给命令加上 `-H "Authorization: Bearer ${API_TOKEN}"`。

   ```bash
   export API_BASE_URL=http://127.0.0.1:8080
   uvx schemathesis@4.34.3 run openapi.yaml --url "$API_BASE_URL" \
     --phases=examples,coverage,fuzzing --max-examples=10 \
     --report=html --report-html-path=schemathesis-report/html
   ```

   `--max-examples=10` 在这条命令中限制每个 API 操作在 `fuzzing` 阶段生成的用例数。规范示例和边界覆盖阶段会另行生成用例。

3. 打开 `schemathesis-report/html/index.html` 查看结果。在 Agent 中打开待测项目，请它读取本仓库的 [SKILL.md](SKILL.md) 和相关指南，并提供 OpenAPI 文件及已隐藏凭证的失败信息。Agent 可以协助区分规范问题、接口实现问题和测试数据问题。报告怎么写可参考 [测试分析报告](references/test-report.md)。

## 从业务描述生成多接口测试

1. 把 OpenAPI 文件和业务场景交给 Agent。可以直接输入以下示例，按待测 API 修改接口约定。

   > 请先根据 OpenAPI 起草「创建订单后查询同一笔订单」的测试场景。创建成功后保存响应中的订单 ID，查询时使用这个 ID，并验证查到的订单 ID 与创建时一致。测试结束后删除本次创建的订单。请先给我看 YAML 草稿。

2. Agent 核对 OpenAPI 中的操作，有缺失信息时向你确认，然后给出 `scenarios/create_then_get_order.scenario.yaml` 草稿。检查操作顺序、订单 ID 的来源和去向、各步预期结果以及清理方式，确认当前版本后，Agent 才生成 `tests/test_scenario_create_then_get_order.py`。
3. 在待测项目中设置测试服务地址 `API_BASE_URL` 和 Bearer 测试令牌 `API_TOKEN`，然后单独运行已经生成的 Python 测试。YAML 用于记录和确认场景，pytest 执行对应的 Python 文件。

   ```bash
   uvx --with requests pytest -q tests/test_scenario_create_then_get_order.py
   ```

完整的 YAML、pytest 和失败归因示例见 [场景化多接口测试](references/scenario-patterns.md)。需要可复制的完整提示词时，参见 [Agent 提示词第 6 节](references/agent-prompts.md#6-generate-a-multi-operation-scenario)。

## 参考资料

| 想做什么 | 文档 |
| --- | --- |
| 审查 OpenAPI、运行测试或重放失败 | [技能工作流](SKILL.md) |
| 配置生成量和单个操作的覆盖范围 | [用例数量与配置示例](references/case-volume.md) |
| 为接口添加业务断言 | [自定义检查与 hooks 示例](references/hooks-examples.md) |
| 配置 PR 快测和夜间测试 | [GitHub Actions 模板](references/ci-examples.md) |
