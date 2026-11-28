---
description: "Draft and confirm multi-operation API scenarios, then generate pytest tests"
connections: [agent-prompts, test-report]
---

# 场景化多接口测试

1. Agent 从 OpenAPI 列出与场景相关的操作。用户用自然语言描述链路、步骤间传参和预期结果。缺少操作、数据来源、断言或清理约定时，先向用户确认缺失信息。
2. Agent 输出 `scenarios/<name>.scenario.yaml` 草稿，请用户确认业务分支、操作顺序、传参、断言和清理。用户修改后重新确认。用户确认当前草稿后，再生成 `tests/test_scenario_<name>.py`。

YAML 用于确认场景。pytest 执行根据已确认草稿生成的 Python 测试。

## 业务分支清单

根据 OpenAPI 和已确认的业务规则，列出每条必测路径的触发条件、执行步骤和预期结果。检查正常路径、合法边界、拒绝路径及状态变化哪些适用于当前场景。缺少业务约定时在草稿中标记「待确认」，向用户补齐后重新确认。预期状态码和断言以已确认的契约为准，待确认的分支暂不生成可执行用例。

为每个分支分配唯一且稳定的 case ID。确认后的 ID 同时用于 pytest 参数 ID 和报告。相同步骤和断言仅输入不同的用例共用参数化函数，步骤或断言路径不同的分支用独立函数。

## 覆盖矩阵

先按 [场景提示词第 6 节](agent-prompts.md#6-generate-a-multi-operation-scenario) 盘点契约，再把 `coverage` 加到草稿中。它可附加于既有 YAML，旧单路径的顶层 `steps`、`asserts`、`cleanup` 保持原义。为旧单路径补算覆盖时可给整条链路分配一个 case ID，无需改写步骤。没有覆盖矩阵时先补盘点，勿用现有用例数推算覆盖。

以下为与下方四条最小用例对应的模拟契约切片，只列该切片确认的八个预期。若实际契约另有下限、上限外输入或失败后状态规则，应继续枚举并补用例。每条见证引用一个 case ID、步骤和该步骤的具体断言，可在实际 pytest 中按该用例和断言定位。同一用例可见证多项逻辑。一个逻辑 ID 下的见证全部必要。

```yaml
coverage:
  required:
    - id: create-normal
      source: 模拟建单约定
      trigger: 数量 2
      expected: 建单成功
      witnesses:
        - {case: order-normal, step: create_order, assertion: "status == 201"}
    - id: create-id
      source: 模拟建单响应约定
      trigger: 正常建单成功
      expected: 返回可传递的订单 ID
      witnesses:
        - {case: order-normal, step: create_order, assertion: "id is a non-empty string"}
    - id: lookup-found
      source: 模拟查询约定
      trigger: 查询正常创建的订单
      expected: 可查到订单
      witnesses:
        - {case: order-normal, step: get_order, assertion: "status == 200"}
    - id: lookup-same-id
      source: 模拟查询响应约定
      trigger: 用建单响应的 ID 查询
      expected: 查询响应对应同一订单
      witnesses:
        - {case: order-normal, step: get_order, assertion: "response.id == order_id"}
    - id: accept-max
      source: 模拟数量上限约定
      trigger: 数量 10
      expected: 上限允许创建
      witnesses:
        - {case: order-max-quantity, step: create_order, assertion: "status == 201"}
    - id: reject-zero
      source: 模拟拒绝约定
      trigger: 数量 0
      expected: 拒绝该数量
      witnesses:
        - {case: order-invalid-quantity, step: reject_order, assertion: "status == 422"}
    - id: delete-created
      source: 模拟删除约定
      trigger: 删除本次创建的订单
      expected: 删除成功
      witnesses:
        - {case: order-deleted-not-found, step: delete_order, assertion: "status == 204"}
    - id: lookup-after-delete
      source: 模拟删除后查询约定
      trigger: 查同一笔已删除订单
      expected: 已删除订单查不到
      witnesses:
        - {case: order-deleted-not-found, step: get_deleted_order, assertion: "status == 404"}
  pending: []
  not_applicable:
    - rule: 事务回滚
      reason: 模拟切片只有单资源操作，无适用的事务契约
```

草稿检查每项必要见证能否执行以及是否绑定具体断言。计划覆盖率须为 100%，否则补齐或列出阻塞 ID。存在未知规则时，在 `pending` 列出规则、缺少的依据及待确认问题（如删除后是否允许恢复），不要填猜测的预期或报总体达标。请用户确认当前草稿。确认后的必测逻辑总数是本轮固定分母，新增或变更规则后重新确认并另记统计基线。生成和运行均使用 [测试分析报告](test-report.md) 逐项核对。

## 订单示例

以下接口、数量和状态码均为本地模拟契约。数量允许 1 至 10。建单成功返回 201 和非空订单 ID，数量 0 返回 422 且不创建订单。查询存在的订单返回 200 和订单 ID，删除成功返回 204，查询已删除的同一订单返回 404。

```yaml
# scenarios/create_then_get_order.scenario.yaml
name: create_then_get_order
steps:
  - id: create_order
    method: POST
    path: /orders
    json:
      sku: demo-item
      quantity: '${quantity}'
    save:
      order_id: "$.id"
  - id: get_order
    method: GET
    path: '/orders/${order_id}'
  - id: reject_order
    method: POST
    path: /orders
    json:
      sku: demo-item
      quantity: '${quantity}'
    save_on_unexpected_creation:
      order_id: "$.id"
  - id: delete_order
    method: DELETE
    path: '/orders/${order_id}'
  - id: get_deleted_order
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
  - step: reject_order
    status: 422
  - step: delete_order
    status: 204
  - step: get_deleted_order
    status: 404
cleanup:
  - method: DELETE
    path: '/orders/${order_id}'
    when_saved: order_id
    unless_step_succeeded: delete_order
    status: 204
cases:
  - id: order-normal
    required: true
    inputs: {quantity: 2}
    steps: [create_order, get_order]
  - id: order-max-quantity
    required: true
    inputs: {quantity: 10}
    steps: [create_order, get_order]
  - id: order-invalid-quantity
    required: true
    inputs: {quantity: 0}
    steps: [reject_order]
  - id: order-deleted-not-found
    required: true
    inputs: {quantity: 2}
    steps: [create_order, delete_order, get_deleted_order]
```

- `name` 使用适合文件名的简短标识。已有单路径草稿可继续按顶层 `steps` 的顺序执行。使用 `cases` 时，顶层 `steps` 定义操作，各 case 的 `steps` 列出该用例的执行顺序。
- `save` 将响应的字段保存为变量，`$.id` 表示 JSON 顶层的 `id`。`${order_id}` 引用已保存的变量。`${quantity}` 替换为整个 YAML 值时保留输入的整数类型。
- `asserts` 按步骤 ID 关联断言。本步骤断言失败后跳过后续业务步骤，仍清理本次已创建且尚未删除的资源。已成功删除的订单从待清理记录移除。拒绝分支若意外返回建单成功，先记录响应中可识别的本次订单以供清理，再报告拒绝断言失败。分别记录业务断言和清理结果。

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


def create_order(orders_api, quantity):
    session, base_url, _ = orders_api
    created = session.post(
        f"{base_url}/orders",
        json={"sku": "demo-item", "quantity": quantity},
        timeout=10,
    )
    assert created.status_code == 201, (
        f"create_order: expected 201, got {created.status_code}"
    )
    return remember_created_order(orders_api, created)


def remember_created_order(orders_api, created):
    _, base_url, created_order_urls = orders_api
    order_id = created.json().get("id")
    assert isinstance(order_id, str) and order_id, "create_order: missing id"
    order_url = f"{base_url}/orders/{quote(order_id, safe='')}"
    created_order_urls.append(order_url)
    return order_id, order_url


@pytest.mark.parametrize("quantity", [
    pytest.param(2, id="order-normal"),
    pytest.param(10, id="order-max-quantity"),
])
def test_create_then_get_order(orders_api, quantity):
    session, _, _ = orders_api
    order_id, order_url = create_order(orders_api, quantity)
    fetched = session.get(order_url, timeout=10)
    assert fetched.status_code == 200, (
        f"get_order: expected 200, got {fetched.status_code}"
    )
    assert fetched.json().get("id") == order_id, "get_order: id mismatch"


@pytest.mark.parametrize("quantity", [pytest.param(0, id="order-invalid-quantity")])
def test_reject_invalid_quantity(orders_api, quantity):
    session, base_url, _ = orders_api
    rejected = session.post(
        f"{base_url}/orders",
        json={"sku": "demo-item", "quantity": quantity},
        timeout=10,
    )
    if rejected.status_code == 201:
        remember_created_order(orders_api, rejected)
    assert rejected.status_code == 422, (
        f"reject_order: expected 422, got {rejected.status_code}"
    )


@pytest.mark.parametrize("quantity", [pytest.param(2, id="order-deleted-not-found")])
def test_get_deleted_order(orders_api, quantity):
    session, _, created_order_urls = orders_api
    _, order_url = create_order(orders_api, quantity)

    deleted = session.delete(order_url, timeout=10)
    assert deleted.status_code == 204, (
        f"delete_order: expected 204, got {deleted.status_code}"
    )
    created_order_urls.remove(order_url)

    fetched = session.get(order_url, timeout=10)
    assert fetched.status_code == 404, (
        f"get_deleted_order: expected 404, got {fetched.status_code}"
    )
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

按 [测试分析报告](test-report.md) 逐个 case ID 汇总必测分支，再记录首个失败步骤、实际响应或异常、变量来源和去向、后续跳过步骤及清理结果。比如建单返回 201 却缺少 `$.id`，将失败归于建单响应，查单记为跳过。若建单成功保存 ID，且查单使用同一 ID 仍返回 404，记录查单的状态码和传参证据，根因待查。报告中的请求和响应应隐藏凭证。
