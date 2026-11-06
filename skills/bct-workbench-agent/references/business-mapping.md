# 业务能力与字段映射

以下来自既有项目的适配经验，不能直接作为其他版本的接口契约。修改时以当前工具 Schema、前端和实际请求核对。示例名称、账号和编号为占位内容，不用于生产录入。

## 合约查询

| 用户语义 | 页面工具字段 | 已确认的页面请求映射 |
| --- | --- | --- |
| 精确合约编号 | `filters.contractCode` | `contractCode` 数组 |
| 存续 / 非存续（已了结） | `filters.contractStatus` | `LIVE` / `CLOSED` 数组 |
| 交易达成日 | `dateRanges[].field=tradeDate` | `tradeStartDate` / `tradeEndDate` |
| 起始日 | `effectiveDate` | `effectiveStartDate` / `effectiveEndDate` |
| 期末观察日，接口也称到期日 | `expirationDate` | `expirationStartDate` / `expirationEndDate` |
| 结算日 | `settlementDate` | `settlementStartDate` / `settlementEndDate` |

多条件是 AND。既有工具每次替换旧条件，未指定字段不限制；不能保留默认近七天窗口后声称查了全部。

“最近一个月的非存续合约”缺少日期口径，需要确认；“最近一个月成交的非存续合约”已有交易达成日口径，可以直接组合：

```json
{
  "action": "query_contracts",
  "filters": {
    "contractStatus": ["CLOSED"],
    "dateRanges": [{"field": "tradeDate", "period": "LAST_MONTH"}]
  }
}
```

`LAST_MONTH` 是一个月前同日至今天；`LAST_30_DAYS` 是含今天的 30 个日历日；`PREVIOUS_MONTH` 是上个自然月；`THIS_MONTH` 是本月截至今天。由服务端按 `Asia/Shanghai` 计算，明确起止日期与相对 period 不混用。

“最近一个月发生了结”需要了结事件日期，不能用成交日或结算日替代。审批状态、持仓状态与合约存续状态也不同。确认书编号与合约编号分开识别，先查清对应关系再定位。

页面查询与 Swagger 中的列表接口可能不同；保留页面原生请求方式。文档用于解释字段，不授权替换页面端点。

## 收益互换原生簿记

动作：`prepare_booking`；当前已适配多头互换/空头互换与单条股票标的。先查工具 Schema，其他产品、复杂收益结构或多标的需要另行适配。

```json
{
  "action": "prepare_booking",
  "entry": {
    "booking": {
      "productType": "多头互换",
      "counterPartyName": "示例交易对手",
      "bookName": "示例交易簿",
      "capitalAccountId": "示例账户显示名称"
    },
    "underlying": {
      "code": "600000.SH",
      "quantity": "100",
      "quantityUnit": "SHARE",
      "price": "6"
    }
  }
}
```

注意参数名与语义的差异：当前 `capitalAccountId` 页面参数实际使用账户显示名称，不是内部 ID；以 Schema 和候选回读为准。示例不能直接用于真实交易。

- 用户给“100手”时保留手数意图，确认实际股数。不能未经验证直接乘 100；标的字段的最小变动单位/买卖单位不自动证明每手股数。
- 页面百分数字段 `5` 表示 5%，接口比例可能为 `0.05`；分开映射，不能混填。
- 价格、数量、乘数、币种、本金口径分别核实；期初、当前和标的本金不是同一个字段。
- “方向反转”不能自动解释为资产、利息、保证金和预付金全部反向；按具体条款确认。
- 用户要先填页面时，可只填已知字段，编号和未指定日期保留原生默认；不从历史合约复制原编号、系统 ID、审批状态或价格。
- 模板或历史合约只提供有明确依据的条款。未适配或未回读字段不代表为空，不宣称完整校验通过。
- 工具 `businessValidated=false` 表示本次没有业务校验成功证据。用户最终点击原生确认；不得改走后台创建接口完成页面任务。

## 标的物维护

动作：`query_instruments`、`create_instrument`、`edit_instrument`、`inspect_form`、`form_options`。

精确代码用于定位；编辑只变更用户明确指定的字段，不改标的代码、UUID 或系统控制字段。按动态表单映射页面字段，不能照抄接口 DTO，例如页面交易所键可能是 `exchange`，资产类别可能需要层级名称数组。

篮子、期权扩展、自定义属性等未适配内容明确待补。新增/编辑填写不等于保存。

## 交易对手与机构

| 目的 | 交易对手动作 | 机构动作 |
| --- | --- | --- |
| 查询 | `query_parties` | `query_institutions` |
| 查看 | `view_party` | `view_institution` |
| 新增填表 | `create_party` | `create_institution` |
| 编辑填表 | `edit_party` | `edit_institution` |

查询用 `partyQuery` / `institutionQuery`；查看编辑用精确 `partyCode` / `institutionCode`；填写用 `party` / `institution`。编号与显示名不能互换，编号包含中文也要原样保留。

新增交易对手可关联已有机构，不能顺便新建机构；修改交易对手不自动授权修改共享机构。销售路径的每个数组元素是一条完整路径，例如 `["示例分公司/示例营业部/示例销售"]`，不是拆成三个选项。

当前适配基础资料；协议、人员、部门、银行账户、授信、附件等复杂步骤需单独核对。看到默认“是”不证明客户资质事实。原生“下一步”也可能生成编号或调用业务接口，不能假定无副作用并自动点击。

后端所谓一段式/二段式新增可能分别指“同时创建机构与交易对手”和“关联已有机构”，不能按名称推断为 UI 向导第一步/第二步。

查询摘要可能截断：`rows` 不是全部记录，结合 `totalCount`、`rowsLimited` 和 `detailsOmitted` 说明边界，不补写未返回内容。
