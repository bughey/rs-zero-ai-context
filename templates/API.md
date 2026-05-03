# {Service Name} API

## Base URL

```text
http://127.0.0.1:{port}
```

## Endpoints

### {METHOD} {PATH}

说明 endpoint 行为。

Request:

```json
{}
```

Response:

```json
{}
```

Error codes:

- `400`：请求不合法。
- `404`：资源不存在。
- `500`：服务内部错误。

## Observability

- route label 使用模板路径，不使用完整 URL。
- metrics endpoint：`/metrics`。
- readiness/health endpoint：按服务实现填写。

## Test Examples

```bash
curl -X GET http://127.0.0.1:{port}/ready
```
