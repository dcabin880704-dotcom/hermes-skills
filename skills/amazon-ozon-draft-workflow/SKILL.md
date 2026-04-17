---
name: amazon-ozon-draft-workflow
description: 在 ~/ozon_pipeline 中推进 Amazon 采集结果到 Ozon draft payload 的本地开发与验证流程。覆盖 TDD、run_import 接线、web_meta 桥接、落盘校验与真实样本验证。
category: ecommerce
tags: [amazon, ozon, draft, tdd, unittest, ecommerce]
---

# amazon-ozon-draft-workflow

当用户要求继续处理 `~/ozon_pipeline` 里的 Amazon → Ozon draft 流程时，使用这个技能。

## 适用场景
- 用户要继续做 Amazon 商品采集后的 Ozon draft 生成
- 需要检查 `publish_units -> ozon_drafts` 接线
- 需要在 `tasks/run_import.py` 中落盘 `ozon_drafts.json` / `manifest.json`
- 需要用真实 Amazon 商品验证 `products/amazon_*` 产物

## 必守约束
1. 用户如果说不要看飞书/队列，就不要擅自切回 Feishu 线。
2. 不要把 Amazon 逻辑硬塞进 1688 脚本；保持 `source adapter -> normalized product -> publish_units -> ozon_drafts` 分层。
3. Ozon 变体必须是“每个变体一个独立 SKU/offer_id”，不能把变体塞进 `complex_attributes`。
4. 不能用 `pytest`；本环境必须用：
   - `python3 -m unittest ...`
5. 先看真实 failing test / failing import，再改代码；不要跳过根因盲改。

## 已验证的关键文件
- `~/ozon_pipeline/tasks/run_import.py`
- `~/ozon_pipeline/pipeline/ozon_drafts.py`
- `~/ozon_pipeline/tests/test_run_import.py`
- `~/ozon_pipeline/tests/test_ozon_drafts.py`
- `~/ozon_pipeline/tests/test_publish_units.py`
- `~/ozon_pipeline/tests/test_source_amazon_normalize.py`
- `~/ozon_pipeline/products/*/web_meta.json`

## 当前推荐工作流

### 1. 先读代码和测试
优先读取：
- `tasks/run_import.py`
- `tests/test_run_import.py`
- `pipeline/ozon_drafts.py`
- `products/*/web_meta.json`

目标：确认当前缺口到底是 import-time 错误、builder 输入缺失，还是主流程没接线。

### 2. 先补/看失败测试，再实现
推荐先跑：
```bash
python3 -m unittest tests.test_run_import -v
```
如果要做跨层回归，跑：
```bash
python3 -m unittest tests.test_run_import tests.test_ozon_drafts tests.test_publish_units -v
```

### 3. run_import 必须具备的行为
`tasks/run_import.py` 应完成：
1. 下载图片 / 建立 workdir
2. 生成 `publish_units.json`
3. 读取可用的 `web_meta.json`
4. 计算 pricing
5. 满足前提时调用 `build_ozon_drafts(...)`
6. 写出 `ozon_drafts.json`
7. 写出 `manifest.json`

## draft 生成的最小前提
只有以下条件成立时才生成 drafts：
- `pricing` 存在
- `description_category_id` 存在
- `type_id` 存在

`category_mapping` 最小结构：
```json
{
  "description_category_id": 17039636,
  "type_id": 94682,
  "category_label": "Одежда / Футболки"
}
```

`provided_attributes` 通常来自：
```json
web_meta["ozon_attributes"]
```

## 真实接线要点
- `WEB_META_FILES` 不要硬编码单一路径；优先扫描：
  - `PRODUCTS_DIR.glob("*/web_meta.json")`
- `build_ozon_drafts(...)` 需要传入：
  - `publish_units`
  - `category_mapping`
  - `required_attribute_defs=web_meta.get("required_attribute_defs") or None`
  - `provided_attributes=web_meta.get("ozon_attributes") or {}`
  - `pricing`

## 必查输出文件
真实或临时 workdir 里至少要核对：
- `product.json`
- `publish_units.json`
- `ozon_drafts.json`
- `manifest.json`
- 以及 `pricing.json` 或 `pricing_blocked.json`

## 已知坑点
1. `python` 命令可能不存在；必须显式用 `python3`。
2. `pytest` 可能未安装；不要再尝试 `python3 -m pytest ...`。
3. 当前目录未必是 git repo，不能依赖 `git diff/log`。
4. `web_meta` 可能只有历史样本，真实 Amazon workdir 可能还没生成，需要主动跑真实输入验证。
5. 当前 `load_web_meta()` 如果只是“取第一个文件”，这是临时桥接策略；后续如出现错配，要收敛为更明确的匹配规则。

## 验证标准
完成一次实现或修复后，至少验证：
1. `python3 -m unittest tests.test_run_import tests.test_ozon_drafts tests.test_publish_units -v` 通过
2. `manifest.json` 里的 `publish_units_count` 与 `ozon_drafts_count` 正确
3. `ozon_drafts.json` 实际存在且非空（在满足前提时）
4. 对真实 Amazon 商品跑完后，`~/ozon_pipeline/products/amazon_*` 下能看到完整产物

## 真实样本验证的收尾动作
当测试闭环通过后，不要停在“理论已接通”。继续：
1. 用真实 Amazon 商品执行 `tasks/run_import.py`
2. 找到真实 `products/amazon_*` workdir
3. 检查 `offer_id`、`variant_source_id`、`source_url`、pricing、category mapping 和 attributes 是否合理

## 建议回答风格
- 直接读文件、跑测试、改代码
- 先给结论，再给证据
- 不要只讲计划，不执行
