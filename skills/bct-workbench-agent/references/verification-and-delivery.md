# 代码定位、验证与交付

## 到已有项目里找什么

以下为整理时 `spring-ai-starter` 的相对路径；使用前先核对实际文件和项目约定。

| 路径 | 作用 |
| --- | --- |
| `AGENTS.md`、`agent.md` | 用户确认的开发顺序、前端边界和后续修正 |
| `src/main/java/com/example/ai/agent/AgentService.java` | 模型、任务和工具调度 |
| `src/main/java/com/example/ai/agent/BookingTools.java` | 接口检索、草稿及受限 HTTP 执行 |
| `src/main/java/com/example/ai/agent/BrowserActions.java` | 页面工具 Schema、命令归属、期限及回执 |
| `src/main/java/com/example/ai/agent/PageEntry.java` | 页面参数白名单与归一化 |
| `src/main/java/com/example/ai/agent/ContractQuery.java` | 合约筛选字段与日期范围 |
| `src/main/java/com/example/ai/openapi/` | 接口目录、通用客户端、Schema 校验 |
| `src/main/java/com/example/ai/auth/` | 当前业务身份与登录验证 |
| `src/main/resources/static/widget.js`、`assistant.js` | 宿主挂件、消息传递、轮询和回执 |
| `src/main/resources/static/bct-login.js` | 已有工作台登录机制接入 |
| `src/main/resources/static/bct-page-actions.js` | 导航、查询、只读请求观察 |
| `src/main/resources/static/bct-form-actions.js` | 合约与标的物表单及页面状态 |
| `src/main/resources/static/bct-reference-actions.js` | 交易对手、机构查询与表单 |
| `src/main/resources/static/page-fields.json` | 已适配字段配置 |
| `src/main/resources/skills/return-swap-booking/SKILL.md` | 运行中的业务助手指令 |
| `src/main/resources/openapi/` | 本地接口快照，不随此经验仓库发布 |
| `src/test/js/`、`src/test/java/` | DOM、任务、参数与状态回归 |
| `deploy/` | 部署说明、改动记录及回滚脚本 |

## 开发前记录

记录目标原生入口、实际业务请求、部署包版本/哈希，以及“业务语义 → 页面控件 → 请求字段”的证据。没有源码时可以读取已授权构建包和后端 JAR；反编译证据只用于解释实现，不扩展业务权限。

旧部署文档可能写着更早的能力或验收限制。与当前代码、新修复记录和真实运行证据交叉核对，不把旧文档中的“所有弹窗都阻止导航”等历史行为当作现行要求。

## 有意义的验证

修改到哪一层，就验证该层的可观察结果：

- 参数与语义：未知字段拒绝、日期边界、日期口径、精确编号、单位转换和多条件组合。
- 身份与任务：跨用户/标签页拒绝、退出登录、凭据不进入模型或持久化、重复命令和超时不重放。
- 页面：旧筛选清理、模板重建、候选精确匹配、查询条件与实际请求一致、HTTP 200 业务错误、正常零条。
- 表单：原生弹窗入口、空白接管、本任务继续、人工/其他任务保护、嵌套弹窗、联动加载等待、用户点击确认后停填、原弹窗关闭后重新创建。
- 保存状态：助手未提交不等于用户未保存；人工声明只建立可失效基线，不生成后台保存证明。
- 失败结果：部分填写和已经打开的表单仍应如实回读。

既有项目使用 Java 21；按项目实际 Maven 配置构建，常用检查为：

```sh
mvn -B package
node --test src/test/js/*.test.cjs
```

新机器先核对 Java/Maven 版本。测试失败应定位原因；通过后不无依据重复全量检查。DOM 测试不能代替必要的真实页面验收。

## 页面验收边界

使用独立验收标签页，执行本次授权的只读查询或表单填写；不为测试保存虚构标的或提交真实/虚构交易。完成后关闭自己创建的临时表单，保留用户原有现场。

核对用户能看到的状态与结构化回执：路由、原生弹窗标题、实际字段、查询条件和错误。截图应能让用户验证结果；对外分享时另行脱敏。

保存成功后的关闭/刷新可通过原生代码分析确定预期行为；若没有实际保存验收，必须如此标注，不用源码分析冒充端到端结果。

## 已授权部署

1. 先完成代码、相关测试和可核验的构建产物，明确改动文件。
2. 核对目标环境和正在执行的任务，保留旧 Agent 包、主入口及必要配置的受限备份。
3. 发布 Agent，原入口只更新助手引用/必要配置；业务 JS/CSS 不修改。脚本缓存版本与构建内容保持一致。
4. 核验服务健康、部署包哈希、实际返回脚本和页面行为。`modelReady` / `businessReady` 只说明已配置，不能替代真实调用。
5. 提供回滚路径。回滚前验证主入口仍是本次部署版本；发现他人更新则停止覆盖，先合并或制定局部回退。

不要把环境文件、密码、模型密钥、Bearer token、任务数据、原系统构建包或真实业务截图放入 Git。示例配置只能有占位值。

## 每次交付说明

简要写清：用户问题与现在的行为、Skill/协议/执行器分别改了什么、原业务文件实际修改范围、验证证据、未验证项、发布与回滚位置。不要只写“已优化”或把所有问题统称为模型幻觉。
