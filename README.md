# 小红书新手运营 AI 助手

这是一个用于辅助小红书图文创作者的 Codex Skill，支持：

- 创作主题深挖
- 创作地图生成
- 原创图文创作
- 历史作品优化 Demo

## 不适用范围

- 视频内容创作
- 自动爬取小红书数据
- 跨账号复用私有知识
- 承诺固定曝光、点赞、收藏或涨粉结果

本项目不是小红书官方工具，也不代表小红书官方立场。

## 安装

将本仓库内容复制到：

```text
~/.codex/skills/xiaohongshu-new-creator-assistant/
```

安装后，Codex 可以根据 `SKILL.md` 自动识别并调用该 Skill。

## 目录结构

- `SKILL.md`：Skill 主入口和路由规则
- `agents/openai.yaml`：界面展示信息和默认提示
- `references/`：创作、深挖、地图、优化、知识库和结构化契约文件

公开知识库文件：

- `references/creative_deep_dive_rules.md`：模型自主深挖创作思路规则
- `references/product_initial_knowledge.md`：产品初始知识库

## 隐私说明

本公开包不包含真实 API Key、真实账号资料、用户截图、私有案例或运行结果。使用者应自行负责输入材料的隐私和版权合规。

## 当前状态

这是一个可独立分发的 Skill 包，不包含本项目的 Agent Runtime、测试运行器、私有案例存储或完整研发代码。

## 许可证

许可证正在由项目维护者确认。未正式添加许可证前，不应将本仓库内容视为授予了复制、修改或商业使用权。
