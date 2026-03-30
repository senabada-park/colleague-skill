# 同事.skill 安装说明

---

## 选择你的平台

### A. Claude Code（推荐）

本项目遵循官方 [AgentSkills](https://agentskills.io) 标准，整个 repo 就是 skill 目录。克隆到 Claude skills 目录即可：

```bash
# 方式 1：放到当前项目的 .claude/skills/ 目录下
mkdir -p .claude/skills
git clone https://github.com/titanwings/colleague-skill .claude/skills/colleague-creator

# 方式 2：放到全局目录，所有项目都能用
git clone https://github.com/titanwings/colleague-skill ~/.claude/skills/colleague-creator
```

然后在 Claude Code 中说 `/create-colleague` 即可启动。

生成的同事 Skill 默认写入 `./colleagues/` 目录。

---

### B. OpenClaw

```bash
# 克隆到 OpenClaw 的 skills 目录
git clone https://github.com/titanwings/colleague-skill ~/.openclaw/workspace/skills/colleague-creator
```

重启 OpenClaw session，说 `/create-colleague` 启动。

---

## 依赖安装

```bash
# 基础（Python 3.9+）
pip3 install pypinyin        # 中文姓名转拼音 slug（可选但推荐）

# 飞书浏览器方案（内部文档/需要登录权限的文档）
pip3 install playwright
playwright install chromium  # 仅需安装 chromium，不需要完整 Chrome

# 飞书 MCP 方案（公司授权文档，通过 App Token 读取）
npm install -g feishu-mcp    # 需要 Node.js 16+

# 其他格式支持（可选）
pip3 install python-docx     # Word .docx 转文本
pip3 install openpyxl        # Excel .xlsx 转 CSV
```

### 平台方案选择指南

| 场景 | 推荐方案 |
|------|---------|
| 飞书用户，有 App 权限 | `feishu_auto_collector.py` |
| 飞书内部文档（无 App 权限）| `feishu_browser.py` |
| 飞书手动指定链接 | `feishu_mcp_client.py` |
| 钉钉用户 | `dingtalk_auto_collector.py` |
| 钉钉消息采集失败 | 手动截图 → 上传图片 |

**飞书自动采集初始化**：
```bash
python3 tools/feishu_auto_collector.py --setup
# 输入飞书开放平台的 App ID 和 App Secret
```

**钉钉自动采集初始化**：
```bash
python3 tools/dingtalk_auto_collector.py --setup
# 输入钉钉开放平台的 AppKey 和 AppSecret
# 首次运行加 --show-browser 参数以完成钉钉登录
```

**飞书 MCP 初始化**（手动指定链接时使用）：
```bash
python3 tools/feishu_mcp_client.py --setup
```

**飞书浏览器方案**（首次使用会弹窗登录，之后自动复用登录态）：
```bash
python3 tools/feishu_browser.py \
  --url "https://xxx.feishu.cn/wiki/xxx" \
  --show-browser    # 首次使用加这个参数，登录后不再需要
```

---

## 快速验证

```bash
cd ~/.claude/skills/colleague-creator   # 或你的项目 .claude/skills/colleague-creator

# 测试飞书解析器
python3 tools/feishu_parser.py --help

# 测试邮件解析器
python3 tools/email_parser.py --help

# 列出已有同事 Skill
python3 tools/skill_writer.py --action list --base-dir ./colleagues
```

---

## 目录结构说明

本项目整个 repo 就是一个 skill 目录（AgentSkills 标准格式）：

```
colleague-skill/        ← clone 到 .claude/skills/colleague-creator/
├── SKILL.md            # skill 入口（官方 frontmatter）
├── prompts/            # 分析和生成的 Prompt 模板
├── tools/              # Python 工具脚本
├── docs/               # 文档（PRD 等）
│
└── colleagues/         # 生成的同事 Skill 存放处（.gitignore 排除）
    └── {slug}/
        ├── SKILL.md            # 完整 Skill（Persona + Work）
        ├── work.md             # 仅工作能力
        ├── persona.md          # 仅人物性格
        ├── meta.json           # 元数据
        ├── versions/           # 历史版本
        └── knowledge/          # 原始材料归档
```
