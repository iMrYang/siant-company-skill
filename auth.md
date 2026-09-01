# 信安通认证与会话

认证中心：`http://192.168.6.235:9045`  
综合行政系统：`http://192.168.5.235:91`  
综合行政 API 根地址：`http://192.168.5.235:91/dev-api`

## 会话文件

会话只允许保存在当前用户目录的 `~/.siant/siant-oa.json`；目录不存在时创建。文件可保存 `username`、`password`、`token`、`tenantId`、`savedAt`，但不得提交到仓库、写入项目配置或在用户可见输出中回显。

1. 若文件中存在 `token` 和 `tenantId`，先请求 `GET /dev-api/getInfo` 验证目标系统令牌并读取当前用户和部门。
2. 遇到 HTTP 401、403 或业务失败时，仅删除缓存的 `token`、`tenantId`；保留账号密码以便重新认证。
3. 没有可用凭据时向用户索取。验证码启用时，先获取并展示验证码，再请求用户填写。
4. 成功完成单点登录并取得目标系统令牌、租户及用户信息后，覆盖会话文件。例行查询不得重复登录。
5. 令牌在业务写请求中失效时，可以重新认证，但不得自动重放写请求；重新展示完整数据并重新取得确认。

## 已验证的认证接口

认证 API 根地址：`http://192.168.6.235:9045/prod-api`。

| 方法与路径 | 用途 | 请求/响应要点 |
| --- | --- | --- |
| `GET /captchaImage` | 获取验证码配置 | 返回 `captchaEnabled`，启用时包含 `img`、`uuid`。 |
| `POST /login` | 账号密码登录 | 请求 `{ username, password, code?, uuid? }`；验证码启用时后两项必填。 |
| `GET /getInfo` | 认证中心当前用户 | 使用认证中心令牌或会话 Cookie。 |
| `GET /sso/authorize` | 进入综合行政系统 | 查询参数使用 `clientId=jxkh` 与认证中心生成的 `state`；前端据此建立目标系统会话。 |

## 目标系统令牌校验

```http
GET /dev-api/getInfo
Authorization: Bearer <token>
X-Tenant-Id: <tenantId>
```

成功响应的 `data.user` 提供 `userId`、`deptId`、`tenantId`、`userName`、`nickName` 和可选 `dept`。将响应中的租户和用户信息作为后续查询的唯一来源；不得硬编码租户或假设当前用户。

## 综合行政 API 请求头

```http
Authorization: Bearer <token>
X-Tenant-Id: <tenantId>
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.9
Referer: <当前功能页面 URL>
```

`POST`、`PUT`、`PATCH` 的 JSON 请求另加 `Content-Type: application/json`。不要默认复制浏览器 Cookie、伪造 `Origin` 或添加未实测的头；端点确有额外要求时，先用浏览器捕获真实请求再补充。

## 未恢复的业务流程

出差、外出、报销、用章、消息通知和其他流程的固定 API 未在可恢复上下文中保留。需要处理时：

1. 先按本文件获得有效会话，并打开对应页面。
2. 只读取页面结构、下拉选项、已有记录和真实的 `GET`/加载请求。
3. 记录 URL、方法、查询参数、请求字段、响应字段和页面含义；敏感值脱敏。
4. 只有读取到真实请求 schema 后才可构造待提交数据；仍须遵守根技能的完整展示和单次确认规则。
