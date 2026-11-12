# harnesses 技术方案

日期：2026-11-13。状态：开发起点，服务尚未部署。产品范围以[总体需求](product.md)为准。

## 1. 架构与选型

Git 仓库保存拆分的结构化资料。构建先校验实体和关系，再生成不可变快照。网页读取该快照生成产品和厂商页，API 和 MCP 复用查询模块。客户端自行决定如何使用规则进行本机识别、安装和调用。

| 层 | 选型 | 用途 |
| --- | --- | --- |
| 开发 | Node.js 24、TypeScript、npm workspaces | 同一套类型、数据构建和查询代码。锁文件固定依赖 |
| 数据 | UTF-8 JSON、JSON Schema 2020-12、Ajv | 让贡献可以逐条审核，并提供语言无关的数据契约 |
| 网页 | Astro 静态生成，交互用 TypeScript | 预生成中英文目录、厂商页、产品页及 Docs |
| 服务 | Cloudflare Workers、Static Assets、R2 | 同域提供网页、API、MCP 与版本化下载 |
| MCP | 官方 TypeScript SDK 稳定 v2、Streamable HTTP | 只读查询，按协议兼容性测试固定具体 SDK 版本 |
| CI | GitHub Actions | 验证资料、运行测试、构建快照和预览 |

本地开发沿用已安装的 Node 24 与 npm。第一阶段在本机验证契约和快照，部署环境、具体依赖版本和兼容日期在进入相应阶段时固定。[选型记录](decisions/0001-versioned-catalog.md)列出了规模边界和替换条件。

## 2. 仓库结构

以下目录随实施阶段建立。

~~~text
data/
  vendors/<vendor-id>.json
  harnesses/<harness-id>/
    product.json
    releases.json
    downloads.json
    rules/
      detection.json
      paths.json
      configuration.json
      extensions.json
      operations.json
      sessions.json
      capabilities.json
  sources/<source-id>.json
  evidence/<evidence-id>.json
  relationships/<relation-id>.json
schemas/
  v1/
packages/
  catalog/        # 规范化、条件匹配、证据关联、搜索与投影
scripts/
  build-catalog/  # 读取源文件，生成快照、清单和变更报告
apps/
  web/           # 静态目录、档案、对照工具、Docs
  worker/        # HTTP、MCP 与存储适配
docs/
~~~

JSON Schema 是数据契约的来源。构建产生 TypeScript 类型，CI 检查生成结果和快照一致。Ajv 在构建和 CI 校验原始资料。服务入口另行校验请求参数，不在边缘请求中动态编译资料 Schema。

## 3. 收录范围与数据模型

### 身份、关系和覆盖

每个可核验的独立产品有稳定的 slug ID。现名、旧名、中英文名、包名和可执行文件名用于查询。重命名保留旧引用和跳转。平台、架构、渠道和版本作为条件。独立发布或有独立官方身份的形态可分档，共用内核、共享目录和发行关系通过独立记录关联。

WorkBuddy 与 WorkBuddy AI 是两条独立产品档案。其厂商关系与官方来源分别核验。厂商可有开发、发行、运营等多种角色和有效时间，旗下列表由产品关系反向构建。停止维护的产品保留可查询的历史资料。

收录覆盖 CLI、桌面、IDE 扩展、云端和自托管产品，以及不同地区与发布渠道。候选发现从官方站点、公开仓库、包注册表、应用或 IDE 扩展市场、厂商产品目录和社区提名入手。候选条目保留发现入口和核验状态，按产品身份去重。已有可追溯身份依据的产品可先发布最小档案，逐步补充其他模块。未知状态按字段记录，统计分别给出已收录产品数和各资料模块覆盖数。

### 条件与证据

| 记录 | 关键字段 |
| --- | --- |
| 产品、厂商 | 稳定 ID、名称、别名、形态、状态、关系、官方链接及证据引用 |
| 版本、下载 | 上游版本字符串、渠道、发布时间、平台与架构、URL、已知校验和或签名 |
| 规则 | 稳定规则 ID、产品 ID、类型、适用条件、作用域、结构化规则体、证据引用 |
| 来源 | URL、类型、抓取或核验日期、可选提交或版本定位 |
| 证据 | 引用来源、被支持或质疑的字段或规则、验证方法、适用环境、结论与时间 |

首版规则条件包含平台、架构、版本、渠道、形态和作用域。发布解析显式区分已匹配、条件不足、零匹配、冲突、已知不支持和未知。版本采用上游版本字符串，只有经确认支持的版本体系才使用范围比较。相互冲突且条件重叠的记录一起返回，附来源和冲突原因。规则替代或优先级需要记录依据。给定同一快照与条件时结果可复现。

路径通过主目录、项目根、环境变量等逻辑占位符表达。规则允许有默认值和官方约定的覆盖顺序。调用规范用程序名、参数数组、前置条件、输入输出格式、退出码、超时和取消行为表达。检测规范记录信号类型、强弱、共享或残留条件，避免只凭目录存在便确认产品。数据对照比较前保留原始作用域和适用条件。

示意记录使用虚构产品和来源，首批真实数据由审核流程核验。

~~~json
{
  "id": "example-harness:paths:user-config",
  "harness_id": "example-harness",
  "type": "path",
  "purpose": "configuration",
  "when": { "platforms": ["macos", "linux"], "scope": "user" },
  "value": { "template": "{home}/.example-harness/config.json", "target": "file" },
  "evidence_ids": ["example-harness:docs:config"]
}
~~~

## 4. 构建、查询和快照

构建顺序为加载、Schema 校验、引用完整性检查、归一化、索引生成、发布产物校验。CI 拒绝重复 ID、断开引用、未知枚举和非法适用条件。存在争议的候选事实可以记录为冲突或待核验，公开事实仍须有来源。历史快照保留原有内容，更正通过新快照发布。

快照 ID 来自稳定序列化的规范化数据、完整 schema_version 和产物格式版本。生成时间、Git SHA 和含 ID 的外层字段独立于摘要。索引或投影规则改变产物内容时提升产物格式版本。重复发布同一个快照沿用原始清单，CI 比对已有产物摘要。清单记录生成来源 Git SHA、schema 版本、发布时间、文件摘要和计数。

| 产物 | 消费者 |
| --- | --- |
| manifest.json | 获取 schema 版本、哈希、文件大小、计数和可下载文件 |
| catalog.json | 离线完整数据，包含产品、厂商、关系、规则、来源及证据 |
| index.zh.json、index.en.json | 搜索词、可筛选字段、稳定 ID、名称和摘要 |
| harnesses/<id>.json、vendors/<id>.json | 产品与厂商详情按需加载 |
| changes/<from>/<to>.json | 比较两个固定快照的新增、删除、更新及关系修正 |

查询对输入执行 Unicode NFKC 归一化，采用 ID、正式名称、别名和包名精确命中优先，再做前缀、标记和中英文文本匹配。以稳定 ID 为同分时的排序条件。初期对压缩摘要索引做搜索，详情和规则按需加载。浏览器按页显示，服务游标绑定快照、归一化查询及排序位置，变化后返回可恢复错误。

选型验证使用至少 1,000 个合成产品、多个重名别名和具有版本条件的规则。样例规模下设工程预算：每语言搜索索引 gzip 后不超过 300 KiB，普通列表每页默认 20、上限 100。若超预算，再对摘要索引分片或将搜索索引迁到专用读模型，接口仍读取版本化快照。

## 5. 网页、API 和 MCP

网页以产品目录和数据接入为首页主线。中文、英文入口分别预生成，产品事实使用同一 ID。目录支持查询和组合筛选，状态写入 URL。产品页显示官方入口、平台、版本、规则、出处和待核验内容。厂商页由关系生成，比较作为数据使用场景。现有视觉原型可作为交互参考，正式页面直接读取构建产物。

首版 API 以 /api/v1 为前缀。读接口公开，入口实施查询长度、结果量、字段选择和请求频率限制。OpenAPI 契约随首个 API 实现发布。

| 请求 | 用途 |
| --- | --- |
| GET /api/v1/manifest.json | 当前快照、摘要和文件 URL |
| GET /api/v1/catalog.json | 当前整库 JSON |
| GET /api/v1/snapshots/{snapshot_id}/catalog.json | 固定快照 |
| GET /api/v1/harnesses | 查询、筛选、分页 |
| GET /api/v1/harnesses/{id} | 档案和按需展开的规则、下载与证据 |
| GET /api/v1/harnesses/{id}/rules | 按条件查询规则和未决项 |
| GET /api/v1/vendors/{id} | 厂商及关联产品 |
| GET /api/v1/sources/{id} | 来源与核验信息 |

列表、详情与规则返回 schema_version 和 snapshot_id，接受 snapshot 参数固定版本。列表支持 q、vendor、surface、platform、capability、cursor 和 limit。稳定错误体包含 code、message 与可恢复信息。条件不足与规则冲突用结构化查询结果表达，产品不存在用 404，未知快照与无法继续的游标分别有稳定错误码。

MCP 以 /mcp 为入口。第一批工具为 search_harnesses、get_harness、get_vendor、get_rules。输入支持相同的快照、字段、分页及规则条件，输出包含结构化数据、来源引用和未知项。每个工具调用使用同一查询层。兼容性测试覆盖当前稳定 SDK 对应的协议版本及至少一个拟支持的远程客户端，Docs 按实测列出支持矩阵。服务器提供公共资料，安装器与 Agent 负责本机操作。[MCP Streamable HTTP 规范](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports) · [官方 TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)

## 6. 发布与运行

1. PR 执行资料校验、类型检查、构建和契约测试。向审核者显示每条变更的来源与适用范围。
2. 合并后生成单次构建的快照、网页和发布清单，先上传不可变产物至 R2。已存在的同 ID 产物复核摘要，一致后可重试发布。
3. 在预览环境验证网页的 snapshot_id 与 HTTP、MCP、下载清单一致。验证完成后以一个 Worker 版本切换最新站点与查询入口。失败保留上一版本。
4. 当前清单采用短缓存并支持 ETag。固定快照 URL 按不可变资源缓存，整库 JSON 使用流式读取。跨版本请求通过 snapshot_id 锁定，不依赖 CDN 同时刷新。
5. 需要回退时恢复已验证的 Worker 版本，已发布快照的固定 URL 继续可读。记录发布版本、Git SHA 与快照 ID 的对应关系。

Astro 构建的静态页与 Worker 使用同一个发布 ID，客户端后续查询显式传该 ID。Worker 的业务逻辑读取绑定的快照，R2 和环境变量在适配层处理。请求时不会按用户提供的来源 URL 抓取内容。公共规则中的外部文档、命令与路径以数据展示和返回。跨域读取、Origin 校验、请求大小与配额按入口的公开访问需求验证。运行指标跟踪延迟、错误码、缓存命中和正在服务的快照版本。

实际部署前验证资产文件数、文件大小、CPU、内存与费用预算，再配置生产域名。具体版本和限制以部署时平台为准。[Worker 静态资源路由](https://developers.cloudflare.com/workers/static-assets/routing/worker-script/) · [R2 Workers API](https://developers.cloudflare.com/r2/api/workers/workers-api-reference/) · [Worker 限制](https://developers.cloudflare.com/workers/platform/limits/)

## 7. 实施顺序和验收

| 阶段 | 交付 | 验收 |
| --- | --- | --- |
| 数据契约与构建 | 产品、厂商、关系、来源、证据、规则 Schema，第一份可下载快照 | 同厂商不同产品可分别查询，断开引用或缺少证据的事实导致校验失败，相同内容生成同一快照 ID |
| 查询和资料录入 | 条件解析、搜索投影、候选与审核流程 | 中英文、重名、别名和包名可检索，平台、版本、未知和冲突有测试，1,000 条合成数据通过分页和体积检查 |
| 网页和 API | 首页、目录、档案、厂商页、Doc 与读 API | 从公开页面到数据和来源链路可走通，同条件查询与快照文件一致，暗色和中英文切换可用 |
| MCP 和预览发布 | 远程工具、版本化产物存储、预览和回退 | HTTP 和 MCP 返回同一投影，旧快照可读，失败部署保留上一版，至少一个实际客户端走通 |
| 扩大覆盖 | 持续发现、去重、复核和发布更新 | 产品名录与资料覆盖分别度量，能持续接收增量，已下线产品及历史资料可追溯 |

第一份实现从资料构建开始。先定最小实体与证据约束，再用不同形态、至少一组关联产品和一条按版本或平台变化的规则贯通构建。发布门槛按可追溯性与查询一致性判断，样例和压力测试数据分别标注。

正式对外提供可复用数据前需确定代码与数据许可、第三方来源引用规则、贡献条款和服务配额。部署开通与付费额度在预览实测后决定。

## 8. 实现参考

- [Astro 静态路由](https://docs.astro.build/en/guides/routing/)
- [Ajv 与 JSON Schema](https://ajv.js.org/json-schema.html)
- [Cloudflare Workers 版本回退](https://developers.cloudflare.com/workers/versions-and-deployments/rollbacks/)
