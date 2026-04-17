---
name: ozon-1688-upload-pipeline
description: 完整的 1688→Ozon 跨境上货流程：从1688抓取商品数据和图片，OCR+翻译图片中文为俄语，生成Rich-Контент富文本详情，再通过Ozon API绕过Web前端直接上架。
category: ecommerce
tags: [ozon, 1688, translation, apipudding, ecommerce, opencli]
---

# ozon-1688-upload-pipeline

这是用户经过验证的、针对 Ozon 跨境电商的上架全流程操作指南。必须严格按照此流程执行，**不要启动 Web UI**，直接使用命令行和 Python 脚本调用 API。

## 关键坑点（血泪教训）

### ⚠️ /v3/product/import 覆盖陷阱
- 重新提交 `/v3/product/import`（如为了更新图片）会**覆盖所有属性**为提交时传入的值
- 已填好的描述、标签、技能、学科等属性会被清空！
- **必须在 re-import 后立刻用 `/v1/product/attributes/update` 补回所有属性**
- 或者在 re-import 的 body 里就把所有属性都带上

### ⚠️ Rich Content 外链图片
- RC 里的图片 URL 可以用外链（如 iili.io），不必是 CDN URL
- RC JSON 格式必须严格：外层 `{"content": [...], "version": 0.3}`
- 每个 block 必须有 `imgLink`, `text: {content: null}`, `title: {content: null}`
- widget 顶层也要有 `text: {content: null}`

### ⚠️ 上架后必须全面审查
- 提交后等待 10 秒再查属性，确认每个字段真正生效
- 特别注意 re-import 后属性是否被清空

## 核心限制与规则
1. **分类和标题**：必须主动检索、动态匹配俄罗斯语言的 Ozon 分类树 (`description-category/tree` API) 和对应的属性(`type_id`)进行填报，**严禁自行捏造分类或直接代填空字典**。
2. **图片翻译逻辑**：优先使用**阿里云 TranslateImage API**（脚本 `~/ozon_pipeline/ali_translate_image.py`），自动去水印+翻译中文为俄语，效果最好。备选方案：豆包图生图、NanoBanana（余额有限）。先用 OCR 预过滤（Tesseract），有中文的图片（含 `[\u4e00-\u9fff]`）必须翻译。**不包含中文的图片直接复制，跳过翻译以节省 API 额度**。如果 OCR 漏判了带有艺术字的中文字符图片，必须强制调用接口翻译。
3. **上传限制绕过 (富文本图片致命坑点)**：
   - 由于 Ozon 的 `/v1/media/upload` 或 base64 上传经常报错 404，**必须将所有翻译后的图片提前上传到公共免费图床**生成公开外链，再通过 API 绑定。
   - **致命警告**：严禁使用 `litterbox.catbox.moe`, `uguu.se`, 或 `imgur.com` 作为富文本图片的图床！这些图床在俄罗斯被墙（或拦截了 Ozon 抓取），会导致 API 成功但俄罗斯买家前台看到的是**破损图片/显示出错**！
   - **公共仓库脱敏要求**：可以使用对俄罗斯友好的图床（例如 `freeimage.host` 或等效方案），但真实 API key / upload token 必须从**本地私有配置**读取，绝不能写进 skill、仓库、日志或截图。
4. **定价算法**：
   - 公式：`最终售价 = (采购成本 + 运费 10 CNY) / 0.8 / 0.7`
   - Ozon 系统的原价（划线价）：`原价 (old_price) = 最终售价 * 2`
5. **归档解除**：导入后新建的临时货品大部分会进入归档（Archived）状态，创建后必须紧接着调用 `/v2/product/unarchive` 或 `/v1/product/unarchive` 将其解封。注意必须传 `product_id`。
6. **营销与内容填充 (极其重要)**：上架完成后，必须针对该产品提取或生成以下俄语营销属性字段，并使用 `/v1/product/attributes/update` 更新进 Ozon。
   - **清理本地缓存**：商品成功上架且所有富文本、图片均验证无误后，**务必立即通过 terminal 或 python** 删除本地 `~/ozon_pipeline/products/{offer_id}` 的目录释放空间。
   - **Rich-Контент (ID: 11254) 格式 (2024 最新严格修正)**：
     - **极度警告**：Ozon 已经废弃或严格限制了 `raImage` 组件！如果你仅使用 `raImage` 插入图片，Ozon 后端会报错或默默清空商品详情导致显示空白。
     - **必须使用 `raShowcase` 嵌套组件**，外层必须包含 `content` 数组和 `version: 0.3`：
     ```json
     {
       "content": [
         {
           "widgetName": "raShowcase",
           "type": "roll",
           "blocks": [
             {
               "imgLink": "",
               "img": {
                 "src": "图片URL1",
                 "srcMobile": "图片URL1",
                 "widthMobile": 800,
                 "heightMobile": 800
               },
               "text": {"content": null},
               "title": {"content": null}
             }
           ],
           "text": {"content": null}
         }
       ],
       "version": 0.3
     }
     ```
     - **重要**：如果只传 `[{widgetName:...}]` 数组而不包裹在 `{content:[...], version:0.3}` 里，Ozon 会报 `invalid_rich_content_json` 并静默清空！
     - **富文本营销图过滤（与详情图同规则）**：构建RC widgets时，必须再次确认每张图不含营销内容。流程：详情图下载→OCR过滤营销图→翻译→上传私有图床/对象存储→构建RC。只有通过过滤的图才能进入RC的widgets数组。最末尾固定追加一个通用品牌海报/售后说明 widget；具体 URL 必须从本地私有配置读取，不能硬编码在公开 skill 里。
     - **图片大小**：翻译后的 PNG 主图往往 1MB+，Ozon 俄罗斯服务器下载超时概率很高。必须先用 Pillow 转为 JPG（quality=82，通常可压到 100-250KB），再上传到对俄罗斯可访问的图床。
     - **品牌字典值**：品牌 id=85 必须使用 `dictionary_value_id` 而非手写文本。"Нет бренда" 的具体 ID 必须通过 `/v1/description-category/attribute/values/search` 在运行时动态查询，不要在公开 skill 中硬编码。
   - **#Хештеги (标签规则)**：必须以 `#` 开头，仅包含字母和数字，2个以上单词用下划线 `_` 连接。每个标签最大 30 字符，最多 30 个标签。

## 工作流拆解 (Step-by-Step)

### 1. 数据及图像抓取 (从 1688 / 原始源)
- 提取目标产品在 1688 的 Offer ID，抓取全部商品信息。注意：如果遇到 `product_data.json` 返回缓存 JSON 而不是 HTML (`{"raw_api": {...}}`)，请在 Python 里从 `data['raw_api']` 解析商品信息（如 `title` 和 `price_text`）。
- **详情图营销图筛除（必须执行）**：1688详情图经常混入营销/引流图片，这些图绝对不能上Ozon！下载详情图后必须逐张审查，跳过以下类型：
  1. **带原链接/二维码的图**：含淘宝链接、1688链接、微信号、QQ号、店铺二维码、"扫码下单"等
  2. **店铺引流图**：含"收藏店铺"、"关注有礼"、"加购物车"、"复制链接打开淘宝/1688"等
  3. **平台促销图**：含"满减"、"包邮"、"限时特价"、"双11"、"618"等中国平台促销语
  4. **售后/物流声明图**：纯文字的"关于发货"、"退换货说明"、"温馨提示"等非产品内容图
  5. **尺码表/参数纯文字图**：全是中文文字表格且无产品实物的图（纯参数图）
  6. **证书/资质图**：质检报告、3C认证、CCC认证、ISO证书、专利证书、检测报告、SGS报告、合格证、营业执照等
  审查方法：对每张详情图做OCR（Tesseract chi_sim+eng），匹配关键词（淘宝|天猫|1688|阿里巴巴|微信|QQ|扫码|收藏店铺|关注|复制链接|满减|包邮|限时|特价|发货|退换|售后|温馨提示|旺旺|客服|下单|优惠|促销|活动|领券|加购|好评|返现|红包|拼团|质检|检测报告|3C|CCC|ISO|认证|证书|专利|合格证|营业执照|SGS|检验）。命中的直接跳过不下载/不翻译/不上传。同时用视觉模型（vision_analyze）抽查是否有二维码。宁可误删也不能把营销图或证书图传到Ozon。
- **1688 抓图坑点与突破口 (极深隐藏图)**：有些商品（如玩具、厂货）的图文详情是动态注入的，完全不暴露在页面的初始 DOM 或无头浏览器的默认滚动加载中。此时，**不要被欺骗**。请在原始 HTML 源码中全局搜索 `offerdetails.1688.com` 或 `detailUrl` 包含 `desc` 或 `icoss` 的外接动态 JSON/JS 接口（例如 `itemcdn.tmall.com/1688offer/`）。提取出真正的动态描述请求文件，通过 `requests.get` 拉取后获得带有真实图片 URL 的内容，然后正则提取 `https://cbu01.alicdn.com/img/ibank/...` 即可获得真正的详情长图序列。遇到抓不到详情时，绝对不能毫无排查就直接宣称“只有主图”。
- **严重验证码拦截 (Captcha/HTTP 420)**：如果 `requests` 或无头浏览器完全被 1688 的“验证码拦截”或“安全验证”页面阻断，导致无法获取详情图，**必须指示用户在本地终端运行带界面 (headed) 的 Playwright 脚本（如 `fetch_1688_playwright.py`）**。脚本应包含循环检测逻辑（`while "验证码" in title or "登录" in title:`），弹出真实浏览器让用户手动拖动滑块或扫码，验证通过后程序自动接管抓取。绝不能轻易放弃抓取详情图。
- **OCR 漏判防御 (艺术字陷阱)**：Tesseract OCR (`chi_sim`) 经常对特殊的彩色边框艺术字、手写体、变体字（尤其是卡通玩具类产品）**漏判**，导致判断为“无中文”而跳过翻译。如果产品图包含大量解说或用户坚持指出图中有字，**绝对不要相信 Tesseract 的 False 判断**，直接将提取到的原图过一遍 apipudding [官逆C]Nano banana 2 强行翻译。
- **翻译后中文遗漏二次审查（必须执行）**：每张图翻译完成后，必须对翻译结果再做一轮 Tesseract OCR（chi_sim），检测是否仍有中文残留（`[\u4e00-\u9fff]`）。如果发现残留中文：
  1. 第一次残留：用阿里云 TranslateImage 重新翻译一次
  2. 第二次仍残留：调用 vision_analyze 让大模型判断残留内容是否影响阅读。如果是关键文字（产品名、说明文字），标记为"需人工处理"；如果是装饰性小字或水印残留，可放行但在检查单标注
  3. 绝不能让大段中文产品说明出现在Ozon详情图上
- **翻译 API**：必须使用 `apipudding.com` 直连网关 (Model: `[官逆C]Nano banana 2`，注意有前缀 `[官逆C]`)。代码中的 `translate_image_nano_banana(img_path, output_path)` 仅接受 2 个参数，提示词已硬编码在函数内。Ozon 强制要求主图比例为 3:4！如果 1688 抓取的是 1:1 主图，在调用 API 翻译前，必须先用 Python 图像库（Pillow/CV2）上下填充背景/留白将其改造为 3:4 比例（即 高度 = 宽度 * 4/3），再发给大模型翻译。提示词：「把图片中的中文翻译成俄语。保持3:4比例」。详情图则保持原图（长图）自带比例不变。
- 注意关闭 macOS 本地代理引发的 Python Request 报错 (`s.trust_env = False`)。

### 3. 公共图床中转 (Image Hosting)
- 必须将处理好的图片上传到**对俄罗斯可访问**且**凭据保存在本地私有配置**中的图床/对象存储。公开 skill 只保留选择标准与流程，不能包含真实 upload key、bucket URL、token 或固定资源直链。

### 4. 匹配产品类目与创建 Payload
- 调用 Ozon API 查询目录：`POST https://api-seller.ozon.ru/v1/description-category/tree`。
- **无品牌商品 (Нет бренда)**：切勿直接猜测或硬编码 Brand ID（如 5059062），极易导致 `error_attribute_values_out_of_range` 错误。必须调用 `POST /v1/description-category/attribute/values/search`，传参 `"value": "Нет бренда"` 动态查出当前有效的 `dictionary_value_id`。

### 5. 绑定图床图片至 Ozon
- 将前一步获取的外链图床 URLs 封装，调用 `/v1/product/pictures/import` 或在 `/v3/product/import` 创建时直接传入 `images` 数组。

### 6. 激活产品并验收
- 调用 `/v1/product/unarchive` 解封产品。
- **避坑**：Payload 必须为 `{"product_id": [这里是 Ozon 返回的数字型 Product ID]}`。绝对不能使用 `{"offer_id": [...]}`，否则会报 400 (invalid RestoreItemsRequest.ItemIds) 错误。

### 7. 补充营销详情与富内容 (Rich-Content)
- **查阅确切的属性 ID** (`/v1/description-category/attribute`)：切勿凭经验猜测属性 ID（如把 4194 当作 Комплектация 纯文本提交可能会报错 invalid URL）。必须校验该类目下属性的 `type` 是否为 `String`。
- 通常：`Rich-контент JSON` ID 为 **11254**，`Аннотация` (长描述) ID 为 **4191**。
- **`/v1/product/attributes/update` payload 关键坑点**：`attributes` 数组里的字段名必须是 **`id`**，不是 `attribute_id`。最小可用 body 形如：
  ```json
  {
    "items": [
      {
        "offer_id": "589236742340",
        "attributes": [
          {"id": 9048, "complex_id": 0, "values": [{"value": "Карусель с голосовым управлением №1"}]},
          {"id": 11254, "complex_id": 0, "values": [{"value": "{\"content\":[...],\"version\":0.3}"}]}
        ]
      }
    ]
  }
  ```
- **属性更新任务查询**：`/v1/product/attributes/update` 成功后返回的是 `task_id`，后续同样使用 `POST /v1/product/import/info` 轮询任务状态，直到 `status=imported` 且 `errors=[]`。
- **验收接口（已实测可用）**：用 `POST /v4/product/info/attributes` 验证属性是否真正写入，推荐 body：
  ```json
  {
    "filter": {"offer_id": ["589236742340"], "visibility": "ALL"},
    "last_id": "",
    "limit": 100,
    "sort_dir": "ASC"
  }
  ```
  返回结果中的 `attributes` 列表可直接检查 9048 / 4191 / 11254 / 23171 等是否生效。
- 最终通过 `POST https://api-seller.ozon.ru/v1/product/attributes/update` 将富文本推送至服务器。检查无报错即告全部完成。

### 8. 状态查询与审核坑点
- **查询审核状态及报错原因**：切勿使用经常报 404 的 `/v2/product/info`。当商品在后台显示“不可出售” (Not for sale / Не продается) 时，**必须使用 `POST /v3/product/info/list`** 接口进行查询。Payload 格式为 `{"offer_id": ["your_id"]}`。注意：千万不要同时传 `offer_id` 和 `product_id`，否则会报错 400。
- **玩具类风控拦截**：Ozon 视觉审核极严。如果在提取的返回 json 字典中，`status` 下出现 `declined` 且错误信息提示“儿童商品类别禁止出现烟草制品图像 (В категории Детские товары запрещено изображение табачной продукции)”，说明产品详情图中含有疑似抽烟的元素被 AI 误判。若遇此情况，建议直接在后台删除或将产品归档处理。
