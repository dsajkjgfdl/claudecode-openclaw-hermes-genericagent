---
Task: T18-R
Status: DRAFT_DONE
Date: 2026-04-15
Chapter: 18 — 技能系统 (Skills System)
Keywords: 技能发现/加载/执行、Skills Hub 生态、自我改进、安全沙箱
Sources:
  - agent/skill_commands.py (368 lines)
  - agent/skill_utils.py (465 lines)
  - agent/prompt_builder.py (1043 lines — skills sections)
  - hermes_cli/skills_config.py (177 lines)
  - hermes_cli/skills_hub.py (1238 lines)
  - tools/skills_tool.py (1419 lines)
  - tools/skill_manager_tool.py (761 lines)
  - tools/skills_guard.py (928 lines)
  - tools/skills_sync.py (316 lines)
  - skills/ directory (bundled skills)
  - CONTRIBUTING.md (SKILL.md format spec)
---

# Ch18 研究笔记：技能系统 (Skills System)

## 1. 总体架构概览 (Architecture Overview)

Hermes Agent 的技能系统 (Skills System) 是一个 **程序性记忆层** (procedural memory layer)，将可复用的工作流知识编码为结构化 Markdown 文件 (SKILL.md)。整个系统由 6 个核心模块组成：

```
┌───────────────────────────────────────────────────────────────┐
│                    Skills System Architecture                  │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │ prompt_     │    │ skill_       │    │ skills_hub.py   │  │
│  │ builder.py  │◄───│ commands.py  │    │ (CLI/Slash)     │  │
│  │ (Injection) │    │ (/ Commands) │    │ browse/search/  │  │
│  └──────┬──────┘    └──────┬───────┘    │ install/audit   │  │
│         │                  │            └────────┬────────┘  │
│         ▼                  ▼                     ▼           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              skills_tool.py (Core Engine)             │    │
│  │   skills_list() / skill_view() — Discovery & Load     │    │
│  └──────────────────┬────────────────────────────────────┘    │
│                     │                                        │
│  ┌─────────────┐    │    ┌──────────────┐  ┌──────────────┐  │
│  │ skill_      │    │    │ skills_      │  │ skills_      │  │
│  │ manager_    │◄───┼───►│ guard.py     │  │ sync.py      │  │
│  │ tool.py     │    │    │ (Security)   │  │ (Bundled     │  │
│  │ (CRUD)      │    │    └──────────────┘  │  Seeding)    │  │
│  └─────────────┘    │                      └──────────────┘  │
│                     ▼                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │          ~/.hermes/skills/ (Filesystem Storage)       │    │
│  │   skills/<category>/<skill-name>/SKILL.md             │    │
│  │   skills/<category>/DESCRIPTION.md                    │    │
│  │   skills/.bundled_manifest                            │    │
│  └──────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

**核心设计理念 (Core Design Philosophy):**
- 技能是 **纯文本 Markdown** — 不是可执行代码，而是 **指令文档**
- Agent 在系统提示中看到技能索引，**按需加载** (lazy loading) 完整内容
- Agent 可以 **自主创建和改进技能** — 实现自我进化 (self-improvement)
- 多层安全扫描 (multi-layer security scanning) 防护恶意内容

## 2. 技能存储与目录结构 (Storage & Directory Structure)

### 2.1 文件系统布局 (Filesystem Layout)

```
~/.hermes/skills/                              # SKILLS_DIR — 单一数据源
├── .bundled_manifest                          # 内置技能同步清单 (v2 格式)
├── software-development/                      # 类别目录 (Category)
│   ├── DESCRIPTION.md                         # 类别描述 (category-level)
│   ├── writing-plans/                         # 技能目录 (Skill directory)
│   │   ├── SKILL.md                           # 主技能文件 (Main skill file)
│   │   ├── references/                        # 参考文档 (Reference docs)
│   │   │   └── api.md
│   │   ├── templates/                         # 模板文件 (Templates)
│   │   │   └── config.yaml
│   │   ├── scripts/                           # 脚本文件 (Scripts)
│   │   │   └── validate.py
│   │   └── assets/                            # 补充资源 (Supplementary files)
│   │       └── logo.png
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   └── subagent-driven-development/
├── mlops/                                     # 另一个类别
│   ├── training/
│   ├── inference/
│   └── evaluation/
├── research/
├── productivity/
├── media/
├── social-media/
├── smart-home/
└── red-teaming/
```

**关键常量 (Key Constants):**
- `SKILLS_DIR = get_hermes_home() / "skills"` — 在 `skills_tool.py` 和 `skills_sync.py` 中定义
- `MANIFEST_FILE = SKILLS_DIR / ".bundled_manifest"` — 同步清单位置
- `EXCLUDED_SKILL_DIRS = frozenset((".git", ".github", ".hub"))` — 扫描时排除

### 2.2 外部技能目录 (External Skills Directories)

技能系统支持通过 `config.yaml` 配置外部目录：

```yaml
skills:
  external_dirs:
    - /path/to/shared/skills
    - /path/to/team/skills
```

- `get_external_skills_dirs()` 从配置中读取
- `get_all_skills_dirs()` 返回 `[SKILLS_DIR, *external_dirs]`
- **本地优先 (Local takes precedence):** 名称冲突时本地技能覆盖外部
- 外部目录是 **只读** — 新技能始终创建在本地目录

## 3. SKILL.md 格式规范 (SKILL.md Format Specification)

### 3.1 YAML Frontmatter 字段

SKILL.md 使用 `---` 分隔的 YAML 前置元数据 (frontmatter) + Markdown 正文 (body)：

```yaml
---
name: writing-plans                    # 技能名称 (required)
description: "Creates implementation plans..."  # 简短描述 (required)
version: 1.1.0                         # 语义版本号
author: Hermes Agent                   # 作者
license: MIT                           # 许可证
platforms:                             # 平台限制 (可选)
  - macos
  - linux
required_environment_variables:        # 需要的环境变量
  - name: OPENAI_API_KEY
    description: "OpenAI API key"
    optional: false
    help: "Get your key at https://..."
required_credential_files:             # 需要的凭证文件
  - ~/.config/gcloud/credentials.json
metadata:
  hermes:                              # Hermes 专有扩展
    tags: [planning, workflow]         # 标签
    related_skills: [tdd, debugging]   # 关联技能
    fallback_for_toolsets: [browser]   # 当该 toolset 可用时隐藏 (条件激活)
    requires_toolsets: [terminal]      # 需要该 toolset 才显示
    fallback_for_tools: [web_search]   # 当该 tool 可用时隐藏
    requires_tools: [terminal]         # 需要该 tool 才显示
    config:                            # 技能配置变量
      - key: wiki.path
        description: Path to wiki directory
        default: "~/wiki"
        prompt: Wiki directory path
compatibility:                         # agentskills.io 兼容性字段
  agents: ["hermes", "claude"]
---

# Skill Title

## Overview
...instructions for the agent...
```

### 3.2 前置元数据解析 (Frontmatter Parsing)

`agent/skill_utils.py` 中的 `parse_frontmatter(content)` 解析流程：

1. 检测 `---` 起始行
2. 查找第二个 `---` 关闭标记
3. 使用简单的键值解析 (非完整 YAML 库) 提取字段
4. 返回 `(frontmatter_dict, body_text)` 元组

**设计选择:** 使用轻量级自定义解析器而非完整 YAML 库，减少依赖并提高解析速度。

### 3.3 条件激活 (Conditional Activation)

`extract_skill_conditions(frontmatter)` 提取四种条件字段：

| 字段 | 含义 |
|------|------|
| `fallback_for_toolsets` | 当指定 toolset 可用时 **隐藏** 此技能 |
| `requires_toolsets` | 需要指定 toolset 可用才 **显示** 此技能 |
| `fallback_for_tools` | 当指定 tool 可用时 **隐藏** 此技能 |
| `requires_tools` | 需要指定 tool 可用才 **显示** 此技能 |

**应用场景:** 一个技能可以是 web_search 工具的 fallback — 当 web_search 不可用时自动激活，提供替代工作流。

### 3.4 技能配置变量 (Skill Config Variables)

技能可以声明需要的配置项，存储在 `~/.hermes/config.yaml` 中 `skills.config.*` 前缀下：

```python
SKILL_CONFIG_PREFIX = "skills.config"
# e.g., key "wiki.path" → stored as skills.config.wiki.path in config.yaml
```

- `extract_skill_config_vars()` — 提取声明的配置变量
- `resolve_skill_config_values()` — 从 config.yaml 中读取当前值
- `discover_all_skill_config_vars()` — 扫描所有启用技能的配置声明
- 配置值在技能激活时注入 `[Skill config: ...]` 块

## 4. 技能发现与加载 (Discovery & Loading)

### 4.1 发现流程 (_find_all_skills)

`tools/skills_tool.py` 中的 `_find_all_skills()` 执行完整扫描：

```
1. 扫描 SKILLS_DIR + get_external_skills_dirs()
2. 递归查找所有 SKILL.md (rglob)
3. 排除 .git/.github/.hub 目录
4. 解析 frontmatter → 获取 name, description, category, tags, version
5. 去重: seen_names 集合 — 同名技能只保留首次发现的
6. 过滤禁用的技能 (_is_skill_disabled)
7. 平台匹配 (skill_matches_platform)
8. 返回 metadata dict 列表
```

### 4.2 平台过滤 (Platform Filtering)

```python
PLATFORM_MAP = {"macos": "darwin", "linux": "linux", "windows": "win32"}
```

`skill_matches_platform(frontmatter)` 检查技能的 `platforms` 字段与当前 `sys.platform` 是否匹配。无 platforms 字段 = 全平台适用。

### 4.3 禁用机制 (Disable Mechanism)

`hermes_cli/skills_config.py` 管理技能的启用/禁用：

```yaml
# ~/.hermes/config.yaml
skills:
  disabled:              # 全局禁用
    - skill-a
    - skill-b
  platform_disabled:     # 按平台禁用
    macos:
      - macos-only-skill
    linux:
      - linux-only-skill
```

- `hermes skills enable/disable <name>` — CLI 命令
- `hermes skills enable/disable <name> --platform macos` — 按平台
- `_get_disabled_skill_names()` 合并全局 + 当前平台的禁用列表

### 4.4 技能就绪状态 (SkillReadinessStatus)

```python
class SkillReadinessStatus(str, Enum):
    AVAILABLE = "available"        # 可用
    SETUP_NEEDED = "setup_needed"  # 需要配置
    UNSUPPORTED = "unsupported"    # 平台不支持
```

`skill_view()` 在加载技能时检查：
- 环境变量是否设置 (`required_environment_variables`)
- 凭证文件是否存在 (`required_credential_files`)
- 缺失项触发 `SETUP_NEEDED` 状态

## 5. 技能注入系统提示 (Skill Injection into System Prompt)

### 5.1 两层缓存架构 (Two-Layer Caching)

`agent/prompt_builder.py` 的 `build_skills_system_prompt()` 使用精心设计的缓存：

**Layer 1 — 进程内 LRU 缓存 (In-Process LRU):**
```python
_SKILLS_PROMPT_CACHE: OrderedDict  # 最多 _SKILLS_PROMPT_CACHE_MAX 条目
cache_key = (skills_dir, external_dirs, tools, toolsets, platform_hint)
```

**Layer 2 — 磁盘快照 (Disk Snapshot):**
```python
# ~/.hermes/.skills_prompt_snapshot.json
{
    "version": _SKILLS_SNAPSHOT_VERSION,
    "manifest": {"category/skill/SKILL.md": [mtime_ns, size], ...},
    "skills": [{"skill_name": ..., "category": ..., "description": ..., "conditions": ...}],
    "category_descriptions": {"category": "description text"}
}
```

**失效逻辑 (Invalidation):**
- `_build_skills_manifest()` 比较所有 SKILL.md/DESCRIPTION.md 的 mtime + size
- 任何文件变化 → 快照失效 → 重新扫描文件系统
- `clear_skills_system_prompt_cache(clear_snapshot=True)` — 手动清除

### 5.2 技能索引格式 (Skills Index in System Prompt)

生成的系统提示片段：

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even
partially relevant to your task, you MUST load it with skill_view(name)
and follow its instructions. Err on the side of loading...

<available_skills>
  software-development: Development tools and workflows
    - writing-plans: Creates comprehensive implementation plans...
    - test-driven-development: TDD workflow with...
  mlops: Machine learning operations
    - axolotl: Fine-tuning with Axolotl...
</available_skills>

Only proceed without loading a skill if genuinely none are relevant.
```

**关键设计意图:** 系统提示用强制性语言 (mandatory) 要求 Agent **必须** 加载相关技能，确保技能知识被实际使用。

### 5.3 技能激活路径 (Skill Activation Paths)

技能有三种激活方式：

| 激活方式 | 入口 | 消息格式 |
|---------|------|---------|
| **Agent 主动加载** | `skill_view(name)` tool call | JSON 返回含 content, linked_files, setup_note |
| **斜杠命令** | `/skill-name [instruction]` | `_build_skill_message()` 格式化 |
| **会话预加载** | `hermes --skill name` CLI 参数 | `build_preloaded_skills_prompt()` |

#### 斜杠命令激活 (Slash Command Activation)

`agent/skill_commands.py`:
1. `scan_skill_commands()` — 启动时扫描所有技能，构建 `/command → skill_info` 映射
2. 技能名规范化: `name.lower().replace(' ', '-').replace('_', '-')` → 去除非法字符
3. `resolve_skill_command_key(command)` — 用户输入匹配 (连字符/下划线互通)
4. `build_skill_invocation_message()` → `_build_skill_message()` 构建完整消息

#### 消息构建 (_build_skill_message)

```
[SYSTEM: The user has invoked the "skill-name" skill...]   ← 激活通知
                                                             ← 空行
<SKILL.md content>                                           ← 技能内容
                                                             ← 空行
[Skill config (from ~/.hermes/config.yaml):                  ← 配置注入
  wiki.path = ~/wiki
]

[This skill has supporting files you can load:]              ← 支撑文件
- references/api.md
- templates/config.yaml

The user has provided the following instruction:              ← 用户指令
<user_instruction>

[Runtime note: ...]                                          ← 运行时备注
```

### 5.4 技能配置注入 (Skill Config Injection)

`_inject_skill_config()` 在 `skill_commands.py` 中：
1. 解析技能的 frontmatter
2. 提取 `metadata.hermes.config` 声明
3. 从 `config.yaml` 读取实际值
4. 注入 `[Skill config: ...]` 块到消息中

## 6. 技能工具注册 (Tool Registration)

### 6.1 skills_list — 技能列表工具

```python
SKILLS_LIST_SCHEMA = {
    "name": "skills_list",
    "description": "List available skills...",
    "parameters": {
        "properties": {
            "category": {"type": "string", "description": "Optional category filter"}
        }
    }
}
registry.register(name="skills_list", toolset="skills", ...)
```

### 6.2 skill_view — 技能查看工具

```python
SKILL_VIEW_SCHEMA = {
    "name": "skill_view",
    "description": "Load a skill's full content or access linked files...",
    "parameters": {
        "properties": {
            "name": {"type": "string"},
            "file_path": {"type": "string", "description": "Path to linked file..."}
        }
    }
}
registry.register(name="skill_view", toolset="skills", ...)
```

`skill_view()` 两种模式：
- **无 file_path:** 返回 SKILL.md 内容 + frontmatter + linked_files 元数据 + 就绪状态
- **有 file_path:** 返回技能目录内指定文件的内容 (有路径遍历防护)

### 6.3 skill_manage — 技能管理工具

```python
SKILL_MANAGE_SCHEMA = {
    "name": "skill_manage",
    "description": "Manage skills (create, update, delete). Skills are your procedural memory...",
    "parameters": {
        "properties": {
            "action": {"enum": ["create", "patch", "edit", "delete", "write_file", "remove_file"]},
            "name": {"type": "string"},
            "content": {"type": "string"},           # for create/edit
            "old_string": {"type": "string"},         # for patch
            "new_string": {"type": "string"},         # for patch
            "replace_all": {"type": "boolean"},       # for patch
            "category": {"type": "string"},           # for create
            "file_path": {"type": "string"},          # for write_file/remove_file/patch
            "file_content": {"type": "string"}        # for write_file
        }
    }
}
```

六种动作 (Actions):

| Action | 描述 | 安全扫描 |
|--------|------|---------|
| `create` | 创建新技能 (完整 SKILL.md + 可选 category) | ✅ 创建后扫描，失败回滚 |
| `edit` | 完全重写 SKILL.md | ✅ 写入后扫描，失败回滚到原始内容 |
| `patch` | 针对性查找替换 (支持 fuzzy matching) | ✅ patch 后扫描，失败回滚 |
| `delete` | 删除技能目录 + 清理空类别目录 | ❌ |
| `write_file` | 添加/覆盖支撑文件 | ✅ 写入后扫描 |
| `remove_file` | 删除支撑文件 | ❌ |

**Patch 模式亮点:** 使用 `tools.fuzzy_match.fuzzy_find_and_replace` 进行模糊匹配，处理空白差异和缩进不同的情况。

## 7. 安全扫描系统 (Security Scanning — skills_guard.py)

### 7.1 信任级别 (Trust Levels)

```python
# 四个信任级别
TRUST_LEVELS = {
    "builtin":       # 内置技能 — 最高信任
    "trusted":       # 来自受信任仓库
    "community":     # 社区贡献
    "agent-created": # Agent 自主创建
}

TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}
```

### 7.2 安装策略矩阵 (Install Policy Matrix)

```python
INSTALL_POLICY = {
    "builtin":       {"safe": "allow", "caution": "allow", "dangerous": "allow"},
    "trusted":       {"safe": "allow", "caution": "allow", "dangerous": "block"},
    "community":     {"safe": "allow", "caution": "block", "dangerous": "block"},
    "agent-created": {"safe": "allow", "caution": "allow", "dangerous": "block"},
}
```

### 7.3 威胁检测模式 (Threat Detection Patterns)

60+ 正则表达式规则分 6 类：

| 类别 (Category) | 示例模式 |
|-----------------|---------|
| `exfiltration` | curl/wget 上传数据、base64 编码外发 |
| `injection` | 提示注入 (prompt injection)、系统指令覆盖 |
| `destructive` | rm -rf、磁盘格式化 |
| `persistence` | crontab 添加、启动项修改 |
| `obfuscation` | base64 解码执行、eval/exec |
| `network` | 可疑网络连接、反向 shell |

### 7.4 结构限制 (Structural Limits)

```python
MAX_FILE_COUNT = 50         # 每个技能最多文件数
MAX_TOTAL_SIZE_KB = 1024    # 每个技能最大总大小 (1 MB)
SCANNABLE_EXTENSIONS = {".md", ".py", ".sh", ".yaml", ".yml", ".json", ...}
SUSPICIOUS_BINARY_EXTENSIONS = {".exe", ".dll", ".so", ".dylib", ...}
```

### 7.5 不可见字符检测 (Invisible Unicode Detection)

扫描不可见 Unicode 字符（如零宽空格、方向覆盖字符），防止视觉隐藏的注入攻击。

### 7.6 扫描流程 (Scan Flow)

```
scan_skill(skill_dir, source)
  ├─ 结构检查: 文件数、总大小、可疑二进制
  ├─ 遍历每个可扫描文件:
  │   ├─ scan_file(path) → 正则匹配 + Unicode 检查
  │   └─ 收集 Finding(severity, category, line, description)
  ├─ _determine_trust_level(source) → builtin/trusted/community
  ├─ _determine_verdict(findings) → safe/caution/dangerous
  ├─ should_allow_install(verdict, trust_level) → bool
  └─ 返回 ScanResult + format_scan_report()
```

### 7.7 Agent 创建技能的安全扫描

`skill_manager_tool.py` 中 `_security_scan_skill(skill_dir)`:
- 每次 create/edit/patch/write_file 操作后自动扫描
- 扫描失败 → **原子回滚** (atomic rollback) 到操作前状态
- Trust level = `agent-created` → caution 允许, dangerous 阻止

## 8. 技能同步系统 (Skills Sync — skills_sync.py)

### 8.1 清单格式 (Manifest Format)

位于 `~/.hermes/skills/.bundled_manifest`，v2 格式：

```
skill-name-1:md5hash1
skill-name-2:md5hash2
```

v1 格式（纯名称）自动迁移为 v2。

### 8.2 同步逻辑 (Sync Logic)

`sync_skills()` 的决策树：

```
对于每个内置技能:
  ├─ 不在 manifest 中 (NEW):
  │   ├─ 目标目录不存在 → 复制 (copytree), 记录哈希
  │   └─ 目标已存在 → 跳过, 记录哈希
  ├─ 在 manifest 中 + 目标存在 (EXISTING):
  │   ├─ 无 origin hash (v1 迁移) → 设置基准, 跳过
  │   ├─ 用户副本 ≠ origin hash → 用户修改过, 跳过 (保护用户改动)
  │   └─ 用户副本 = origin hash:
  │       ├─ 内置版本 ≠ origin hash → 更新 (backup → copy → remove backup)
  │       └─ 内置版本 = origin hash → 跳过 (无变化)
  └─ 在 manifest 中 + 目标不存在 (USER DELETED):
      → 尊重用户删除, 跳过
```

**关键设计:** 通过 MD5 哈希对比 (`_dir_hash`) 区分用户自定义 vs 上游更新，永远不覆盖用户修改。

### 8.3 原子写入 (Atomic Write)

清单写入使用 `tempfile + os.replace()` 模式，防止进程中断导致文件损坏。更新操作使用 backup + restore 模式。

## 9. Skills Hub 生态 (Skills Hub Ecosystem)

### 9.1 Hub 命令体系

`hermes_cli/skills_hub.py` 提供统一的命令接口：

| 命令 | 功能 | 入口 |
|------|------|------|
| `browse` | 分页浏览可用技能 | CLI + `/skills browse` |
| `search <query>` | 按关键词搜索 | CLI + `/skills search` |
| `install <id>` | 安装技能 (含安全扫描) | CLI + `/skills install` |
| `inspect <id>` | 预览技能 (不安装) | CLI + `/skills inspect` |
| `list` | 列出已安装技能 | CLI + `/skills list` |
| `check [name]` | 检查更新 | CLI + `/skills check` |
| `update [name]` | 应用更新 | CLI + `/skills update` |
| `audit [name]` | 重新安全扫描 | CLI + `/skills audit` |
| `uninstall <name>` | 卸载技能 | CLI + `/skills uninstall` |
| `publish <path>` | 发布技能到 GitHub PR | CLI + `/skills publish` |
| `snapshot export/import` | 导出/导入技能配置 | CLI + `/skills snapshot` |
| `tap list/add/remove` | 管理技能源 (Taps) | CLI + `/skills tap` |

### 9.2 技能来源类型 (Skill Source Types)

```python
# do_list() 区分三种来源
source_types = {
    "hub":     # 从 Skills Hub 安装
    "builtin": # 内置随发行版分发
    "local":   # 用户本地创建 (包括 Agent 创建)
}
```

### 9.3 短名称解析 (_resolve_short_name)

支持多种标识符格式：
- 完整 GitHub 路径: `owner/repo/path/to/skill`
- 短名称: `skill-name`
- Hub 格式: `hub:skill-name`

### 9.4 Tap 系统 (Skill Taps)

类似 Homebrew 的 tap 机制，管理外部技能仓库：
- `hermes skills tap add owner/repo`
- `hermes skills tap remove owner/repo`
- `hermes skills tap list`

### 9.5 快照导出/导入 (Snapshot Export/Import)

支持技能配置的完整导出和导入，用于团队共享或环境迁移。

## 10. 自我改进模式 (Self-Improvement Patterns)

### 10.1 技能作为程序性记忆

`skill_manage` 的描述中明确定义了创建时机：

```
Create when:
- complex task succeeded (5+ calls)
- errors overcome
- user-corrected approach worked
- non-trivial workflow discovered
- user asks you to remember a procedure

Update when:
- instructions stale/wrong
- OS-specific failures
- missing steps or pitfalls found during use
```

### 10.2 自动修复循环

系统提示中的关键指令：
> "If a skill has issues, fix it with skill_manage(action='patch')."
> "If a skill you loaded was missing steps, had wrong commands, or needed pitfalls you discovered, update it before finishing."

这创建了一个 **闭环改进循环 (closed-loop improvement cycle):**
1. Agent 加载技能
2. 执行技能中的步骤
3. 遇到问题或发现更好方法
4. 使用 `skill_manage(action='patch')` 立即更新技能
5. 下次遇到类似任务时使用改进后的技能

### 10.3 创建建议

> "After difficult/iterative tasks, offer to save as a skill."
> "Skip for simple one-offs. Confirm with user before creating/deleting."

Agent 在完成复杂任务后主动建议创建技能，但简单一次性任务不建议。

## 11. 环境变量与凭证管理 (Env Vars & Credentials)

### 11.1 环境变量自动捕获

`skill_view()` 在加载技能时：
1. 解析 `required_environment_variables` frontmatter 字段
2. 检查哪些变量已设置 (`load_env()`)
3. 调用 `_capture_required_environment_variables()` 尝试自动获取缺失变量
4. 注册可用变量到 `env_passthrough` (用于沙箱执行环境)

### 11.2 凭证文件注册

```python
required_credential_files:
  - ~/.config/gcloud/credentials.json
  - ~/.aws/credentials
```

- `register_credential_files()` 将凭证文件注册到远程沙箱挂载系统
- 缺失的凭证文件触发 `setup_needed = True`

### 11.3 远程后端感知

```python
_REMOTE_ENV_BACKENDS = {"modal", "docker", "ssh", ...}
# 远程后端需要在远程环境中也设置这些变量
```

## 12. 路径安全 (Path Security)

### 12.1 skill_view 路径遍历防护

```python
from tools.path_security import validate_within_dir, has_traversal_component

# 1. 检查路径中是否有 '..' 组件
if has_traversal_component(file_path):
    return error("Path traversal ('..') is not allowed.")

# 2. 解析后验证仍在技能目录内
traversal_error = validate_within_dir(target_file, skill_dir)
```

### 12.2 skill_manage 文件路径限制

```python
ALLOWED_SUBDIRS = {"references", "templates", "scripts", "assets"}
# 只允许在这四个子目录下操作文件
```

### 12.3 外部技能安全警告

`skill_view()` 对非 SKILLS_DIR 目录的技能显示安全警告。

## 13. 设计模式总结 (Design Patterns)

### 13.1 Lazy Loading (惰性加载)
- 系统提示只含 **索引** (名称+描述)
- Agent 按需调用 `skill_view()` 加载完整内容
- 减少每次对话的 token 开销

### 13.2 Two-Layer Cache (两层缓存)
- L1: 进程内 OrderedDict LRU
- L2: 磁盘 JSON 快照 (mtime/size 校验)
- 冷启动优化: 避免每次启动都做文件系统扫描

### 13.3 Manifest-Based Sync (基于清单的同步)
- MD5 哈希追踪内置技能的原始版本
- 用户修改检测: 比较当前哈希 vs origin hash
- 三方合并语义: 用户改动优先

### 13.4 Atomic Operations (原子操作)
- `_atomic_write_text()` — tempfile + os.replace
- Backup + restore — 安全扫描失败时回滚
- 清单写入的 fsync + replace

### 13.5 Security-by-Default (默认安全)
- 每次写入操作后自动安全扫描
- Trust level 决定允许的 severity 阈值
- 失败自动回滚 — 不允许危险内容存在

### 13.6 Namespace Deduplication (命名空间去重)
- `seen_names` 集合确保同名技能只出现一次
- 本地优先于外部目录
- 扫描顺序决定优先级

### 13.7 Platform-Aware Filtering (平台感知过滤)
- Frontmatter 声明平台兼容性
- 运行时根据 `sys.platform` 过滤
- 网关支持多平台: `HERMES_SESSION_PLATFORM` 环境变量

## 14. 内置技能类别 (Bundled Skill Categories)

已发现的内置类别和技能：

| 类别 | 示例技能 |
|------|---------|
| `software-development` | writing-plans, test-driven-development, systematic-debugging, subagent-driven-development, requesting-code-review, plan |
| `mlops` | axolotl, (vector-databases, training, inference, evaluation 子类别) |
| `research` | arxiv, llm-wiki, blogwatcher, polymarket, research-paper-writing |
| `productivity` | linear, notion, google-workspace, powerpoint, nano-pdf, ocr-and-documents |
| `media` | (各种媒体处理) |
| `social-media` | xitter (Twitter/X) |
| `smart-home` | openhue |
| `red-teaming` | godmode |
| `note-taking` | (笔记相关) |

## 15. 关键数据流 (Key Data Flows)

### 15.1 启动时技能初始化

```
hermes 启动
  → sync_skills() — 同步内置技能到 ~/.hermes/skills/
  → build_skills_system_prompt() — 构建技能索引
    → 尝试 L1 缓存 → 尝试 L2 磁盘快照 → 回退到文件系统扫描
  → scan_skill_commands() — 构建斜杠命令映射
  → 技能索引注入系统提示
```

### 15.2 用户交互时技能加载

```
用户提问或使用 /skill-name
  → Agent 识别相关技能 (从系统提示索引)
  → Agent 调用 skill_view("skill-name")
    → 查找 SKILL.md
    → 解析 frontmatter
    → 检查环境变量就绪状态
    → 注册 env passthrough
    → 注册 credential files
    → 返回完整内容 + 元数据
  → Agent 按照技能指令执行任务
  → (可选) Agent 使用 skill_manage(action='patch') 改进技能
```

### 15.3 技能安装流程

```
/skills install owner/repo/skill
  → _resolve_short_name() — 解析标识符
  → 下载技能文件
  → scan_skill(skill_dir, source) — 安全扫描
    → 确定 trust_level
    → 确定 verdict (safe/caution/dangerous)
    → should_allow_install() — 检查策略矩阵
  → 通过 → 复制到 ~/.hermes/skills/
  → 阻止 → 显示扫描报告, 拒绝安装
  → clear_skills_system_prompt_cache() — 使缓存失效
```

## 16. 文件清单与代码行数

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `tools/skills_tool.py` | 1419 | 核心发现和加载引擎 |
| `hermes_cli/skills_hub.py` | 1238 | Skills Hub CLI 和斜杠命令 |
| `agent/prompt_builder.py` | 1043 | 系统提示构建 (含技能索引) |
| `tools/skills_guard.py` | 928 | 安全扫描引擎 |
| `tools/skill_manager_tool.py` | 761 | Agent CRUD 操作 |
| `agent/skill_utils.py` | 465 | 轻量级实用工具 |
| `agent/skill_commands.py` | 368 | 斜杠命令处理 |
| `tools/skills_sync.py` | 316 | 内置技能同步 |
| `hermes_cli/skills_config.py` | 177 | 启用/禁用配置 |
| **总计** | **~6,715** | |

---

## 待深入探索 (Further Investigation)

- [ ] `hermes_cli/skills_hub.py` 中 `do_install()` 的详细下载和验证流程
- [ ] `do_publish()` 的 GitHub PR 创建机制
- [ ] Skills Hub 后端 API (skills.sh, well-known, github 等 source types)
- [ ] Plugin namespace 技能 (`plugin:skill-name`) 的完整生命周期
- [ ] `env_passthrough` 和 `credential_files` 在 Modal/Docker/SSH 后端的具体挂载机制
- [ ] agentskills.io 标准的完整兼容性
