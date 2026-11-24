---
Task: TB-R
Status: DRAFT_DONE
Date: 2026-04-15
Keywords: 技能开发与发布
Sources:
  - tools/skill_manager_tool.py (761 lines)
  - tools/skills_tool.py (1419 lines)
  - tools/skills_guard.py (928 lines)
  - tools/skills_hub.py (3053 lines)
  - agent/skill_commands.py (368 lines)
  - agent/skill_utils.py (465 lines)
  - agent/prompt_builder.py (build_skills_system_prompt)
  - hermes_cli/skills_hub.py (1238 lines)
  - hermes_cli/skills_config.py (skills TUI)
  - CONTRIBUTING.md (SKILL.md format spec)
---

# 附录 B 研究笔记 — 实战：编写一个自定义技能
# Appendix B Research Notes — Hands-On: Writing a Custom Skill

## 1. SKILL.md 格式详解 (SKILL.md Format Specification)

### 1.1 YAML Frontmatter 字段 (YAML Frontmatter Fields)

每个技能的核心是 `SKILL.md` 文件，它由 YAML frontmatter + Markdown body 组成。

**必填字段 (Required Fields):**

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 技能名称，匹配正则 `^[a-z0-9][a-z0-9._-]*$`，最长 64 字符 |
| `description` | string | 简短描述（显示在搜索结果中），最长 1024 字符 |

**可选字段 (Optional Fields):**

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | string | 语义版本号（如 `1.0.0`） |
| `author` | string | 作者名 |
| `license` | string | 许可证（如 MIT） |
| `platforms` | list | 限制平台：`[macos, linux, windows]`，省略则全平台 |
| `required_environment_variables` | list | 安全的环境变量加载时收集 |
| `required_credential_files` | list | 需要挂载到沙箱的凭证文件 |
| `prerequisites` | dict | 旧式运行时需求（`env_vars`, `commands`） |
| `compatibility` | dict | agentskills.io 标准兼容性字段 |

**metadata.hermes 子字段 (Hermes Metadata Sub-fields):**

```yaml
metadata:
  hermes:
    tags: [Research, Arxiv, Papers]        # 分类标签 (Category tags)
    related_skills: [ocr-and-documents]    # 关联技能 (Related skills)
    fallback_for_toolsets: [web]           # 回退条件：当 web 工具集不可用时显示
    requires_toolsets: [terminal]          # 前提条件：仅当 terminal 可用时显示
    fallback_for_tools: [web_search]       # 工具级回退条件
    requires_tools: [terminal]             # 工具级前提条件
    config:                                # 技能配置变量声明
      - key: wiki.path
        description: Path to wiki directory
        default: "~/wiki"
        prompt: Wiki directory path
```

**源码位置：**
- 验证逻辑: `skill_manager_tool.py:_validate_frontmatter()` — 检查 YAML 解析、`name`/`description` 必填
- 解析逻辑: `agent/skill_utils.py:parse_frontmatter()` — 使用 `yaml.CSafeLoader`（C 加速），失败回退到 `yaml.SafeLoader`
- 条件提取: `agent/skill_utils.py:extract_skill_conditions()` — 从 `metadata.hermes` 提取 4 个条件字段
- 平台匹配: `agent/skill_utils.py:skill_matches_platform()` — `PLATFORM_MAP = {macos→darwin, linux→linux, windows→win32}`

### 1.2 Markdown Body 结构 (Markdown Body Structure)

CONTRIBUTING.md 定义的推荐章节：

```markdown
# Skill Title

Brief intro.

## When to Use
触发条件 — 何时应该加载此技能？

## Quick Reference
常用命令或 API 调用的速查表。

## Procedure
Agent 要遵循的分步指令。

## Pitfalls
已知的失败模式及处理方法。

## Verification
Agent 如何确认操作成功。
```

**设计原则 (Design Principles):**
- **Progressive disclosure (渐进式披露):** 最常用工作流放在前面，边缘情况放在后面
- **No external dependencies (无外部依赖):** 优先使用 stdlib Python、curl、Hermes 内置工具
- **Include helper scripts:** 复杂逻辑放入 `scripts/` 目录

### 1.3 完整示例 — arxiv 技能

```yaml
---
name: arxiv
description: Search and retrieve academic papers from arXiv using their free REST API.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [Research, Arxiv, Papers, Academic, Science, API]
    related_skills: [ocr-and-documents]
---

# arXiv Research

Search and retrieve academic papers from arXiv via their free REST API.

## Quick Reference

| Action | Command |
|--------|---------|
| Search papers | `curl "https://export.arxiv.org/api/query?..."` |
| Get specific paper | `curl "https://export.arxiv.org/api/query?id_list=2402.03300"` |
```

## 2. 技能目录结构 (Skill Directory Structure)

### 2.1 标准布局 (Standard Layout)

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # 必需：主指令文件
│   ├── references/            # 可选：参考文档（仅 *.md）
│   │   └── api.md
│   ├── templates/             # 可选：模板文件（多种格式）
│   │   └── config.yaml
│   ├── scripts/               # 可选：辅助脚本
│   │   └── helper.py
│   └── assets/                # 可选：补充文件（agentskills.io 标准）
│       └── diagram.svg
├── category-name/             # 可选：按类别组织
│   └── another-skill/
│       └── SKILL.md
└── .hub/                      # Hub 状态目录（排除扫描）
    ├── lock.json
    ├── quarantine/
    ├── audit.log
    ├── taps.json
    └── index-cache/
```

**源码验证:**
- 允许的子目录: `skill_manager_tool.py:ALLOWED_SUBDIRS = {"references", "templates", "scripts", "assets"}`
- 排除的目录: `skill_utils.py:EXCLUDED_SKILL_DIRS = {".git", ".github", ".hub"}`
- 类别描述: 每个类别目录可放置 `DESCRIPTION.md` 文件

### 2.2 链接文件的发现 (Linked File Discovery)

`skills_tool.py:skill_view()` 自动发现四类链接文件：

```python
# references/ — 仅 *.md 文件
reference_files = [f for f in references_dir.glob("*.md")]

# templates/ — 多种格式
for ext in ["*.md", "*.py", "*.yaml", "*.yml", "*.json", "*.tex", "*.sh"]:
    template_files.extend(templates_dir.rglob(ext))

# assets/ — 所有文件（agentskills.io 标准）
asset_files = [f for f in assets_dir.rglob("*") if f.is_file()]

# scripts/ — 可执行脚本
for ext in ["*.py", "*.sh", "*.bash", "*.js", "*.ts", "*.rb"]:
    script_files.extend(scripts_dir.glob(ext))
```

返回的 JSON 结构：
```json
{
  "linked_files": {
    "references": ["references/api.md"],
    "templates": ["templates/config.yaml"],
    "assets": ["assets/diagram.svg"],
    "scripts": ["scripts/helper.py"]
  },
  "usage_hint": "To view linked files, call skill_view(name, file_path)"
}
```

## 3. 技能发现与加载机制 (Skill Discovery & Loading)

### 3.1 三层渐进式披露 (Three-Tier Progressive Disclosure)

Hermes 使用精心设计的三层架构来最小化 token 消耗：

| 层级 | 工具 | 返回内容 | Token 消耗 |
|------|------|----------|-----------|
| Tier 1 | `skills_list()` | name + description + category | 极低 |
| Tier 2 | `skill_view(name)` | 完整 SKILL.md + 链接文件列表 | 中等 |
| Tier 3 | `skill_view(name, file_path)` | 特定链接文件内容 | 按需 |

### 3.2 系统提示词注入 (System Prompt Injection)

`agent/prompt_builder.py:build_skills_system_prompt()` 在会话启动时：

1. **热路径 (Hot path):** 读取快照缓存（避免文件系统扫描）
2. **冷路径 (Cold path):** 递归扫描 `~/.hermes/skills/` + 外部目录 → 写入快照
3. 对每个技能检查：平台兼容、禁用状态、条件激活
4. 生成 `## Skills (mandatory)` 系统提示词块

生成的提示词格式：
```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially relevant
to your task, you MUST load it with skill_view(name) and follow its instructions...

<available_skills>
  research:
    - arxiv: Search and retrieve academic papers...
    - blogwatcher: Monitor blog feeds...
  productivity:
    - notion: Manage Notion workspace...
</available_skills>
```

**条件激活逻辑 (`_skill_should_show()`):**
```python
# fallback_for: 当主工具集可用时隐藏
for ts in conditions.get("fallback_for_toolsets", []):
    if ts in available_toolsets:
        return False  # 主工具可用，不需要回退

# requires: 当需要的工具集不可用时隐藏
for ts in conditions.get("requires_toolsets", []):
    if ts not in available_toolsets:
        return False  # 缺少前提工具
```

### 3.3 斜杠命令注册 (Slash Command Registration)

`agent/skill_commands.py:scan_skill_commands()` 将每个技能注册为 `/skill-name` 命令：

```python
cmd_name = name.lower().replace(' ', '-').replace('_', '-')
cmd_name = _SKILL_INVALID_CHARS.sub('', cmd_name)  # 去除非法字符
cmd_name = _SKILL_MULTI_HYPHEN.sub('-', cmd_name).strip('-')
_skill_commands[f"/{cmd_name}"] = {
    "name": name,
    "description": description,
    "skill_md_path": str(skill_md),
    "skill_dir": str(skill_md.parent),
}
```

调用时生成激活消息：
```
[SYSTEM: The user has invoked the "arxiv" skill, indicating they want
you to follow its instructions. The full skill content is loaded below.]
```

### 3.4 外部技能目录 (External Skills Directories)

通过 `~/.hermes/config.yaml` 配置：
```yaml
skills:
  external_dirs:
    - ~/my-team-skills
    - /shared/company-skills
```

解析逻辑 (`skill_utils.py:get_external_skills_dirs()`):
- 展开 `~` 和环境变量
- 排除与本地 skills 目录相同的路径
- 去重
- 仅保留实际存在的目录

## 4. skill_manage 工具 API (Skill Management Tool API)

### 4.1 操作清单 (Action Inventory)

| 操作 | 说明 | 验证步骤 |
|------|------|----------|
| `create` | 创建新技能（SKILL.md + 目录） | name 验证 → frontmatter 验证 → 名称冲突检查 → 原子写入 → 安全扫描 |
| `edit` | 完全重写 SKILL.md | frontmatter 验证 → 备份原内容 → 原子写入 → 安全扫描（失败回滚） |
| `patch` | 定向查找替换（支持模糊匹配） | 使用 `fuzzy_find_and_replace` 引擎 → 大小检查 → frontmatter 完整性 → 安全扫描 |
| `delete` | 删除整个技能目录 | 查找技能 → `shutil.rmtree` → 清理空父目录 |
| `write_file` | 添加/覆盖辅助文件 | 路径验证 → 大小限制（1MiB）→ 原子写入 → 安全扫描 |
| `remove_file` | 删除辅助文件 | 路径验证 → 删除文件 → 清理空子目录 |

### 4.2 原子写入与回滚 (Atomic Writes & Rollback)

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

**安全扫描回滚模式:**
```
create:  写入 → 扫描失败 → shutil.rmtree(整个目录)
edit:    备份 → 写入 → 扫描失败 → 恢复备份
patch:   备份 → 写入 → 扫描失败 → 恢复备份
write_file: 写入 → 扫描失败 → 删除新文件（恢复旧文件）
```

### 4.3 大小限制 (Size Limits)

```python
MAX_NAME_LENGTH = 64
MAX_DESCRIPTION_LENGTH = 1024
MAX_SKILL_CONTENT_CHARS = 100_000   # ~36k tokens at 2.75 chars/token
MAX_SKILL_FILE_BYTES = 1_048_576    # 1 MiB per supporting file
```

## 5. 安全扫描机制 — Skills Guard (Security Scanning)

### 5.1 扫描架构 (Scanning Architecture)

`tools/skills_guard.py` 实现三层安全检查：

```
scan_skill(skill_path, source)
  ├─ _check_structure(skill_dir)    # 结构检查
  ├─ scan_file(file) × N            # 内容检查（每个文件）
  └─ _determine_verdict(findings)   # 综合判定
```

### 5.2 结构检查 (Structural Checks)

| 检查项 | 阈值 | 严重度 |
|--------|------|--------|
| 文件数量 | > 50 | medium |
| 总大小 | > 1024KB | medium |
| 单文件大小 | > 256KB | medium |
| 可疑二进制扩展名 | .exe/.dll/.so/.dylib 等 | high |
| 符号链接逃逸 | 指向技能目录外 | critical |

### 5.3 威胁模式匹配 (Threat Pattern Matching)

`THREAT_PATTERNS` 列表包含 40+ 正则匹配规则，分为 8 大类：

| 类别 (category) | 示例模式 | 严重度 |
|-----------------|----------|--------|
| `injection` | 系统提示覆写、角色切换、DAN 越狱 | critical |
| `exfiltration` | 数据外传到 URL | high |
| `code_execution` | `eval()`, `exec()`, `os.system()` | high |
| `network` | `curl\|wget` 管道到 shell | high |
| `supply_chain` | `pip install`, `npm install` | medium |
| `privilege_escalation` | `sudo`, `setuid`, `NOPASSWD` | critical |
| `persistence` | 修改 AGENTS.md、.hermes/config.yaml | critical |
| `credential_exposure` | 硬编码 API key、私钥 | critical |

### 5.4 不可见 Unicode 字符检测 (Invisible Unicode Detection)

```python
INVISIBLE_CHARS = {
    '\u200b',  # zero-width space
    '\u200c',  # zero-width non-joiner
    '\u200d',  # zero-width joiner
    '\u202e',  # right-to-left override (经典注入手法)
    '\ufeff',  # BOM
    # ... 共 16 个字符
}
```

### 5.5 判定与信任等级 (Verdicts & Trust Levels)

**判定 (Verdicts):** `clean` → `suspicious` → `dangerous`

**信任等级 (Trust Levels):**
- `builtin`: Hermes 内置技能
- `trusted`: 来自官方/认证源
- `community`: 社区贡献
- `agent-created`: Agent 自己创建的

**安装策略矩阵 (Install Policy Matrix):**
```
               clean    suspicious    dangerous
builtin:       allow    allow         allow
trusted:       allow    allow         ask
community:     allow    ask           block
agent-created: allow    block         block   ← 最严格
```

## 6. Skills Hub — 搜索、安装与发布 (Search, Install & Publish)

### 6.1 Hub 架构 (Hub Architecture)

```
~/.hermes/skills/.hub/
├── lock.json          # 已安装技能的注册表
├── quarantine/        # 隔离区（扫描未通过）
├── audit.log          # 审计日志
├── taps.json          # 自定义源仓库
└── index-cache/       # 索引缓存（TTL: 3600 秒）
```

**核心数据结构 (`tools/skills_hub.py`):**

```python
@dataclass
class SkillMeta:
    name: str
    description: str
    identifier: str      # 如 "owner/repo/path/skill-name"
    source: str          # "official" | "github" | "clawhub" | "lobehub"
    trust_level: str     # "builtin" | "trusted" | "community"
    tags: list
    extra: dict

@dataclass
class SkillBundle:
    files: Dict[str, Union[str, bytes]]  # 文件名 → 内容
    trust_level: str
```

### 6.2 技能源适配器 (Skill Source Adapters)

```python
class SkillSource(ABC):
    @abstractmethod
    def search(self, query: str) -> List[SkillMeta]: ...
    @abstractmethod
    def fetch(self, identifier: str) -> Optional[SkillBundle]: ...

class GitHubSource(SkillSource): ...
class OptionalSkillSource(SkillSource): ...
# 还有 ClawHub, LobHub 等适配器
```

### 6.3 安装流程 (Installation Flow)

```
hermes skills install <identifier>
  ↓
1. 解析标识符 → 路由到对应 SkillSource
2. 获取 SkillMeta（元数据）
3. 下载 SkillBundle（文件包）
4. scan_skill() 安全扫描
5. should_allow_install() 策略判定
6. 解压到 ~/.hermes/skills/<category>/<name>/
7. 更新 lock.json
8. clear_skills_system_prompt_cache() 使缓存失效
```

### 6.4 发布流程 (Publishing Flow)

`hermes_cli/skills_hub.py:do_publish()`:

```
hermes skills publish <path> --to github --repo owner/repo
  ↓
1. 验证 SKILL.md 存在且有 description
2. scan_skill(path, source="self") 自我扫描
3. 如果 verdict == "dangerous" → 拒绝发布
4. GitHub API 流程:
   a. Fork 目标仓库
   b. 获取默认分支
   c. 创建新分支 add-skill-<name>
   d. 上传所有文件（Base64 编码）
   e. 创建 Pull Request
```

### 6.5 Tap 机制 (Custom Sources)

```bash
hermes skills tap add owner/repo     # 添加自定义源
hermes skills tap remove owner/repo  # 移除
hermes skills tap list               # 列出所有
```

`TapsManager` 管理 `taps.json`，使 `create_source_router()` 包含自定义 GitHub 源。

## 7. 技能配置管理 (Skills Enable/Disable)

`hermes_cli/skills_config.py` 提供 TUI 界面管理技能启用/禁用：

```yaml
# ~/.hermes/config.yaml
skills:
  disabled: [gif-search, notion]      # 全局禁用
  platform_disabled:                   # 平台特定禁用
    linux: [openhue]
  external_dirs:                       # 外部目录
    - ~/team-skills
```

**禁用检查 (`skills_tool.py:_is_skill_disabled()`):**
1. 先检查平台特定禁用列表
2. 再检查全局禁用列表
3. 禁用的技能在 `skills_list()`、系统提示词、斜杠命令中均不显示

## 8. 自我改进模式 (Self-Improvement Patterns)

### 8.1 技能作为程序性记忆 (Skills as Procedural Memory)

系统提示词中强制要求：
```
"After difficult/iterative tasks, offer to save as a skill.
If a skill you loaded was missing steps, had wrong commands, or needed
pitfalls you discovered, update it before finishing."
```

这创造了一个自我改进循环：
1. Agent 遇到问题 → 加载技能
2. 技能缺少步骤或有错误 → Agent 使用 `skill_manage(action='patch')` 更新
3. 下次同类任务 → 技能已改进

### 8.2 Agent 创建技能的安全约束

Agent 通过 `skill_manage` 创建的技能受到最严格的安全策略：
- source = `"agent-created"` 
- `suspicious` verdict → **block**（而非 community 的 ask）
- `dangerous` verdict → **block**
- 扫描失败则回滚整个操作

## 9. 分步教程 — 编写自定义技能 (Step-by-Step Tutorial)

### Step 1: 确定技能范围

技能应该是**窄而深**的，而非**宽而浅**。好的技能：
- ✅ "使用 arXiv API 搜索学术论文"
- ✅ "使用 himalaya CLI 管理邮件"
- ❌ "通用 Python 编程"（太宽泛）

### Step 2: 创建目录结构

```bash
mkdir -p ~/.hermes/skills/my-category/my-skill
```

### Step 3: 编写 SKILL.md

```markdown
---
name: my-api-tool
description: 使用 MyAPI 查询和管理数据的完整工作流
version: 1.0.0
author: Your Name
license: MIT
metadata:
  hermes:
    tags: [API, Data, Automation]
    requires_toolsets: [terminal]
---

# MyAPI Tool

使用 MyAPI 进行数据查询和管理。

## When to Use
- 用户提到 MyAPI 或相关数据源
- 需要查询/更新/删除 MyAPI 数据

## Quick Reference

| 操作 | 命令 |
|------|------|
| 列出数据 | `curl -H "Authorization: Bearer $MYAPI_KEY" https://api.example.com/data` |
| 创建条目 | `curl -X POST -d '{"name": "test"}' ...` |

## Procedure
1. 确认 `MYAPI_KEY` 环境变量已设置
2. 使用 Quick Reference 中的命令进行操作
3. 解析 JSON 响应

## Pitfalls
- Rate limit: 100 次/分钟，注意间隔
- 分页：默认每页 20 条，使用 `?page=2` 翻页

## Verification
- 检查 HTTP 状态码（200 = 成功）
- 确认返回的 JSON 包含预期字段
```

### Step 4: 添加辅助文件（可选）

```bash
# 添加参考文档
mkdir -p ~/.hermes/skills/my-category/my-skill/references
echo "# API 字段说明..." > ~/.hermes/skills/my-category/my-skill/references/fields.md

# 添加脚本
mkdir -p ~/.hermes/skills/my-category/my-skill/scripts
cat > ~/.hermes/skills/my-category/my-skill/scripts/query.py << 'EOF'
#!/usr/bin/env python3
"""Helper script for MyAPI queries."""
import json, sys, urllib.request
# ...
EOF
```

### Step 5: 或让 Agent 创建

```
> 请帮我创建一个管理 MyAPI 的技能

Agent 会调用:
skill_manage(
    action="create",
    name="my-api-tool",
    category="api-tools",
    content="---\nname: my-api-tool\n..."
)
```

### Step 6: 测试技能

```bash
hermes --toolsets skills -q "Use the my-api-tool skill to list all data"
```

### Step 7: 发布技能

```bash
hermes skills publish my-category/my-api-tool --to github --repo myorg/hermes-skills
```

## 10. 设计模式总结 (Design Patterns Summary)

### 10.1 Progressive Disclosure（渐进式披露）
- 系统提示词仅显示 name + description（Tier 1）
- Agent 按需加载完整内容（Tier 2）
- 辅助文件按需读取（Tier 3）
- **动机:** 最小化系统提示词 token 消耗

### 10.2 Write-Audit-Rollback（写入-审计-回滚）
- 每次技能文件变更都经过 `skills_guard` 安全扫描
- 扫描失败则自动回滚到原内容
- Agent 创建的技能采用最严格策略

### 10.3 Source Adapter（源适配器）
- `SkillSource` 抽象基类统一多种技能源
- GitHub、ClawHub、LobHub 等均实现相同接口
- Tap 机制允许添加自定义源

### 10.4 Snapshot Caching（快照缓存）
- `prompt_builder` 使用磁盘快照 + 内存 LRU 缓存
- 避免每次构建系统提示词都扫描文件系统
- `clear_skills_system_prompt_cache()` 在安装/卸载后失效缓存

### 10.5 Convention over Configuration（约定优于配置）
- 目录名 = 类别名（如 `research/arxiv/`）
- 文件位置 = 文件类型（`references/` 放文档、`scripts/` 放脚本）
- SKILL.md 在目录中的存在 = 这是一个技能

### 10.6 Defense in Depth（纵深防御）
- 多层安全：结构检查 → 模式匹配 → Unicode 检测 → 信任等级策略
- 路径安全：`has_traversal_component()` + `validate_within_dir()`
- 注入检测：斜杠命令加载时也检查 `_INJECTION_PATTERNS`

### 10.7 Self-Improvement Loop（自我改进循环）
- 技能 = Agent 的程序性记忆
- Agent 发现技能不完整 → 自动 patch 更新
- 困难任务完成后 → 提议保存为新技能
- 形成"使用 → 改进 → 使用"的正反馈循环

## 11. 内置技能类别索引 (Built-in Skill Categories)

从源码 `skills/` 目录扫描到的类别：

| 类别 | 示例技能 |
|------|----------|
| `software-development` | writing-plans, tdd, systematic-debugging, subagent-driven-development |
| `research` | arxiv, blogwatcher, llm-wiki, polymarket, research-paper-writing |
| `productivity` | notion, linear, google-workspace, powerpoint, ocr-and-documents |
| `social-media` | xitter |
| `smart-home` | openhue |
| `red-teaming` | godmode |

## 12. 关键源码文件索引 (Key Source File Index)

| 文件 | 行数 | 职责 |
|------|------|------|
| `tools/skill_manager_tool.py` | 761 | Agent CRUD 操作：create/edit/patch/delete/write_file/remove_file |
| `tools/skills_tool.py` | 1419 | 渐进式技能展示：skills_list + skill_view |
| `tools/skills_guard.py` | 928 | 安全扫描：威胁模式 + 结构检查 + Unicode 检测 |
| `tools/skills_hub.py` | 3053 | Hub 库：SkillSource/SkillBundle/LockFile/TapsManager |
| `agent/skill_commands.py` | 368 | 斜杠命令注册与技能消息构建 |
| `agent/skill_utils.py` | 465 | 共享工具：frontmatter 解析、平台匹配、条件提取 |
| `agent/prompt_builder.py` | ~200 行 | 系统提示词中的技能索引生成 |
| `hermes_cli/skills_hub.py` | 1238 | CLI 层：install/search/inspect/publish/audit/tap |
| `hermes_cli/skills_config.py` | ~200 行 | TUI 启用/禁用管理 |

## 13. 工具注册信息 (Tool Registration)

```python
# skills_list — 列出技能（Tier 1）
registry.register(
    name="skills_list",
    toolset="skills",
    schema=SKILLS_LIST_SCHEMA,  # 参数: category (可选)
    handler=skills_list,
    emoji="📚",
)

# skill_view — 查看技能内容（Tier 2/3）
registry.register(
    name="skill_view",
    toolset="skills",
    schema=SKILL_VIEW_SCHEMA,  # 参数: name (必填), file_path (可选)
    handler=skill_view,
    emoji="📚",
)

# skill_manage — Agent CRUD
registry.register(
    name="skill_manage",
    toolset="skills",
    schema=SKILL_MANAGE_SCHEMA,
    # 参数: action, name, content, category, old_string, new_string,
    #        file_path, file_content, replace_all
    handler=skill_manage,
    emoji="📚",
)
```

## 14. 环境变量安全收集 (Secure Environment Variable Collection)

技能加载时 (`skill_view`) 的环境变量处理：

1. 解析 `required_environment_variables` 字段
2. 检查 `~/.hermes/.env` 中是否已持久化
3. 如果缺少且是 CLI 会话 → 安全提示用户输入
4. 如果是 Gateway 会话 → 提示用户运行 `hermes setup`
5. 已设置的变量通过 `register_env_passthrough()` 传递到沙箱
6. 向 Agent 仅暴露元数据（`stored_as`, `skipped`, `validated`），**绝不暴露密钥值**

```python
# 返回给 Agent 的安全字段
result = {
    "required_environment_variables": required_env_vars,
    "missing_required_environment_variables": remaining_missing,
    "setup_needed": bool(remaining_missing),
    "setup_skipped": capture_result["setup_skipped"],
    "readiness_status": "setup_needed" | "available",
}
```

## 15. 插件技能 (Plugin Skills)

技能也可以通过插件系统提供，使用限定名称 `namespace:skill-name`:

```python
# 解析限定名
namespace, bare = parse_qualified_name("superpowers:writing-plans")

# 加载插件技能
plugin_skill_md = get_plugin_manager().find_plugin_skill(name)

# Bundle 上下文横幅
banner = f"[Bundle context: This skill is part of the '{namespace}' plugin.\n"
         f"Sibling skills: {sib_list}.]"
```

插件技能同样经过注入检测（`_INJECTION_PATTERNS`），但信任级别由插件系统管理。
