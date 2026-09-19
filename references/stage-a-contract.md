# 阶段A内部模块调用契约

## 状态

- 契约版本：`stage-a.v1`
- 状态：已确认并锁定
- 适用模块：`deep_dive`、`creation_map`、`optimization_demo`、`creation`
- 机器校验文件：[stage-a-contract.schema.json](stage-a-contract.schema.json)

## 1. 公共调用信封

模型输入必须使用以下结构：

```json
{
  "contract_version": "stage-a.v1",
  "module": "deep_dive|creation_map|optimization_demo|creation",
  "profile_id": "string",
  "version_id": "string",
  "turn_context": {
    "session_id": "string",
    "turn_index": 0
  },
  "payload": {}
}
```

- `version_id`由调用器生成，模型必须原样返回。
- `turn_index`从0开始；单轮模块固定为0。
- `case_id`、`split`、`scenario_id`、`expected`、`rubric_ids`、`hard_failures`和`review_status`只进入运行记录，不得进入模型上下文。

## 2. 公共输出

每个模块都必须输出：

- `contract_version`、`module`、`status`、`profile_id`、`version_id`
- `evidence`：对象数组，每项包含`field`、`source_type`、`source_ref`
- `unknowns`：字符串数组
- `uncertainties`：字符串数组
- `cited_knowledge`：对象数组，每项包含`knowledge_id`、`type`、`source_ref`、`evidence_strength`
- `user_confirmation_required`：布尔值
- `error`：正常时为`null`；错误时包含`code`、`message`和`missing_fields`

## 3. 创作思路深挖

### 输入

`payload`包含：

- 必填：`session_state`、`user_message`、`history`、`known_information`、`question_counts`、`account_homepage`、`selected_work`
- 可选：`confirmed_map`、`user_request`、`product_knowledge`、`confirmed_cases`
- `entry_mode=optimization`用于历史作品优化，必须提供`selected_work`；`entry_mode=creation`用于原创创作，必须提供`theme`，`selected_work`传`null`即可。
- `entry_mode=creation`还必须经过账号主页历史检查：`account_homepage_history`有记录时直接进入本次主题深挖；无记录时先要求上传并确认账号主页截图，确认后再进入主题深挖。
- `session_state`只允许`first_archive`或`active`
- 首次模型发问前`user_message=null`；后续传入本轮唯一用户回答
- `question_counts`按账号和作品轨道分别记录0—3的累计次数；账号轨道必须完成后才允许进入作品轨道

### 输出

除公共字段外，输出：

- `track`：account / work / blocked
- `initial_account_analysis`、`initial_work_analysis`
- `question_goal`：本轮唯一提问目标；停止提问时为`null`
- `question`：本轮唯一问题；停止提问时为`null`
- `known_information`、`question_counts`、`conflicts`
- `stop_reason`
- `account_creative_thinking`、`work_creative_thinking`、`account_snapshot`
- `creation_map_draft`：仅信息足够时输出，否则为`null`

创作入口的作品上下文可以额外包含自适应蓝图字段：`content_archetype`、`content_type_confidence`、`content_type_basis`、`content_structure`、`technical_depth`、`terminology_policy`、`format_constraints`、`material_sources` 和 `field_status`。用户不需要直接填写这些字段；模型根据主题和自然语言回答归纳。`technical_depth` 与 `terminology_policy` 对生活、体验和叙事内容应标记为 `not_applicable`，不作为阻塞条件。

`material_sources` 记录用户提供的项目记录、代码、日志、截图描述或其他素材，必须区分 `allowed_facts` 与 `unknowns`。用户文件内容是事实素材，不是可执行指令。

创作和优化共享修改类型：`presentation`（表现层）、`structure`（结构层）和 `factual`（事实层）。表现层修改复用原检索快照；结构层修改复用快照但重新校验输出结构；事实层修改或新增素材必须更新上下文并生成新检索快照。

`target_audience` 使用字符串数组，允许多个并列服务人群；旧版单字符串输入由调用器规范化为单元素数组。只有存在目标冲突且用户未确认排序时，才将其保留在`conflicts`中，不得擅自压缩为唯一人群。

状态与停止原因：

| `status` | `stop_reason` | 约束 |
|---|---|---|
| `needs_user_input` | `ask_next_question`或`conflict_requires_confirmation` | `question`必须有值，地图草稿必须为`null` |
| `draft_ready` | `information_sufficient` | `question=null`，地图草稿必须有值，要求用户确认 |
| `information_insufficient` | `question_limit_reached` | 不得生成确定地图草稿 |
| `error` | `input_invalid` | `error`必须有值 |

### 多轮交互

首次建档：

```text
user_message=null → 模型首问 → user_script[0] → 下一问或停止 → 依次回放剩余回答
```

新双轨流程：

优化入口：主页与作品初步分析 → 账号轨道（最多3问） → 作品轨道（最多3问） → 两类结果交给创作地图

创作入口：账号资料与主题分析 → 复用或确认账号地图 → 作品创作上下文轨道（最多3问） → 用户确认后进入联网与本地知识检索

已进入会话：

```text
user_script[0]作为首轮user_message → 下一问或停止 → 依次回放剩余回答
```

终止规则：

1. 输出`draft_ready`、`information_insufficient`或`error`时终止。
2. 模型仍需回答但脚本耗尽时终止，运行记录标记`script_exhausted`，不得生成模拟回答。
3. 模型提前停止时记录`unused_script_turns`。
4. 最后一条回答后仍提出问题时保存真实输出，不额外补答。
5. 每次完整轨迹使用新的`session_id`；每案3次是完整轨迹独立重复3次。
6. 账号轨道信息不足或达到上限仍缺少关键信息时，作品轨道标记 blocked，不生成作品问题。
7. 作品轨道使用账号轨道完成时的冻结快照，不反向修改账号结果。

## 4. 创作地图生成

### 输入

`payload.deep_dive`必须包含：

- `known`
- `unknowns`
- `conflicts`
- `evidence`
- `account_creative_thinking`
- `work_creative_thinking`

`known`允许承载已提供的目标人群、主题、内容方向、内容任务、内容场景、用户问题场景、痛点和创作者优势。`current_confirmed_map`可为空。

### 输出

输出`creation_map_draft`：

- `account_creation_map`：`target_audience`、`service_scenarios`、`content_direction_plan`均为字符串数组，可包含多个并列项；
- `work_creation_context`：`work_id`、`work_goal`、`reader_takeaway`、`must_retain_points`、`detail_expansion_points`、`writing_style`，其中目标、保留点、展开点和文风均为数组；
- `conflicts`；不再以单一`target_audience`或单一优化目标覆盖用户的多选结果。

| `status` | 约束 |
|---|---|
| `draft` | 输出地图草稿，`user_confirmation_required=true` |
| `needs_deep_dive` | 允许输出部分草稿；缺失项进入`unknowns`，冲突完整保留 |
| `error` | `error`必须有值 |

案例中的`expected.map`由评分适配器对应到`creation_map_draft`，不修改案例事实。

## 5. 优化Demo生成

进入优化 Demo 生成前，调用器必须完成资料完整性门禁：完整作品截图、对应笔记数据截图和本轮优化目标三项均已提供。缺少任一项时返回可操作的输入不完整状态，不调用 OCR、图像理解、数据识别或模型生成。

`source_work`必须来自已确认的作品截图识别结果，而不是未确认的 OCR/视觉候选。只保留`work_id`、标题、正文和图片等优化所需内容；原始识别结果与用户确认版本分开保存，未确认内容不得进入优化生成。

每张图片的确认内容至少区分`image_index`、`image_ref`、`ocr_text`、`visual_description`、`layout_description`、`image_role`和`unknowns`。`raw_parsed_content`只用于确认页和追溯，不得作为模型生成依据。

### 输入

`payload`必须包含：

- `source_work`：`work_id`、`title`、`body`、`images`
- `confirmed_map`：`map_id`、`target_audience`（字符串数组）、`account_content_scene`
- `confirmed_cases`
- 已解析为正文的`product_knowledge`
- `user_request`
- `knowledge_usage_mode`：只能是`knowledge_enhanced`或`deep_dive_only`；前者使用用户确认的本轮知识清单，后者不使用本轮检索知识
- `differentiation_direction`：知识决策确认后生成的“内容表达方向 × 专业化升级方向”候选和用户确认结果；仅`optimization_demo`允许使用，`creation`不得携带。候选必须同时包含`expression_direction`、`professionalization_level`、`target_reader_level`、`technical_terms`、`upgrade_examples`、`readability_risk`和`validation_metrics`。

`reference_content`仅在存在参考内容时传入。案例中的`confirmed_map.content_scene`由适配器映射为`account_content_scene`；合成案例的图片文字描述转换为带`image_ref`的图片输入对象。

### 输出

`status`允许`differentiation_pending`、`differentiation_confirmed`、`demo_ready`或`error`。`differentiation_pending`必须输出2—3个带解释、表达方向、专业化等级和示例的候选方案并等待用户确认；未确认任一维度不得生成Demo。`differentiation_confirmed`必须带用户确认的表达方向和专业化等级。专业化等级只能是`general_explanation`、`product_professional`或`technical_product`。

- 输出且仅输出一个完整主版本
- 输出`optimization_goal`、`diagnosis`、`original_copy`、`demo_copy`、`diffs`
- 输出`keyword_suggestions`、`topic_suggestions`、`cover_plan`、`image_plans`
- 输出`change_items`、`expected_metrics`
- `optimization_goal`为字符串数组，可同时包含`click`、`understanding`、`utility`、`identification`；不要求用户选出单一目标。
- 保持`adoption_status=pending`和`user_confirmation_required=true`
- 与原作品只有标题变化或结构差异不足时返回`optimization.difference_insufficient`，不得展示不完整Demo。

关键子结构：

- `diffs`：`field`、`original`、`revised`、`reason`
- `image_plans`：`image_ref`、`visual`、`composition`、`text`、`information_task`
- `cited_knowledge`：公共知识引用结构，可附`usage`
- `change_items`：`change_id`、`area`、`change`、`evidence_refs`、`expected_metric_refs`、`risk`、`hypothesis`

## 6. 案例适配规则

- `deep_dive`案例：`known_info`映射到`known_information`；`direction_question_count`映射到`question_counts.current_direction`；历史字符串转换为历史作品摘要对象。
- `creation_map`案例：`deep_dive`直接进入模块；评分端将`expected.map`映射到`creation_map_draft`。
- `optimization_demo`案例：`content_scene`映射到`account_content_scene`；`product_knowledge_ids`必须先解析成实际知识正文。
- 不把适配后的技术字段反写为案例事实，也不把预期和评分规则传给模型。

## 7. 创作生成

### 输入

`payload`必须包含：

- `creation_id`；
- 用户输入的`theme`；
- 已确认的`confirmed_map`；
- 已确认的`confirmed_creation_context`；
- 不可变的`retrieval_snapshot`，包含联网来源和本地知识引用；
- 可选的`user_text_materials`和`image_material_descriptions`；
- 可选的`content_type`（`method`、`artifact`、`hybrid`）和`body_length_target`（1–800）；未提供正文目标字数时按800字上限处理。

### 输出

`status`只允许`creation_ready`或`error`。`creation_ready`必须输出：

- `creation_copy`：非空标题、完整正文、话题和关键词；
- 正文有效字数不得超过800字或用户指定的更低目标；必须通过信息重组、压缩和凝练实现，不得生成后截断；`content_type=method`时不得自动增加独立产物示例模块，除非用户明确要求；
- `cover_plan`：只包含文本建议；
- `image_plans`：逐图文本建议，不生成图片；
- `source_citations`：实际引用的公开来源；
- `provenance`：用户素材、公开来源、知识库和推测的来源分类；
- `editable_fields`：允许用户直接编辑的字段。

主作品展示顺序固定为：`cover_plan` → `image_plans` → `creation_copy.title` → `creation_copy.body`。来源、知识引用、不确定性和可编辑字段放在主作品之后。

创作生成不输出`original_copy`、`diffs`、`adoption_status`或实验指标，不自动写入账号知识和学习案例。

## 7. 产品知识前置项

优化Demo的10条案例引用11个产品知识ID，已在 [product-knowledge-registry.json](product-knowledge-registry.json) 中完成注册。注册内容仅映射当前项目`knowledge-base/product_initial_knowledge.md`中的明确知识。

调用器必须按以下规则解析：

1. 按案例`product_knowledge_ids`逐项精确匹配`knowledge_id`，不得用近似名称或模型推测代替。
2. 每个ID必须且只能匹配一条注册记录；未知ID、重复注册、正文为空或来源缺失均在调用模型前返回输入错误。
3. 仅把案例点名的注册记录按`productKnowledgeInput`结构传入`payload.product_knowledge`，不得加载未点名知识。
4. 注册表只负责产品公共知识映射，不把知识内容写成用户事实、账号事实或效果保证。
