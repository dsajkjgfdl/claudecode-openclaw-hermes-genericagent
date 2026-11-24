---
title: "附录 B：实战：编写一个自定义技能"
appendix: B
status: DRAFT
---

# 附录 B：实战——编写一个自定义技能

> **定位：** 本附录是一份动手工坊指南。读完后，你将从零完成一个 Hermes Agent 自定义技能的创建、测试、安全检查与发布全流程，并深入理解技能系统背后的源码机制。

---

## B.1 什么是"技能"？

在 Hermes Agent 中，**技能（Skill）** 是一组以 Markdown 文件为载体的"程序性记忆"——它告诉 Agent 在特定场景下**应该怎么做**。与传统插件不同，技能不需要编写任何可执行代码，只需一个 `SKILL.md` 文件即可定义完整的操作流程。

技能的核心设计哲学可以用三个词概括：

| 原则 | 含义 |
|------|------|
| **约定优于配置** | 目录名即类别、文件存在即技能——无需注册表 |
| **渐进式披露** | 系统提示词只加载名称+描述，完整内容按需读取 |
| **纵深防御** | 每次创建/修改都经过安全扫描，失败则自动回滚 |

技能存放在 `~/.hermes/skills/` 目录下，目录结构本身就是注册机制：

```
~/.hermes/skills/
├── research/                  # 类别目录
│   ├── arxiv/                 # 技能目录
│   │   └── SKILL.md           # ← 核心：技能指令文件
│   └── blogwatcher/
│       └── SKILL.md
├── productivity/
│   └── notion/
│       ├── SKILL.md
│       ├── references/        # 参考文档（仅 *.md）
│       ├── templates/         # 模板文件
│       ├── scripts/           # 辅助脚本
│       └── assets/            # 补充资源
└── .hub/                      # Hub 状态（排除扫描）
    ├── lock.json
    └── ...
```

**源码验证：** 允许的子目录定义在 `skill_manager_tool.py` 中：

```python
ALLOWED_SUBDIRS = {"references", "templates", "scripts", "assets"}
```

而 `.git`、`.github`、`.hub` 等目录在扫描时被排除（`skill_utils.py:EXCLUDED_SKILL_DIRS`）。

---

## B.2 技能的加载机制：三层渐进式披露

在动手编写之前，先理解 Hermes 如何发现和使用技能——这会帮助你写出更高效的 `SKILL.md`。

### 第一层：系统提示词注入（极低 Token 消耗）

每次会话启动时，`agent/prompt_builder.py` 中的 `build_skills_system_prompt()` 会扫描所有已安装技能，在系统提示词中生成一个紧凑的索引：

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially
relevant to your task, you MUST load it with skill_view(name)...

<available_skills>
  research:
    - arxiv: Search and retrieve academic papers...
    - blogwatcher: Monitor blog feeds...
  productivity:
    - notion: Manage Notion workspace...
</available_skills>
```

**性能优化：** 实际实现中使用快照缓存（磁盘快照 + 内存 LRU），避免每次都做文件系统扫描。安装/卸载技能后调用 `clear_skills_system_prompt_cache()` 使缓存失效。

### 第二层：`skill_view(name)` — 加载完整内容（中等消耗）

当 Agent 判断某个技能与当前任务相关时，调用 `skills_list()` 获取列表，再调用 `skill_view(name)` 读取完整的 `SKILL.md` 内容和链接文件列表。

### 第三层：`skill_view(name, file_path)` — 按需加载辅助文件

仅当 Agent 需要查看具体的参考文档、模板或脚本时才加载，最大化节省 Token。

> **设计启示：** 技能的 `description` 字段极其重要——它是第一层中 Agent 判断是否加载该技能的唯一依据。写得太模糊会导致该加载时不加载，太宽泛则浪费 Token。

---

## B.3 动手编写：一个完整的自定义技能

我们以一个实际场景为例：编写一个使用 **JSONPlaceholder** 免费 API 管理待办事项（Todos）的技能。

### Step 1：确定技能范围

好的技能应该是 **窄而深** 的：

- ✅ "使用 JSONPlaceholder API 管理待办事项"——范围清晰
- ✅ "使用 himalaya CLI 管理邮件"——聚焦单一工具
- ❌ "通用 REST API 调用"——太宽泛，Agent 无法判断何时使用

### Step 2：创建目录结构

```bash
mkdir -p ~/.hermes/skills/api-tools/jsonplaceholder
```

这里 `api-tools` 是类别名，`jsonplaceholder` 是技能名。

### Step 3：编写 SKILL.md

这是技能的核心文件，由 **YAML frontmatter** + **Markdown body** 两部分组成。

```markdown
---
name: jsonplaceholder
description: Query and manage todos using the JSONPlaceholder free REST API for testing and prototyping.
version: 1.0.0
author: Your Name
license: MIT
metadata:
  hermes:
    tags: [API, REST, Testing, Todos, Prototyping]
    requires_toolsets: [terminal]
---

# JSONPlaceholder Todos

Query and manage todo items using the JSONPlaceholder free REST API
(https://jsonplaceholder.typicode.com).

## When to Use

- User asks about JSONPlaceholder or mock/fake REST API
- Need to prototype or test REST API workflows
- Want to demonstrate CRUD operations with a free API

## Quick Reference

| Action         | Command |
|----------------|---------|
| List all todos | `curl -s https://jsonplaceholder.typicode.com/todos` |
| Get one todo   | `curl -s https://jsonplaceholder.typicode.com/todos/1` |
| User's todos   | `curl -s https://jsonplaceholder.typicode.com/todos?userId=1` |
| Create todo    | `curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"Buy milk","completed":false,"userId":1}' https://jsonplaceholder.typicode.com/todos` |
| Update todo    | `curl -s -X PUT -H 'Content-Type: application/json' -d '{"id":1,"title":"Updated","completed":true,"userId":1}' https://jsonplaceholder.typicode.com/todos/1` |
| Delete todo    | `curl -s -X DELETE https://jsonplaceholder.typicode.com/todos/1` |

## Procedure

1. **Identify the operation** the user needs (list/get/create/update/delete)
2. **Construct the curl command** using Quick Reference above
3. **Execute the command** in terminal
4. **Parse the JSON response** and present results in a readable format
5. For listing, pipe through `python3 -m json.tool` for pretty-printing
6. For filtered queries, use `jq` if available, otherwise use Python:
   ```bash
   curl -s https://jsonplaceholder.typicode.com/todos | \
     python3 -c "import json,sys; todos=json.load(sys.stdin); \
     [print(f'[{\"✓\" if t[\"completed\"] else \" \"}] #{t[\"id\"]}: {t[\"title\"]}') for t in todos[:10]]"
   ```

## Pitfalls

- **Fake mutations:** POST/PUT/DELETE return success but don't actually persist.
  Always inform the user that this is a mock API.
- **No authentication:** This API is public, no keys needed.
- **Rate limiting:** Be courteous — avoid rapid-fire requests in loops.
- **200 todos total:** The dataset has exactly 200 items for 10 users.

## Verification

- Successful requests return HTTP 200 (GET/PUT/DELETE) or 201 (POST)
- Response body is valid JSON with expected fields: `id`, `title`, `completed`, `userId`
- Use `curl -s -o /dev/null -w '%{http_code}'` to check status code only
```

### Step 4：理解 frontmatter 字段的源码约束

编写 frontmatter 时需要遵守源码中定义的约束条件：

**必填字段：**

| 字段 | 约束 | 源码位置 |
|------|------|----------|
| `name` | 匹配 `^[a-z0-9][a-z0-9._-]*$`，最长 64 字符 | `skill_manager_tool.py:_validate_frontmatter()` |
| `description` | 最长 1024 字符 | 同上 |

**可选字段：**

| 字段 | 说明 |
|------|------|
| `version` | 语义版本号（如 `1.0.0`） |
| `platforms` | 限制平台：`[macos, linux, windows]`。内部映射为 `{macos→darwin, linux→linux, windows→win32}` |
| `required_environment_variables` | 技能加载时安全收集，存入 `~/.hermes/.env`，**绝不暴露给 Agent** |
| `metadata.hermes.requires_toolsets` | 仅当指定工具集可用时才显示此技能 |
| `metadata.hermes.fallback_for_toolsets` | 当指定工具集可用时**隐藏**此技能（回退方案） |

**条件激活**的源码逻辑在 `prompt_builder.py` 的 `_skill_should_show()` 中：

```python
# fallback_for: 当主工具集可用时，隐藏回退技能
for ts in conditions.get("fallback_for_toolsets", []):
    if ts in available_toolsets:
        return False  # 主工具可用，不需要回退

# requires: 当需要的工具集不可用时，隐藏此技能
for ts in conditions.get("requires_toolsets", []):
    if ts not in available_toolsets:
        return False  # 缺少前提工具
```

### Step 5：添加辅助文件（可选）

如果你的技能涉及复杂的 API 字段或需要辅助脚本，可以添加子目录：

```bash
# 参考文档
mkdir -p ~/.hermes/skills/api-tools/jsonplaceholder/references
cat > ~/.hermes/skills/api-tools/jsonplaceholder/references/data-model.md << 'EOF'
# JSONPlaceholder Data Model

## Todo Object

| Field       | Type    | Description              |
|-------------|---------|--------------------------|
| `id`        | integer | Unique identifier (1-200)|
| `userId`    | integer | Owner user ID (1-10)     |
| `title`     | string  | Todo description         |
| `completed` | boolean | Completion status        |

## Available Endpoints

- `/todos` — 200 items
- `/posts` — 100 items
- `/comments` — 500 items
- `/users` — 10 items
EOF
```

```bash
# 辅助脚本
mkdir -p ~/.hermes/skills/api-tools/jsonplaceholder/scripts
cat > ~/.hermes/skills/api-tools/jsonplaceholder/scripts/format_todos.py << 'PYEOF'
#!/usr/bin/env python3
"""Format JSONPlaceholder todos as a readable table."""
import json, sys

def main():
    todos = json.load(sys.stdin)
    if isinstance(todos, dict):
        todos = [todos]
    print(f"{'ID':>4}  {'User':>4}  {'Done':>4}  Title")
    print("-" * 60)
    for t in todos:
        mark = "✓" if t["completed"] else " "
        print(f"{t['id']:>4}  {t['userId']:>4}  [{mark}]   {t['title']}")

if __name__ == "__main__":
    main()
PYEOF
```

这些辅助文件会被 `skills_tool.py:skill_view()` 自动发现。发现逻辑按四类扫描：

```python
# references/ — 仅 *.md
reference_files = [f for f in references_dir.glob("*.md")]

# templates/ — 多种格式
for ext in ["*.md", "*.py", "*.yaml", "*.yml", "*.json", "*.tex", "*.sh"]:
    template_files.extend(templates_dir.rglob(ext))

# scripts/ — 可执行脚本
for ext in ["*.py", "*.sh", "*.bash", "*.js", "*.ts", "*.rb"]:
    script_files.extend(scripts_dir.glob(ext))

# assets/ — 所有文件
asset_files = [f for f in assets_dir.rglob("*") if f.is_file()]
```

当 Agent 调用 `skill_view("jsonplaceholder")` 时，返回的 JSON 会包含：

```json
{
  "linked_files": {
    "references": ["references/data-model.md"],
    "scripts": ["scripts/format_todos.py"]
  },
  "usage_hint": "To view linked files, call skill_view(name, file_path)"
}
```

---

## B.4 另一种方式：让 Agent 帮你创建

除了手动创建文件，你也可以直接对话让 Agent 创建技能：

```
> 请帮我创建一个使用 JSONPlaceholder API 管理待办事项的技能
```

Agent 会调用 `skill_manage` 工具：

```python
skill_manage(
    action="create",
    name="jsonplaceholder",
    category="api-tools",
    content="---\nname: jsonplaceholder\ndescription: ..."
)
```

**创建流程的源码细节（`skill_manager_tool.py`）：**

```
create 动作:
  1. name 格式验证（正则 + 长度）
  2. frontmatter 解析与验证（必填字段检查）
  3. 名称冲突检查（扫描所有技能目录）
  4. 原子写入（临时文件 + os.replace）
  5. 安全扫描（skills_guard.scan_skill）
  6. 扫描失败 → shutil.rmtree 回滚整个目录
```

原子写入的实现值得关注：

```python
def _atomic_write_text(file_path: Path, content: str):
    """使用临时文件 + os.replace() 确保原子性"""
    fd, temp_path = tempfile.mkstemp(
        dir=str(file_path.parent),
        prefix=f".{file_path.name}.tmp.",
    )
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(content)
        os.replace(temp_path, file_path)  # 原子替换
    except Exception:
        os.unlink(temp_path)  # 清理临时文件
        raise
```

---

## B.5 安全扫描：你的技能必须过的关

每次技能创建或修改都会经过 `tools/skills_guard.py` 的三层安全检查。理解这些规则能帮助你避免写出被误判的内容。

### 扫描架构

```
scan_skill(skill_path, source)
  ├─ _check_structure(skill_dir)    # 结构检查
  ├─ scan_file(file) × N            # 内容检查（每个文件）
  └─ _determine_verdict(findings)   # 综合判定
```

### 结构检查

| 检查项 | 阈值 | 严重度 |
|--------|------|--------|
| 文件数量 | > 50 | medium |
| 总大小 | > 1024KB | medium |
| 单文件大小 | > 256KB | medium |
| 可疑二进制 | .exe / .dll / .so / .dylib | high |
| 符号链接逃逸 | 指向技能目录外 | critical |

### 内容模式匹配

`THREAT_PATTERNS` 包含 40+ 条正则规则，分为 8 大类。以下是技能编写中**最容易触发**的几类：

| 类别 | 会触发的内容示例 | 严重度 | 如何避免 |
|------|-----------------|--------|----------|
| `injection` | 系统提示覆写、角色切换文本 | critical | 不要在技能中放置 "ignore previous instructions" 等文本 |
| `code_execution` | `eval()`, `exec()`, `os.system()` | high | 在 scripts/ 中使用，正文用 `subprocess.run()` |
| `supply_chain` | `pip install xxx` | medium | 优先使用 stdlib，必须安装时说明原因 |
| `credential_exposure` | 硬编码的 API key | critical | 使用 `required_environment_variables` 声明 |
| `network` | `curl ... \| bash` | high | 不要管道到 shell |

### 不可见 Unicode 检测

Guard 还会检测 16 种不可见 Unicode 字符，包括零宽空格（`\u200b`）、右到左覆写（`\u202e`，经典注入手法）等。这类字符可能被恶意技能用来隐藏有害指令。

### 判定与信任等级

扫描结果分三级：`clean` → `suspicious` → `dangerous`。

结合技能来源的信任等级，决定是否允许安装：

```
               clean    suspicious    dangerous
builtin:       allow    allow         allow
trusted:       allow    allow         ask
community:     allow    ask           block
agent-created: allow    block         block   ← 最严格
```

> **注意：** Agent 自己创建的技能（source = `agent-created`）采用最严格策略。即使只是 `suspicious`，也会被 block 并回滚。

---

## B.6 测试你的技能

### 方法一：直接对话测试

```bash
hermes --toolsets skills -q "Use the jsonplaceholder skill to list all todos for user 3"
```

Agent 会：
1. 在系统提示词中看到你的技能（Tier 1）
2. 判断相关后调用 `skill_view("jsonplaceholder")`（Tier 2）
3. 按照 `SKILL.md` 中的 Procedure 执行操作

### 方法二：使用斜杠命令

技能注册后会自动生成斜杠命令。源码逻辑在 `agent/skill_commands.py:scan_skill_commands()` 中：

```python
cmd_name = name.lower().replace(' ', '-').replace('_', '-')
# 结果: /jsonplaceholder
```

在对话中输入 `/jsonplaceholder` 会直接加载技能，跳过 Agent 的判断步骤：

```
[SYSTEM: The user has invoked the "jsonplaceholder" skill, indicating they want
you to follow its instructions. The full skill content is loaded below.]
```

### 方法三：验证 frontmatter 解析

如果你想验证 YAML frontmatter 是否能被正确解析，可以用 Python 快速检查：

```python
import yaml

with open("~/.hermes/skills/api-tools/jsonplaceholder/SKILL.md") as f:
    content = f.read()

# 提取 frontmatter（源码使用相同逻辑）
if content.startswith("---"):
    end = content.index("---", 3)
    frontmatter = yaml.safe_load(content[3:end])
    print(f"Name: {frontmatter.get('name')}")
    print(f"Description: {frontmatter.get('description')}")
    print(f"Tags: {frontmatter.get('metadata', {}).get('hermes', {}).get('tags', [])}")
```

> **源码细节：** `agent/skill_utils.py:parse_frontmatter()` 优先使用 `yaml.CSafeLoader`（C 加速版），失败时回退到纯 Python 的 `yaml.SafeLoader`。

### 调试清单

| 问题 | 排查方法 |
|------|----------|
| 技能未出现在列表中 | 检查 `SKILL.md` 是否存在、frontmatter 是否有 `name` 和 `description` |
| 技能被隐藏 | 检查 `requires_toolsets` 条件是否满足、是否在 `disabled` 列表中 |
| 斜杠命令不工作 | 检查技能名是否包含非法字符（仅 `[a-z0-9.-]`） |
| 安全扫描失败 | 检查是否包含 `eval()`/`exec()` 等高危模式、是否有硬编码密钥 |
| 辅助文件未显示 | 确认放在 `references/`、`templates/`、`scripts/`、`assets/` 四个目录之一 |

---

## B.7 技能的自我改进循环

Hermes 的技能系统有一个独特的设计：**技能是 Agent 的程序性记忆**，会随着使用不断改进。

系统提示词中明确要求 Agent：

```
After difficult/iterative tasks, offer to save as a skill.
If a skill you loaded was missing steps, had wrong commands, or needed
pitfalls you discovered, update it before finishing.
```

这创造了一个正反馈循环：

```
遇到任务 → 加载技能 → 发现缺陷 → Agent 用 skill_manage(action='patch') 更新
                                    ↓
                            下次同类任务 → 技能已改进
```

`patch` 操作使用模糊匹配引擎（`fuzzy_find_and_replace`），支持定向查找替换而无需重写整个文件。修改后同样会经过安全扫描，失败则恢复备份。

---

## B.8 发布与分享

### 发布到 GitHub

```bash
hermes skills publish api-tools/jsonplaceholder --to github --repo myorg/hermes-skills
```

发布流程（`hermes_cli/skills_hub.py:do_publish()`）：

1. 验证 `SKILL.md` 存在且有 `description`
2. 自我安全扫描（`scan_skill(path, source="self")`）
3. 如果 verdict == `"dangerous"` → 拒绝发布
4. GitHub API 操作：Fork → 创建分支 `add-skill-<name>` → 上传文件 → 创建 PR

### 使用自定义源（Tap）

```bash
hermes skills tap add myorg/hermes-skills     # 添加自定义技能源
hermes skills tap list                         # 列出所有源
hermes skills search "todo api" --tap myorg    # 从自定义源搜索
hermes skills install myorg/hermes-skills/api-tools/jsonplaceholder
```

### 配置外部技能目录

对于团队共享场景，可以配置外部目录：

```yaml
# ~/.hermes/config.yaml
skills:
  external_dirs:
    - ~/my-team-skills
    - /shared/company-skills
```

`skill_utils.py:get_external_skills_dirs()` 会展开 `~` 和环境变量、去重、验证目录存在后加入扫描路径。

---

## B.9 需要环境变量的技能

如果你的技能需要 API Key 等凭证，使用 `required_environment_variables` 声明：

```yaml
---
name: weather-api
description: Get weather data from OpenWeatherMap API
required_environment_variables:
  - OPENWEATHER_API_KEY
---
```

加载此技能时（`skill_view`），Hermes 会：

1. 检查 `~/.hermes/.env` 中是否已持久化该变量
2. CLI 会话 → 安全提示用户输入（值不会回显）
3. Gateway 会话 → 提示运行 `hermes setup`
4. 通过 `register_env_passthrough()` 传递到沙箱环境
5. 返回给 Agent 的信息中**只有元数据**，绝不包含密钥值：

```python
result = {
    "required_environment_variables": ["OPENWEATHER_API_KEY"],
    "missing_required_environment_variables": [],
    "setup_needed": False,
    "readiness_status": "available",
}
```

---

## B.10 完整参考：技能相关的工具 API

Hermes 注册了三个与技能相关的工具，均属于 `skills` 工具集：

### `skills_list(category?)` — 列出技能

```python
registry.register(
    name="skills_list",
    toolset="skills",
    schema=SKILLS_LIST_SCHEMA,  # 参数: category (可选)
    handler=skills_list,
    emoji="📚",
)
```

返回所有已启用技能的 name + description + category。禁用检查逻辑 `_is_skill_disabled()` 依次检查平台特定禁用列表和全局禁用列表。

### `skill_view(name, file_path?)` — 查看技能内容

```python
registry.register(
    name="skill_view",
    toolset="skills",
    schema=SKILL_VIEW_SCHEMA,  # 参数: name (必填), file_path (可选)
    handler=skill_view,
    emoji="📚",
)
```

不传 `file_path` 时返回完整 `SKILL.md` + 链接文件列表；传 `file_path` 时返回指定辅助文件内容。

### `skill_manage(action, ...)` — CRUD 操作

```python
registry.register(
    name="skill_manage",
    toolset="skills",
    schema=SKILL_MANAGE_SCHEMA,
    handler=skill_manage,
    emoji="📚",
)
```

支持 6 种动作：

| 动作 | 说明 | 失败回滚策略 |
|------|------|-------------|
| `create` | 创建新技能 | `shutil.rmtree` 删除整个目录 |
| `edit` | 完全重写 SKILL.md | 恢复备份内容 |
| `patch` | 模糊查找替换 | 恢复备份内容 |
| `delete` | 删除整个技能目录 | — |
| `write_file` | 添加/覆盖辅助文件（限 1MiB） | 删除新文件或恢复旧文件 |
| `remove_file` | 删除辅助文件 | — |

---

## B.11 大小限制速查

```python
MAX_NAME_LENGTH = 64              # 技能名最长字符数
MAX_DESCRIPTION_LENGTH = 1024     # 描述最长字符数
MAX_SKILL_CONTENT_CHARS = 100_000 # SKILL.md 最大字符数（~36k tokens）
MAX_SKILL_FILE_BYTES = 1_048_576  # 辅助文件最大 1 MiB
```

---

## B.12 小结：编写优秀技能的检查清单

完成技能编写后，用这份清单做最终检查：

- [ ] **名称合规**：全小写，仅 `[a-z0-9._-]`，不超过 64 字符
- [ ] **描述精准**：一句话说清用途，不超过 1024 字符，让 Agent 能快速判断相关性
- [ ] **When to Use 明确**：列出具体触发场景
- [ ] **Quick Reference 实用**：最常用的 3-5 个操作一目了然
- [ ] **Procedure 可执行**：分步指令，Agent 按步骤即可完成任务
- [ ] **Pitfalls 诚实**：列出已知的失败模式和边界条件
- [ ] **Verification 可验证**：给出成功的判断标准
- [ ] **无安全触发**：不含 `eval()`、硬编码密钥、`curl|bash` 等模式
- [ ] **大小合理**：SKILL.md 不超过 100,000 字符，辅助文件不超过 1MiB
- [ ] **平台声明**：如果仅支持特定平台，在 frontmatter 中声明 `platforms`
- [ ] **渐进式设计**：常用操作在前，边缘情况在后，复杂逻辑放入 `scripts/`
