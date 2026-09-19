# 创作地图生成模块

## 输入

- 当前 profile_id；
- 账号创作思路深挖结果及其证据、未知和冲突；
- 当前作品创作思路深挖结果及其证据、未知和冲突；
- 原作品及识别结果；
- 当前有效创作地图（如有）。

## 工作步骤

1. 只整理输入中已有的事实，不重新推测缺失信息；
2. 生成账号层面的长期创作地图；
3. 生成当前单篇作品的创作上下文；
4. 将两类结果分别关联到账号、作品和用户回答证据；
5. 保留未知和冲突；
6. 检查作品创作思路与账号创作思路的一致性；
7. 信息不足或存在关键冲突时返回需要补充或确认的内容；
8. 输出草稿并等待用户确认。

## 输出

在公共输出字段之外输出 creation_map_draft：

- account_creation_map：target_audience、service_scenarios、content_direction_plan；
- work_creation_context：work_id、work_goal、reader_takeaway、must_retain_points、detail_expansion_points、writing_style；
- evidence；
- unknowns；
- conflicts。

work_creation_context 仅适用于当前作品，不自动成为账号长期知识。只有用户确认后的账号地图和作品上下文才可交给优化 Demo。
