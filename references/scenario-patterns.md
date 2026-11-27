---
description: "Draft and confirm multi-operation API scenarios, then generate pytest tests"
connections: [agent-prompts, test-report]
---

# 场景化多接口测试

1. Agent 从 OpenAPI 列出与场景相关的操作。用户用自然语言描述链路、步骤间传参和预期结果。缺少操作、数据来源、断言或清理约定时，先向用户确认缺失信息。
2. Agent 输出 `scenarios/<name>.scenario.yaml` 草稿，请用户确认操作顺序、传参、断言和清理。用户修改后重新确认。用户确认当前草稿后，再生成 `tests/test_scenario_<name>.py`。

YAML 用于确认场景。pytest 执行根据已确认草稿生成的 Python 测试。

## 最小示例

示例 API 约定 `POST /orders` 创建测试订单、`GET /orders/{order_id}` 查询、`DELETE /orders/{order_id}` 清理。将接口、状态码和字段改为实际 OpenAPI 中已确认的约定。

```yaml
# scenarios/create_then_get_order.scenario.yaml
name: create_then_get_order
steps:
  - id: create_order
    method: POST
    path: /orders
    json:
      sku: demo-item
      quantity: 1
    save:
      order_id: "$.id"
  - id: get_order
    method: GET
    path: '/orders/${order_id}'
asserts:
  - step: create_order
    status: 201
    json:
      "$.id":
        type: non_empty_string
  - step: get_order
    status: 200
    json:
      "$.id":
        equals: '${order_id}'
cleanup:
  - method: DELETE
    path: '/orders/${order_id}'
    when_saved: order_id
    status: 204
```

- `name` 使用适合文件名的简短标识。按 `steps` 的顺序请求各操作。
- `save` 将当前响应的字段保存为变量，`$.id` 表示 JSON 顶层的 `id`。`${order_id}` 引用已经保存的变量。
- `asserts` 按步骤 ID 关联断言。本步骤断言失败后，后续业务步骤跳过。
- `cleanup` 只清理本次测试创建的资源。在取得订单 ID 后，即使查询或断言失败也执行清理，并单独记录清理结果。

用户确认草稿后生成的示例如下。

```python
# tests/test_scenario_create_then_get_order.py
import os
from urllib.parse import quote

import pytest
import requests


@pytest.fixture
def orders_api():
    base_url = os.environ["API_BASE_URL"].rstrip("/")
    created_order_urls = []
    with requests.Session() as session:
        session.headers["Authorization"] = f"Bearer {os.environ['API_TOKEN']}"
        yield session, base_url, created_order_urls

        for order_url in created_order_urls:
            response = session.delete(order_url, timeout=10)
            assert response.status_code == 204, (
                f"cleanup_order: expected 204, got {response.status_code}"
            )


def test_create_then_get_order(orders_api):
    session, base_url, created_order_urls = orders_api
    created = session.post(
        f"{base_url}/orders",
        json={"sku": "demo-item", "quantity": 1},
        timeout=10,
    )
    assert created.status_code == 201, (
        f"create_order: expected 201, got {created.status_code}"
    )
    order_id = created.json().get("id")
    assert isinstance(order_id, str) and order_id, "create_order: missing id"
    order_url = f"{base_url}/orders/{quote(order_id, safe='')}"
    created_order_urls.append(order_url)

    fetched = session.get(order_url, timeout=10)
    assert fetched.status_code == 200, (
        f"get_order: expected 200, got {fetched.status_code}"
    )
    assert fetched.json().get("id") == order_id, "get_order: id mismatch"
```

## 运行与鉴权

在测试环境设置 `API_BASE_URL` 和 `API_TOKEN`，令牌从环境传入。pytest 使用 `requests` 执行已确认的场景。Schemathesis 4.34.3 继续用 fuzzing 生成单操作用例，OpenAPI 含可用 links 时也可运行 stateful。

```bash
pytest -q tests/test_scenario_create_then_get_order.py

uvx schemathesis@4.34.3 run openapi.yaml --url "$API_BASE_URL" \
  -H "Authorization: Bearer ${API_TOKEN}" --phases=fuzzing,stateful
```

Schemathesis 保存的 crash 文件会把 Authorization 写成 `[Filtered]`。用 `replay` 重放已保存的 crash 文件时，重传原测试的凭证。

```bash
uvx schemathesis@4.34.3 replay --keep "$CRASH_FILE" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## 失败归因

按 [[test-report]] 记录首个失败步骤、实际响应或异常、变量来源和去向、后续跳过步骤及清理结果。比如建单返回 201 却缺少 `$.id`，将失败归于建单响应，查单记为跳过。若建单成功保存 ID，且查单使用同一 ID 仍返回 404，记录查单的状态码和传参证据，根因待查。报告中的请求和响应应隐藏凭证。
