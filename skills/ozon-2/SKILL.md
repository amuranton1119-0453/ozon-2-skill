---
name: Ozon 2店运营与上架
description: >-
  通过已连接的 ozon2 连接器（Ozon Seller API）查询或操作 2 店「Лягушка в
  Море」时使用：查商品、价格、库存和待发货订单，改价、调库存、按类目属性上架，以及缺货风险、价格红标、内容质量、ABC 分级等分析。严禁用于 1
  店「Фрогги」。
---
# Ozon 2店运营与上架

用已连接的 **ozon2** 连接器调用 [Ozon Seller API](https://docs.ozon.ru/api/seller/)，操作 2 店「Лягушка в Море」。先用 Skill 加载本技能，再用连接器工具，不要改用 1 店的后台、密钥或浏览器配置。

## 适用范围

- 查 2 店商品、价格、库存、待组装的 FBS/rFBS 订单。
- 在店主明确批准后改价、调库存、上架或更新卡片。
- 跑连接器里的现成分析流程：缺货风险、价格竞争力、扣费后单笔利润、目录快照、内容质量、ABC 分级、评价趋势。
- **不适用**：1 店「Фрогги」、其他平台、或还没批准的写入（改价、调库存、上架、归档、删除、花费、对外发消息）。

## 硬性约束

1. **只打 2 店。** 先调用 `ozon_check_auth` 和 `ozon_list_cabinets`，确认当前柜组是 2 店。禁止用本连接器读或改 1 店。柜组配置若可能混用，停下问店主，不要自行切换去碰另一家店。
2. **读可以先做，写必须先批准。** `ozon_call_method`、`ozon_get_*`、`ozon_fetch_all` 是只读。`ozon_write_method`、`ozon_write_raw`、`ozon_set_price`、`ozon_delete_method` 等写入，必须店主在当次对话里明确同意后，才设 `confirm_write=true`。删除或不可逆操作还要 `i_understand_this_modifies_data=true`。没批准就不要发。
3. **写入优先由 2 店运营助手执行。** 其他助手默认只读；只有店主点名时才在本连接器上写入。
4. **不要把 API 密钥写进聊天。** 不要调用 `ozon_add_cabinet` 或 `ozon_set_key` 来粘贴密钥。需要换密钥时，让店主用安装程序或安全表单，不要在对话里收密钥。
5. **不确定就标「待确认」或「不确定」，不要编。** 必填项仍是待确认的，禁止提交上架。可选字段可以留空。目录里标注 UNVERIFIED 的接口，先用 `ozon_describe_method` 对照官方文档，确认请求体后再调用。

## 常用工具怎么选

先查目录，再调用，不要硬猜 path。

- 找接口：`ozon_search_methods`（中俄英关键词都可以）→ `ozon_describe_method` 看参数和文档链接。
- 只读一条：`ozon_call_method`（传 `operation_id`）。
- 快捷只读：`ozon_get_products`、`ozon_get_prices`、`ozon_get_stocks`、`ozon_get_fbs_unfulfilled`。翻页时商品列表读 `result.total_items`，不要读已于 2026-11-23 移除的 `result.total`。需要全量时用 `ozon_fetch_all`。
- 写入一条：`ozon_write_method`（传已批准的 `operation_id` 和 `confirm_write=true`）。改单个 offer 的价格可用 `ozon_set_price`（价格是字符串；Ozon 大约限制每商品每小时 10 次调价）。
- 分析：`ozon_list_workflows` → `ozon_get_workflow` 取步骤，再按步骤调用只读接口。现成流程：
  - `oos_risk_analysis`：即将缺货
  - `pricing_analysis`：价格指数红标
  - `unit_economics`：扣费后单笔利润
  - `catalog_sync`：商品+库存+价格快照
  - `content_quality_audit`：内容评分偏低的卡片
  - `abc_analysis`：按销售额做 ABC 分级
  - `reviews_pulse`：评价情绪和卖家评分趋势

## 上架流程（必须先得到店主批准）

整体顺序：对类目和必填属性 → 备齐公开图片链接和文案 → 把草稿给店主确认 → 提交导入 → 轮询结果 → 再视需要传图、设库存。

1. **确认商店和批准。** 复述将上架的店铺是 2 店、商品名称、类目、各 SKU 价格与库存、重量尺寸。等店主明确说可以提交。
2. **选类目。** 用 `ozon_category_tree`（`/v1/description-category/tree`）选定 `description_category_id` 和 `type_id`。不要用已弃用的 `/v2/category/tree`。
3. **拉必填属性。** 用 `ozon_category_attributes`（`/v1/description-category/attribute`）列出该类型的属性。枚举值用 `ozon_search_attributes_dictionary` 搜索，不要手填没出现在字典里的 ID。无品牌时在字典里搜「Нет бренда」一类取值，不要编品牌 ID，标题和图片上也不要出现未授权品牌。
4. **准备图片。** 本连接器不能直接上传本地文件。主图和详情图必须是 Ozon 能访问的 **公开 URL**。补图或换图用 `ozon_post_v2_product_pictures_import`（`/v2/product/pictures/import`），再用 `ozon_check_image_uploading_status`（`/v2/product/pictures/info`）查看状态。没有可用的公开链接时，停下来让店主或 2 店助手在卖家后台传图，不要为了绕过上传限制改用匿名图床。
5. **组导入稿。** 用 `ozon_product_import`（`POST /v3/product/import`），每请求最多 100 个商品，异步返回 `task_id`。稿子里应包含 offer_id、名称、描述、类目与类型、属性、公开图片 URL、价格、包装重量和长宽高。视频可以后补；除视频外的必填项必须填完。必填未知就标「待确认」并停止提交。
6. **提交并轮询。** 店主批准后，用 `ozon_write_method` 调用 `ozon_product_import`。然后用只读的 `ozon_product_import_info`（`/v1/product/import/info`）轮询任务，直到成功、失败或超时。把失败原因原样报给店主，不要悄悄改稿重提，除非店主要求重试。
7. **库存（rFBS/FBS）。** 导入成功且店主同意设库存后，再写库存（目录中为 `ozon_update_stocks`，`/v1/product/import/stocks`）。仓库必须用该店已配置的仓库，不要猜仓库 ID。先查 `ozon_get_stocks` 核对。
8. **验收。** 用 `ozon_product_info_list`、`ozon_product_visibility_info`（`/v1/product/visibility/info`）和 `ozon_product_rating`（`/v1/product/rating-by-sku`）确认卡片状态、可见性和内容分，把缺口告诉店主。

## 改价与归档

- 改价：先把计算过程和拟改价格发给店主，同意后再用 `ozon_set_price` 或对应写入接口。价格字段用字符串。
- 下架归档：`ozon_product_archive`。永久删除只能针对已归档且无 SKU 的商品（`ozon_product_delete`），必须单独批准，且按破坏性接口确认。

## 查订单

待组装的自发货订单用 `ozon_get_fbs_unfulfilled`。发货、打单或改物流属于写入，同样要店主事先批准。

## 失败时

- 鉴权失败：只报告未就绪，不要把密钥或缺失变量的具体值说出来；请店主检查 2 店连接器。
- 写入被拒绝或任务失败：把接口返回的错误原文转述给店主，停止，等待指示。
- 接口不在目录里：用 `ozon_search_methods` 再搜；仍没有就对照官方文档。不要为了写入去调用来路不明的 host。
