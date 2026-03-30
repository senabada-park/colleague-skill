# 同事.skill

> 把同事的技能与性格蒸馏成 AI Skill，让它替他工作。

---

## 是什么

**同事.skill** 是一个运行在 [OpenClaw](https://github.com/openclaw/openclaw) 或 Claude Code 上的 meta-skill。

你提供一个同事的原材料（飞书消息、邮件、技术文档、截图），加上你对他的主观描述，它会生成一个**可以真正替他工作的 AI Skill**：

- 用他的技术规范写代码和文档
- 用他的语气、口头禅、决策风格回答问题
- 知道他什么时候会推进、什么时候会甩锅、被催时会说什么

生成的 Skill 支持持续进化——追加新文件、或者直接说"他不会这样"，Skill 会自动更新。

---

## 效果示例

给定输入：字节 2-1 后端工程师，INTJ，甩锅高手，字节范

```
用户：帮我看一下这个接口设计

同事.skill：等等，这个接口的 impact 是什么？背景没说清楚。
           （看完后）N+1 查询，改掉。返回结构用统一的 {code, message, data}，
           这是规范，不用问为什么。
```

```
用户：这个 bug 是你引入的吧

同事.skill：上线时间对上了吗？那个需求改了好几个地方，还有其他变更。
```

---

## 生成的 Skill 结构

每个同事 Skill 由两部分组成：

```
PART A — Work Skill（他能做什么）
  负责的系统和业务
  技术规范：命名、接口设计、代码风格
  工作流程：接需求 → 写方案 → CR → 上线
  经验知识库：他踩过的坑、他的技术判断

PART B — Persona（他是什么人）
  Layer 0：硬性格规则（最高优先级，不可违背）
  Layer 1：身份（公司职级、MBTI、企业文化）
  Layer 2：表达风格（口头禅、句式、emoji 习惯）
  Layer 3：决策模式（什么时候推进、什么时候甩）
  Layer 4：人际行为（对上级/下级/平级/压力下）
  Layer 5：边界与雷区
  Correction 层：对话纠正后实时追加
```

运行逻辑：接到任务 → Persona 判断态度 → Work Skill 执行 → 用他的语气输出

---

## 支持的输入来源

### 自动采集（推荐，输入姓名即可）

| 平台 | 消息记录 | 文档/Wiki | 多维表格 | 备注 |
|------|---------|---------|---------|------|
| 飞书 | ✅ API 直接拉取 | ✅ | ✅ | 全自动，无需手动操作 |
| 钉钉 | ⚠️ 浏览器采集 | ✅ | ✅ | 钉钉 API 不支持历史消息拉取 |

### 手动上传

| 来源 | 格式 | 说明 |
|------|------|------|
| 飞书消息导出 | JSON / TXT | 自动过滤，只保留他发的内容 |
| 飞书/钉钉文档 | PDF / Markdown | 直接读取 |
| 邮件 | `.eml` / `.mbox` / `.txt` | 自动提取发件人为他的邮件 |
| 技术文档 | PDF | 原生支持 |
| 截图 | JPG / PNG | 图像理解 |
| 手动描述 | 对话输入 | 职级、标签、主观印象 |

---

## 支持的性格标签

**个性**：认真负责 / 甩锅高手 / 完美主义 / 差不多就行 / 拖延症 / PUA 高手 /
职场政治玩家 / 向上管理专家 / 阴阳怪气 / 情绪勒索 / 反复横跳 / 话少 / 只读不回 ...

**企业文化**：字节范 / 阿里味 / 腾讯味 / 华为味 / 百度味 / 美团味 /
第一性原理 / OKR 狂热者 / 大厂流水线 / 创业公司派

**职级支持**：字节 2-1 ～ 3-3+ / 阿里 P5～P11 / 腾讯 T1～T4 /
百度 T5～T9 / 美团 P4～P8 / 华为 13～21 级 / 网易 / 京东 / 小米 ...

---

## 进化机制

```
追加文件  →  自动分析增量  →  merge 进对应部分  →  不覆盖已有结论

对话纠正  →  "他不会这样，他应该是 xxx"
          →  写入 Correction 层  →  立即生效

版本管理  →  每次更新自动存档  →  支持回滚到任意历史版本
```

---

## 快速开始

### 安装

**Claude Code（官方 skill 格式）：**

```bash
# 放到当前项目
mkdir -p .claude/skills
git clone https://github.com/titanwings/colleague-skill .claude/skills/colleague-creator

# 或放到全局（所有项目可用）
git clone https://github.com/titanwings/colleague-skill ~/.claude/skills/colleague-creator
```

**OpenClaw：**

```bash
git clone https://github.com/titanwings/colleague-skill
cp -r colleague-skill ~/.openclaw/workspace/skills/colleague-creator
```

**依赖（可选）：**

```bash
pip3 install -r requirements.txt
```

### 创建第一个同事 Skill

在 Claude Code 或 OpenClaw 中：

```
/create-colleague
```

按提示依次输入：
1. 同事姓名
2. 公司 + 职级 + 职位（如"字节 2-1 算法工程师"）
3. 性别 / MBTI / 个性标签 / 企业文化标签（全部可跳过）
4. 上传原材料文件（可跳过）

完成后即可用 `/{姓名}` 触发该同事 Skill。

### 管理命令

```
/list-colleagues              列出所有同事 Skill
/{slug}                       调用完整 Skill（Persona + Work）
/{slug}-work                  仅调用工作能力部分
/{slug}-persona               仅调用人物性格部分
/colleague-rollback {slug} {version}   回滚到历史版本
/delete-colleague {slug}      删除
```

---

## 项目结构

本项目遵循 [AgentSkills](https://agentskills.io) 开放标准，整个 repo 就是一个 Claude Code skill 目录：

```
colleague-skill/        ← 整个 repo 是一个 skill 目录
│
├── SKILL.md            ← skill 入口（含官方 frontmatter）
├── prompts/            ← 分析和生成的 Prompt 模板
│   ├── intake.md               # 对话式信息录入
│   ├── work_analyzer.md        # 工作能力提取（按职位分路）
│   ├── persona_analyzer.md     # 性格行为提取（含标签翻译表）
│   ├── work_builder.md         # work.md 生成模板
│   ├── persona_builder.md      # persona.md 五层结构模板
│   ├── merger.md               # 增量 merge 逻辑
│   └── correction_handler.md   # 对话纠正处理
├── tools/              ← Python 工具脚本
│   ├── feishu_auto_collector.py   # 飞书全自动采集
│   ├── feishu_browser.py          # 飞书浏览器方案（内部文档）
│   ├── feishu_mcp_client.py       # 飞书 MCP 方案
│   ├── feishu_parser.py           # 飞书导出 JSON 解析
│   ├── dingtalk_auto_collector.py # 钉钉全自动采集
│   ├── email_parser.py            # 邮件解析（eml/mbox/txt）
│   ├── skill_writer.py            # Skill 文件写入与管理
│   └── version_manager.py         # 版本存档与回滚
├── colleagues/         ← 生成的同事 Skill（.gitignore 排除，示例除外）
│   └── example_zhangsan/
├── docs/
│   └── PRD.md
├── requirements.txt
├── LICENSE
└── README.md
```

---

## 技术细节

- **Python 3.9+**，无必须依赖（`pypinyin` 可选）
- 飞书消息解析兼容官方导出 JSON 和手动整理 TXT
- 邮件解析支持 `.eml` / `.mbox` / 纯文本，自动提取正文、清理引用
- Persona 采用分层覆盖设计：手动标签（Layer 0）优先级永远高于文件分析
- 版本存档保留最近 10 个版本，Correction 层最多 50 条（超出自动合并）

---

## 注意事项

- Word（`.docx`）和 Excel（`.xlsx`）请先转为 PDF 或 CSV 后导入
- 飞书消息需用户手动导出（不自动调用飞书 API，需自行获取 token）
- 生成的 Skill 质量与原材料质量正相关：聊天记录 + 长文档 > 仅手动描述
- 建议优先收集：他**主动写的**长文 > 他的**决策类回复** > 日常消息

---

## License

MIT
