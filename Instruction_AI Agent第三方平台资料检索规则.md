## 第三方平台资料检索规则

### 检索能力来源与使用方式

以下名称表示可移植的能力类别，不代表每台设备均已安装。使用前从当前 Agent 的技能清单、MCP/工具目录和命令行帮助中确认可用能力，并读取对应技能的 SKILL.md 及其引用的说明文件；安装状态、集成配置、命令行程序和登录会话均按当前环境判断。

- **TikHub：**以 tikhub-onboarding 为入口，按需配合 tikhub-endpoint-discovery 及其 tikhub-find-endpoint、references/openapi-index.json；端点以官方实时 OpenAPI 为准。已接入 TikHub MCP 时参照 tikhub-mcp 及当前 MCP 配置，否则按官方 API 文档调用 REST；API Key 通过当前环境的密钥配置使用。端点价格以 Marketplace 为准，所有调用均按付费服务计。
- **OpenCLI / smart-search：**smart-search 是基于 OpenCLI 命令行的检索路由技能，应与 opencli-usage 配套，并按任务读取 smart-search 的相关 references/sources-*.md。通过 opencli list 和所选站点、命令的实时帮助发现适配器及参数；实际执行方式和浏览器/Bridge 要求以适配器当前策略为准。
- **Socai：**通过 socai-social-research 技能使用当前环境的 socai 命令行工具；命令说明见其 references/commands.md，状态和参数以 socai status --json、socai <site> --help 及当前版本支持的平台为准。Socai 使用其管理的 Chrome 会话。
- **Computer Use：**指当前运行环境提供的图形界面交互能力及其 computer-use 技能说明，适用于需要页面操作或视觉核验的只读场景；是否可用以当前 Agent 的能力清单为准。

适用于小红书、知乎、微信公众号、Bilibili 和 YouTube 的公开内容检索。目标是在只读、低风险前提下发现有代表性的候选，并核实入选对象的正文、图片、评论和互动信息；视频网站还包括标题、简介、字幕/转写、缩略图等视频外信息。只有平台提供总量与分页/游标且已完整遍历时，才报告结果或评论全集覆盖。

1. **访问范围与证据**
   - 操作范围限于公开页面和只读接口，按任务需要分页；平台互动、发布、购买和资料修改等写操作不纳入任务。免费检索仅通过 OpenCLI/smart-search、Socai、Computer Use 等支持路径进行，禁止用自写脚本、裸请求或其他未支持方式直接抓取平台数据。图片和媒体按需临时读取，用户未要求保存时仅用于本次核验。
   - 搜索卡、标题、片段、AI 摘要和转述用于发现候选。引用证据时区分平台正文/简介、平台字幕、评论/互动字段、图片可见文字、OCR 转写和视觉判断。
   - 对每个入选对象记录稳定 ID、作者/账号、标题、日期、原始链接、目标字段读取情况及缺口；图片、字幕、评论分别记录可见总量与实际读取量。

2. **工具路由与成本**
   - 工具路由结合任务覆盖、字段需求、证据质量、效率和成本灵活选择。通常先用免费的 OpenCLI/smart-search、Socai 等支持路径；覆盖、字段或读取效果不足时再用付费 TikHub 补足，任务所需时即使费用较高也可调用。Computer Use 用于需要浏览器交互或页面核验的场景，按需采用。
   - 按平台和目标字段选择具体端点；智能路由负责发现路径，覆盖情况以实际下游和原始证据为准，避免重复调用。
   - 批量任务或预计费用较高时，按 Marketplace 当前端点价格和计费粒度估算搜索分页、详情、评论/回复、互动统计、媒体接口及必要试调/重试的费用，并向用户说明预估；免费路径可满足的部分交由相应工具完成，通过缩小范围、按稳定 ID 去重和分阶段读取控制 TikHub 调用量。
   - 保存响应、request_id、查询条件和游标，便于复用及判断是否重试。TikHub 每次实时请求均独立计费，MCP 与 REST 共用计费；cache_url 仅用于查看同一响应。
   - 端点、参数或区域不明确时查 TikHub 实时 OpenAPI 文档；中国大陆按文档选择 `api.tikhub.dev`，使用 TikHub API Key。MCP 会话异常或请求超时后，先判断请求是否已执行，再决定是否切换 REST 或重试。
   - 按实时 OpenAPI 构造请求时，参数定义分散在两处：GET 端点在 `parameters` 的 query 层，POST 端点在 `requestBody` 的模型内；`required`、`example` 和 `default` 也可能只写在 parameter 对象层而非其 `schema` 内。只读取其中一处会漏发必填项并返回 HTTP 422，填参前需同时检查这两层。首次调用某端点先核对响应字段层级，仅打印键名与列表长度，确认解析路径后再判定结果是否为空，不把取值键名错位记作通路故障；确需重试时按新的解析位置复核，不重复同一请求。
   - 仅在所选 OpenCLI 适配器使用浏览器桥接时运行 `opencli doctor`；需要切换当前连接的 Bridge 实例时使用 `opencli profile list`。同一浏览器会话的导航串行执行；Socai、OpenCLI 和 Computer Use 会话相互独立。

3. **检索范围与候选覆盖**
   - 先用精确关键词，再依据实际结果补充同义词、别名、账号、内容类型或时间筛选；视频网站限定视频结果。记录关键词、排序、筛选、页码和游标。
   - 按游标继续检索，直至无更多结果、达到任务所需的主题/时间/观点覆盖、结果高度重复或平台限频。候选兼顾相关性、时间、作者、互动量、内容类型和观点/使用场景差异；高互动及低互动但高度相关的内容均可纳入，并按稳定 ID 去重。
   - 多平台任务维护简要台账，记录目标字段、尝试路径、候选总数、实际返回数、已详细读取数、分页/游标、状态和缺口。

4. **平台读取路径**
   - **小红书：**TikHub 可检索并读取去重后的入选笔记；大批量时优先用 OpenCLI 或 Socai 免费发现候选，OpenCLI 优先于 Socai；TikHub 用于补足未覆盖的质量或字段。OpenCLI 的 `note` 读取正文和互动，`comments --with-replies` 读取评论及楼中楼，`download` 读取图片；Socai 用 `search` 发现候选，以带签名的 `get-notes` 批量读取正文、图片 OCR、评论和回复。Computer Use 用于补充页面核验；详情已有图片 URL 时优先直接核验原图。保留 note_id、xsec_token 和完整签名链接；搜索卡片遇到导航拒绝时，使用同一 ID 与原 token 打开规范的 `/explore/<note_id>?xsec_token=<token>&xsec_source=pc_search`。`opencli doctor` 的 `adapterShadows` 列出被本地 eject 覆盖的适配器；本机 `xiaohongshu/search` 的覆盖用于适配新版页面搜索框，执行 `opencli adapter reset xiaohongshu` 会失去该适配。
   - **知乎：**TikHub 可发现问题、回答和文章，并通过 ID 结构化接口读取正文和评论；OpenCLI/smart-search 用于免费发现或补充读取，Computer Use 用于页面补充。OpenCLI 可用 `answer-detail`、`answer-comments` 读取；专栏读取原文全文及图片引用，页面导航失败时改走 ID/API。详情评论数为 0 且评论接口返回 `EMPTY_RESULT` 时，记为已核实零评论。TikHub 没有覆盖全部内容类型的综合搜索端点，按垂类分别检索：文章用 GET `/api/v1/zhihu/web/fetch_article_search_v3`（`keyword`、`offset`），结果在 `data.data` 数组内，其中混有 `hot_timing` 等非内容条目，按条目 `type` 过滤后再计数。
   - **微信公众号：**TikHub 可搜索和读取原文；smart-search/OpenCLI（含搜狗发现）用于免费发现或补充候选，Computer Use 用于页面补充核验。候选筛选为 `mp.weixin.qq.com/s/...` 原文，并核对账号、标题和发布时间。TikHub 搜索使用 POST `/api/v1/wechat_search/v2/fetch_search`，设置 `business_type=article`、`raw=false` 并保存游标；对入选文章优先 POST `/api/v1/wechat_mp/v2/fetch_article_detail_h5`（`raw=false`）读取正文和图片。`fetch_article_detail` 与 H5 是独立端点，每篇按文档推荐选择详情路径；互动统计、评论和回复另行计费，按需读取并计入预算。按端点文档设置超时；搜索返回 `no_more` 且无结果时记为空。传给 OpenCLI 前删除 `#rd` 等片段并保留有效参数。
   - **Bilibili、YouTube：**TikHub 可限定视频搜索，并按 BV 号或视频 ID 读取标题、频道/作者、时间、简介、互动、缩略图、平台字幕和评论；OpenCLI/smart-search 用于免费发现或补充读取，Computer Use 用于前两者无法满足的页面核验。字幕为空时先用已知有字幕的视频核对工具链，再视需要转写并标明来源。Bilibili 主评论入口上限为 50，按 rpid 读取楼中楼；YouTube 评论请求上限为 100。报告实际返回及读取数量。TikHub 搜索端点为 GET `/api/v1/bilibili/web/fetch_general_search`（必填 `keyword`、`order`、`page`、`page_size`，结果在 `data.data.result`）和 GET `/api/v1/youtube/web_v2/get_general_search_v2`（`keyword`）。
   - Windows 命令行传递含 `&` 的链接时，将完整 URL 作为单个参数；续页使用原始链接、ID 和返回游标。
   - 如使用浏览器，优先使用 opencli daemon连好的Chrome浏览器，因为该浏览器已经登录各平台的账号。

5. **图片、评论与互动**
   - 对纳入范围的对象按展示顺序读取全部可见图片，记录总数、已读数、OCR 数及失败数。分别标注正文文字、图片可见文字、OCR、视觉判断和无法辨认处；关键数字或单位不清时保留不确定性并复核。
   - 图片 CDN、热链、空白预览或 `connect EACCES` 等错误按当前资源或执行路径问题记录；核对原图 URL、备用地址和签名，再试一个独立路径。将未能读取的图片列为缺口，并区分网络环境问题与登录状态。
   - 评论记录平台总数（若提供）、实际返回数、已读数、主评论/回复数、分页和抽样依据。评论较多时按相关性、互动量、时间、作者和观点差异抽样；只读取部分时标为“部分覆盖”并说明数量与范围。“高赞”“热门”依据实际互动字段。

6. **失败处理与用户接管**
   - 保存查询、ID、页码/游标、错误和会话状态；查实时文档、参数、端点与区域设置后，对已明确修正的问题复测一次，再切换独立只读路径。TikHub 超时或连接中断时先查 request_id、响应缓存和账户用量，再判断是否重试。
   - 将 TikHub HTTP 400/404 作为端点或请求路径问题核查；`No valid session ID provided` 先检查 MCP 会话。OpenCLI 的 `Navigation rejected` 记录为浏览器路径问题；`fetch failed` 区分 URL、CDN、网络环境和登录状态。
   - 执行 OpenCLI 浏览器适配器前，先检查 daemon 与 Chrome 扩展连接状态。`opencli daemon status` 出现 `extensionConnected: false` 且 `profiles` 为空时，浏览器命令会等待约 45 秒再以 `BROWSER_CONNECT` 失败。依次确认 Chrome 已启动、扩展装在当前启动的 profile 内；两项成立仍未连接，说明扩展的后台 service worker 已被系统回收，`opencli daemon restart` 不能使其重连，需在该 profile 中打开扩展自身页面（默认安装为 `chrome-extension://ildkmabpimmkaediidaifkhjpohdnifk/popup.html`，扩展 ID 以本机安装为准）唤醒 worker 后复查状态。桥接宿主是承载扩展的 Chrome 窗口，窗口关闭即断链；仍无法唤醒时说明当前状态并请用户处理 Chrome 扩展连接。沙箱内到 `127.0.0.1` 的本地 socket 可能被拒绝（错误码 10013）且 `LOCALAPPDATA` 等路径被改写，`config.toml` 中的 `network_access = true` 因此未必生效，此时浏览器与 API 命令在沙箱外执行。
   - 出现平台频率告警、账号异常、IP 限制或安全提示时立即停止该路径并记录。验证码、登录或浏览器接管需要用户操作时，说明具体应用、窗口和步骤后暂停；用户在平台页面完成登录或验证，凭据仅由用户在平台输入。OpenCLI 桥接默认复用已登录 Chrome；Socai 登录在其 managed Chrome 完成；Computer Use 仅在用户明确授权后操作当前浏览器。随后从已保存游标继续。

7. **完成判定与结果披露**
   - 按平台标明已完成、部分完成或暂时阻塞，报告候选与页数、实际读取对象、图片/OCR、字幕/转写、评论/回复覆盖、未取得字段、路径和错误。
   - 参考资料只列实际读取过原文、图片、字幕或评论的标题、作者、日期和链接；候选发现与原始证据分开。正文、图片文字、OCR、视觉判断、平台字幕、机器转写、评论和互动数字分别注明来源。视频网站以视频外信息为主。
   - 使用 TikHub 时列出各端点调用数、费用估算及可查到的实际扣费；注明实时价格来源和核对时间，未核实金额标为估算。端点单价以实时 `GET /api/v1/tikhub/user/get_endpoint_info?endpoint=<path>` 的 `endpoint_cost` 为准，参数名为 `endpoint`；响应中“本次请求将被计费”的提示对所有成功响应通用，0 单价接口同样会出现，不能据此判断是否计费。`get_user_daily_usage` 返回的是同一 API Key 的当日全量汇总，包含其他会话的调用，不能单独归因本次任务；无法区分时报告本次调用数与按单价推算的金额，并说明账户汇总为独立证据。

8. **工具维护**
   - 安装或更新能力后核对版本和工具列表，再用低风险对象确认所需字段可读；规则、端点或已知限制变化时更新本节。
