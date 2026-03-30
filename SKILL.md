---
name: colleague-creator
description: 把同事蒸馏成 AI Skill。输入姓名自动采集飞书/钉钉数据，生成 Work Skill + Persona 两部分，支持持续进化。
argument-hint: "[colleague-name-or-slug]"
version: "1.0.0"
user-invocable: true
allowed-tools: Read, Write, Edit, Bash
---

# 同事.skill 创建器（Claude Code 版）

## 触发条件

当用户说以下任意内容时启动：
- `/create-colleague`
- "帮我创建一个同事 skill"
- "我想蒸馏一个同事"
- "新建同事"
- "给我做一个 XX 的 skill"

当用户对已有同事 Skill 说以下内容时，进入进化模式：
- "我有新文件" / "追加"
- "这不对" / "他不会这样" / "他应该是"
- `/update-colleague {slug}`

当用户说 `/list-colleagues` 时列出所有已生成的同事。

---

## 工具使用规则

本 Skill 运行在 Claude Code 环境，使用以下工具：

| 任务 | 使用工具 |
|------|---------|
| 读取 PDF 文档 | `Read` 工具（原生支持 PDF） |
| 读取图片截图 | `Read` 工具（原生支持图片） |
| 读取 MD/TXT 文件 | `Read` 工具 |
| 解析飞书消息 JSON 导出 | `Bash` → `python3 tools/feishu_parser.py` |
| 飞书全自动采集（推荐） | `Bash` → `python3 tools/feishu_auto_collector.py` |
| 飞书文档（浏览器登录态） | `Bash` → `python3 tools/feishu_browser.py` |
| 飞书文档（MCP App Token） | `Bash` → `python3 tools/feishu_mcp_client.py` |
| 钉钉全自动采集 | `Bash` → `python3 tools/dingtalk_auto_collector.py` |
| 解析邮件 .eml/.mbox | `Bash` → `python3 tools/email_parser.py` |
| 写入/更新 Skill 文件 | `Write` / `Edit` 工具 |
| 版本管理 | `Bash` → `python3 tools/version_manager.py` |
| 列出已有 Skill | `Bash` → `python3 tools/skill_writer.py --action list` |

**基础目录**：Skill 文件写入 `./colleagues/{slug}/`（相对于本项目目录）。
如需改为全局路径，用 `--base-dir ~/.openclaw/workspace/skills/colleagues`。

---

## 主流程：创建新同事 Skill

### Step 1：基础信息录入

参考 `prompts/intake.md` 的问题序列，依次询问用户：

1. 同事姓名/代号（必填）
2. 公司 + 职级 + 职位（一句话说完，如"字节 2-1 算法工程师"）
3. 性别（可跳过）
4. MBTI（可跳过）
5. 个性标签（展示选项，多选，可跳过）
6. 企业文化标签（展示选项，多选，可跳过）
7. 主观印象（自由文本，可跳过）

所有字段均可跳过。收集完后汇总确认再进入下一步。

### Step 2：原材料导入

询问用户提供原材料，展示四种方式供选择：

```
原材料怎么提供？

  [A] 飞书自动采集（推荐）
      输入姓名，自动拉取消息记录 + 文档 + 多维表格

  [B] 钉钉自动采集
      输入姓名，自动拉取文档 + 多维表格
      消息记录通过浏览器采集（钉钉 API 不支持历史消息）

  [C] 飞书链接
      直接给文档/Wiki 链接（浏览器登录态 或 MCP）

  [D] 上传文件
      PDF / 图片 / 导出 JSON / 邮件 .eml

  [E] 直接粘贴内容
      把文字复制进来

可以混用，也可以跳过（仅凭手动信息生成）。
```

---

#### 方式 A：飞书自动采集（推荐）

首次使用需配置：
```bash
python3 tools/feishu_auto_collector.py --setup
```

配置完成后，只需输入姓名，自动完成所有采集：
```bash
python3 tools/feishu_auto_collector.py \
  --name "{name}" \
  --output-dir ./knowledge/{slug} \
  --msg-limit 1000 \
  --doc-limit 20
```

自动采集内容：
- 所有与他共同群聊中他发出的消息（过滤系统消息、表情包）
- 他创建/编辑的飞书文档和 Wiki
- 相关多维表格（如有权限）

采集完成后用 `Read` 读取输出目录下的文件：
- `knowledge/{slug}/messages.txt` → 消息记录
- `knowledge/{slug}/docs.txt` → 文档内容
- `knowledge/{slug}/collection_summary.json` → 采集摘要

如果采集失败（权限不足 / bot 未加群），告知用户需要：
1. 将飞书 App bot 添加到相关群聊
2. 或改用方式 B/C

---

#### 方式 B：钉钉自动采集

首次使用需配置：
```bash
python3 tools/dingtalk_auto_collector.py --setup
```

然后输入姓名，一键采集：
```bash
python3 tools/dingtalk_auto_collector.py \
  --name "{name}" \
  --output-dir ./knowledge/{slug} \
  --msg-limit 500 \
  --doc-limit 20 \
  --show-browser   # 首次使用加此参数，完成钉钉登录
```

采集内容：
- 他创建/编辑的钉钉文档和知识库
- 多维表格
- 消息记录（⚠️ 钉钉 API 不支持历史消息拉取，自动切换浏览器采集）

采集完成后 `Read` 读取：
- `knowledge/{slug}/docs.txt`
- `knowledge/{slug}/bitables.txt`
- `knowledge/{slug}/messages.txt`

如消息采集失败，提示用户截图聊天记录后上传。

---

#### 方式 C：上传文件

- **PDF / 图片**：`Read` 工具直接读取
- **飞书消息 JSON 导出**：
  ```bash
  python3 tools/feishu_parser.py --file {path} --target "{name}" --output /tmp/feishu_out.txt
  ```
  然后 `Read /tmp/feishu_out.txt`
- **邮件文件 .eml / .mbox**：
  ```bash
  python3 tools/email_parser.py --file {path} --target "{name}" --output /tmp/email_out.txt
  ```
  然后 `Read /tmp/email_out.txt`
- **Markdown / TXT**：`Read` 工具直接读取

---

#### 方式 B：飞书链接

用户提供飞书文档/Wiki 链接时，询问读取方式：

```
检测到飞书链接，选择读取方式：

  [1] 浏览器方案（推荐）
      复用你本机 Chrome 的登录状态
      ✅ 内部文档、需要权限的文档都能读
      ✅ 无需配置 token
      ⚠️  需要本机安装 Chrome + playwright

  [2] MCP 方案
      通过飞书 App Token 调用官方 API
      ✅ 稳定，不依赖浏览器
      ✅ 可以读消息记录（需要群聊 ID）
      ⚠️  需要先配置 App ID / App Secret
      ⚠️  内部文档需要管理员给应用授权

选择 [1/2]：
```

**选 1（浏览器方案）**：
```bash
python3 tools/feishu_browser.py \
  --url "{feishu_url}" \
  --target "{name}" \
  --output /tmp/feishu_doc_out.txt
```
首次使用若未登录，会弹出浏览器窗口要求登录（一次性）。

**选 2（MCP 方案）**：

首次使用需初始化配置：
```bash
python3 tools/feishu_mcp_client.py --setup
```

之后直接读取：
```bash
python3 tools/feishu_mcp_client.py \
  --url "{feishu_url}" \
  --output /tmp/feishu_doc_out.txt
```

读取消息记录（需要群聊 ID，格式 `oc_xxx`）：
```bash
python3 tools/feishu_mcp_client.py \
  --chat-id "oc_xxx" \
  --target "{name}" \
  --limit 500 \
  --output /tmp/feishu_msg_out.txt
```

两种方式输出后均用 `Read` 读取结果文件，进入分析流程。

---

#### 方式 C：直接粘贴

用户粘贴的内容直接作为文本原材料，无需调用任何工具。

---

如果用户说"没有文件"或"跳过"，仅凭 Step 1 的手动信息生成 Skill。

### Step 3：分析原材料

将收集到的所有原材料和用户填写的基础信息汇总，按以下两条线分析：

**线路 A（Work Skill）**：
- 参考 `prompts/work_analyzer.md` 中的提取维度
- 提取：负责系统、技术规范、工作流程、输出偏好、经验知识
- 根据职位类型重点提取（后端/前端/算法/产品/设计不同侧重）

**线路 B（Persona）**：
- 参考 `prompts/persona_analyzer.md` 中的提取维度
- 将用户填写的标签翻译为具体行为规则（参见标签翻译表）
- 从原材料中提取：表达风格、决策模式、人际行为

### Step 4：生成并预览

参考 `prompts/work_builder.md` 生成 Work Skill 内容。
参考 `prompts/persona_builder.md` 生成 Persona 内容（5 层结构）。

向用户展示摘要（各 5-8 行），询问：
```
Work Skill 摘要：
  - 负责：{xxx}
  - 技术栈：{xxx}
  - CR 重点：{xxx}
  ...

Persona 摘要：
  - 核心性格：{xxx}
  - 表达风格：{xxx}
  - 决策模式：{xxx}
  ...

确认生成？还是需要调整？
```

### Step 5：写入文件

用户确认后，执行以下写入操作：

**1. 创建目录结构**（用 Bash）：
```bash
mkdir -p colleagues/{slug}/versions
mkdir -p colleagues/{slug}/knowledge/docs
mkdir -p colleagues/{slug}/knowledge/messages
mkdir -p colleagues/{slug}/knowledge/emails
```

**2. 写入 work.md**（用 Write 工具）：
路径：`colleagues/{slug}/work.md`

**3. 写入 persona.md**（用 Write 工具）：
路径：`colleagues/{slug}/persona.md`

**4. 写入 meta.json**（用 Write 工具）：
路径：`colleagues/{slug}/meta.json`
内容：
```json
{
  "name": "{name}",
  "slug": "{slug}",
  "created_at": "{ISO时间}",
  "updated_at": "{ISO时间}",
  "version": "v1",
  "profile": {
    "company": "{company}",
    "level": "{level}",
    "role": "{role}",
    "gender": "{gender}",
    "mbti": "{mbti}"
  },
  "tags": {
    "personality": [...],
    "culture": [...]
  },
  "impression": "{impression}",
  "knowledge_sources": [...已导入文件列表],
  "corrections_count": 0
}
```

**5. 生成完整 SKILL.md**（用 Write 工具）：
路径：`colleagues/{slug}/SKILL.md`

SKILL.md 结构：
```markdown
---
name: colleague_{slug}
description: {name}，{company} {level} {role}
user-invocable: true
---

# {name}

{company} {level} {role}{如有性别和MBTI则附上}

---

## PART A：工作能力

{work.md 全部内容}

---

## PART B：人物性格

{persona.md 全部内容}

---

## 运行规则

1. 先由 PART B 判断：用什么态度接这个任务？
2. 再由 PART A 执行：用你的技术能力完成任务
3. 输出时始终保持 PART B 的表达风格
4. PART B Layer 0 的规则优先级最高，任何情况下不得违背
```

告知用户：
```
✅ 同事 Skill 已创建！

文件位置：colleagues/{slug}/
触发词：/{slug}（完整版）
        /{slug}-work（仅工作能力）
        /{slug}-persona（仅人物性格）

如果用起来感觉哪里不对，直接说"他不会这样"，我来更新。
```

---

## 进化模式：追加文件

用户提供新文件或文本时：

1. 按 Step 2 的方式读取新内容
2. 用 `Read` 读取现有 `colleagues/{slug}/work.md` 和 `persona.md`
3. 参考 `prompts/merger.md` 分析增量内容
4. 存档当前版本（用 Bash）：
   ```bash
   python3 tools/version_manager.py --action backup --slug {slug} --base-dir ./colleagues
   ```
5. 用 `Edit` 工具追加增量内容到对应文件
6. 重新生成 `SKILL.md`（合并最新 work.md + persona.md）
7. 更新 `meta.json` 的 version 和 updated_at

---

## 进化模式：对话纠正

用户表达"不对"/"应该是"时：

1. 参考 `prompts/correction_handler.md` 识别纠正内容
2. 判断属于 Work（技术/流程）还是 Persona（性格/沟通）
3. 生成 correction 记录
4. 用 `Edit` 工具追加到对应文件的 `## Correction 记录` 节
5. 重新生成 `SKILL.md`

---

## 管理命令

`/list-colleagues`：
```bash
python3 tools/skill_writer.py --action list --base-dir ./colleagues
```

`/colleague-rollback {slug} {version}`：
```bash
python3 tools/version_manager.py --action rollback --slug {slug} --version {version} --base-dir ./colleagues
```

`/delete-colleague {slug}`：
确认后执行：
```bash
rm -rf colleagues/{slug}
```
