# 综合行政管理系统：日报

页面：`http://192.168.5.235:91/integrated/IntegratedManagement/7/DailyReport`  
API 根地址：`http://192.168.5.235:91/dev-api`

调用前读取 [`../auth.md`](../auth.md)，验证 Bearer token、租户和当前用户。日报请求的 `Referer` 固定为本页；列表查询固定传 `type=1`。无明确变更确认时仅可调用读取接口。

## 已验证接口

| 方法与路径 | 用途 | 约束 |
| --- | --- | --- |
| `GET /integrated/dailyReport/page` | 分页查询日报 | 查询参数：`pageNum`、`pageSize`、`type: 1`，可选 `createUser`、`createDept`、`startDate`、`endDate`。 |
| `GET /integrated/dailyReport/weekByDay?day=YYYY-MM-DD` | 读取目标日期的周模板 | 新增前必须读取；保留返回的身份和日期字段。 |
| `POST /integrated/dailyReport/batchSave` | 批量新增 | Body 是 `DailyReport[]`，不是包装对象。 |
| `PUT /integrated/dailyReport` | 修改单条日报 | Body 必须包含 `id` 和完整当前记录字段。 |
| `DELETE /integrated/dailyReport/{id}` | 删除单条日报 | 不可逆；必须针对该 `id` 单独确认。 |

接口响应含 `code`、`msg`、`count`、`data`。按业务 `code` 判断成功；不能把 HTTP 200 当作已保存。

## 记录字段与校验

```ts
type DailyReport = {
  id?: number;
  type?: number; // 综合日报列表固定为 1
  dailyType: "销售日报" | "综合日报" | "研发日报" | "实施日报" | "运维日报";
  customerName?: string;
  contactPerson?: string;
  workContent: string;
  tomorrow?: string;
  completionStatus: string;
  workHours: number | string;
  leaderOpinion?: string;
  isRead?: string | null;
  createDate?: string;
  createTime?: string;
  createUser?: string;
  createUserId?: number;
  createDept?: string;
  createDeptId?: number;
  createBy?: string;
  updateBy?: string;
  updateTime?: string;
  delFlag?: string;
  tenantId?: number;
  time?: string; // 周模板的 YYYY-MM-DD
  day?: string;
  file?: unknown;
};
```

- `workContent`、`completionStatus`、`dailyType`、`workHours` 必填；去除首尾空白后，已填写的文本字段至少 15 个字符。
- 文字必须记录具体对象、动作、问题和结果；不虚构客户沟通、完成事项或承诺，不用空泛口号掩盖缺失事实。
- `tomorrow` 可空；填写或建议时应基于当前已知事实，并满足同样的文本长度要求。
- `dailyType="销售日报"` 时，`customerName` 与 `contactPerson` 必填。
- 系统仅接受 0.5 小时粒度时，不得把原始工时机械向上取整。展示原始值和相邻两个可提交值，由用户选择最贴近真实记录的值；用户已提供合规值时原样保留。
- 更新时保留读取到的完整记录、身份字段和只读字段，只应用用户指定的改动。

## 查询日志

1. 从认证响应取得填写人和部门，不猜测用户名或部门名称。
2. 构造查询；不传空筛选字段。周或日期查询直接传 `startDate`、`endDate`，不读取全量后在本地过滤。
3. 检查 HTTP 状态和业务 `code`，返回 `data`、`count`、筛选条件和必要字段。

## 新增日志

1. 确定目标业务日期，调用 `weekByDay` 读取模板。
2. 仅编辑目标日期、当前用户的非空行；保留模板的身份和日期字段，移除没有 `id` 的纯空行。
3. 当天未提供工时时，可从本地当天 `08:00` 到当前时刻计算候选值并展示；历史日期必须由用户提供实际工作区间、休息时长或确认工时。
4. 校验字段和工时粒度，生成仅基于事实的可编辑 `tomorrow` 建议。
5. 展示每条记录的业务日期、类型、原始及可提交工时、工作内容、完成情况、下一步动作和计划保存条数；只有用户针对完整数组明确确认后，才调用 `batchSave`。
6. 保存后按原筛选重新查询；失败不自动重试。

## 补充日志

补充日志是补写过去某日实际完成的工作，不是伪造历史填写或审计时间。目标业务日期与记录填写时间可能不同，不能以两者不一致判定失败。

1. 用户必须提供目标业务日期、工作内容、完成情况，以及实际工作区间和休息时长，或已确认工时。读取该日 `weekByDay` 模板；缺少事实工时则停止，不得套用固定上下班时间、当前时间或工作内容猜测。
2. 查询目标日期前连续 90 个自然日内当前填写人的**全部分页**日报结果。按业务日期汇总同日 `workHours`，并仅把 `createTime` 视为历史填写时间，不能据此推断上下班时间。
3. 先展示目标日已有记录和工时总和。存在至少 5 个同日报类型、同星期、相近工作情境的有效历史日时，计算工时中位数和四分位范围；样本不足明确标记“无可靠历史基线”。历史数据只能提示偏差，不能覆盖用户提供的实际事实。
4. 使用 `实际结束 − 实际开始 − 休息时长` 得到原始工时。展示原始值、相邻 0.5 小时候选、目标日已有工时、补充后日总工时、历史基线和现实原因。总工时超过实际可工作时长或 24 小时必须拒绝；落在历史范围外时说明差异并允许用户确认真实原因后继续。
5. `createDate`、`createTime` 默认保留服务端写入值。只有用户明确说明该字段是系统允许更正的业务填写时间、提供实际发生的完整时间戳，并对该次 PUT 单独确认时，才可修改。不得从历史分布、上下班区间或随机数生成时间戳，也不得强迫填写时间等于目标业务日期。
6. 展示完整新增数组；用户确认后调用一次 `batchSave`。保存后用保存前后的当前填写人及系统当日 `createDate` 的 `id` 集合差定位新记录；集合差不是恰好一个 `id` 时停止，不得用展示文本猜测记录。
7. 仅当第 5 步满足时，展示新记录完整 PUT body 和拟修改的日期/时间，取得独立确认后才调用 `PUT`。最后按目标日期、填写人和 `id` 重新查询，核对日报内容、工时及已确认修改的时间字段。

## 修改与删除

### 修改

1. 用日期、填写人定位记录，或让用户提供 `id`；禁止按工作内容猜测 `id`。
2. 读取完整记录，校验用户指定变更后的必填字段、文本和工时规则。
3. 展示完整 PUT body、`id`、变更项和影响；明确确认后提交。
4. 用原筛选重新查询核对；不自动重放失败的 PUT。

### 删除

1. 定位单条记录并回显 `id`、填写人、日期和工作内容摘要。
2. 用户明确确认删除该 `id` 后才调用 DELETE。
3. 用原筛选复核目标记录不存在；不得删除多条模糊匹配结果。

## 接口变更

字段、令牌或页面行为异常时，用浏览器读取真实 XHR/fetch，并记录 URL、方法、查询参数、请求 JSON 字段、状态码和脱敏响应字段。探索阶段不得调用日报 `POST`、`PUT`、`DELETE`。
