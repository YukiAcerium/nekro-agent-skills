# Nekro-Agent 插件开发完整指南

> 本文档旨在提供一份全面、系统、实用的 Nekro-Agent 插件开发指南，帮助开发者从零开始掌握插件开发的核心知识和高级技巧。

## 目录

1. [插件开发概述](#1-插件开发概述)
2. [插件结构](#2-插件结构)
3. [核心概念](#3-核心概念)
4. [配置系统](#4-配置系统)
5. [存储机制](#5-存储机制)
6. [提示词注入](#6-提示词注入)
7. [事件系统](#7-事件系统)
8. [高级功能](#8-高级功能)
9. [最佳实践](#9-最佳实践)
10. [CI/CD](#10-cicd)
11. [发布流程](#11-发布流程)
12. [示例插件解析](#12-示例插件解析)
13. [常见问题](#13-常见问题)
14. [参考资源](#14-参考资源)

---

## 1. 插件开发概述

### 1.1 什么是 Nekro-Agent 插件？

Nekro-Agent 插件是一种扩展 Nekro-Agent 核心功能的方式。通过插件，开发者可以为 Agent 添加新的工具、信息源、交互逻辑，甚至与其他服务和集成。每个插件都是一个独立的 Python 模块，通过 Nekro-Agent 提供的 API 与核心系统进行交互。

**核心特点：**
- **模块化设计**：每个插件都是独立的 Python 包
- **沙盒执行**：AI 生成的代码在隔离环境中执行，插件方法通过 RPC 调用
- **事件驱动**：支持多种生命周期事件回调
- **配置灵活**：支持用户自定义配置项
- **持久化存储**：内置 KV 存储和文件存储支持

### 1.2 插件能做什么？

插件几乎可以实现任何你希望 Agent 完成的任务：

**增强 AI 能力：**
- 提供专业领域的知识库查询
- 执行复杂的数学计算或数据分析
- 集成第三方 API（天气查询、新闻聚合、翻译服务等）

**执行具体动作：**
- 发送消息、邮件或通知
- 管理定时任务和提醒
- 控制智能家居设备

**与外部系统交互：**
- 创建 Web API 接入点
- 与其他应用程序进行数据同步
- 处理文件传输和多媒体内容

**个性化用户体验：**
- 根据用户偏好提供定制化内容
- 实现独特的人机交互方式
- 利用向量数据库提供智能语义搜索

### 1.3 插件系统架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Nekro-Agent Core                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Plugin     │  │  Sandbox    │  │  Event System       │  │
│  │  Manager    │  │  Executor   │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │               │                    │               │
│         └───────────────┼────────────────────┘               │
│                         ▼                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Plugin API (RPC)                        │    │
│  │  - message.send_text()                               │    │
│  │  - plugin.store.set/get/delete()                     │    │
│  │  - plugin.get_plugin_data_dir()                      │    │
│  │  - _ctx.fs.mixed_forward_file()                      │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      Plugins                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  builtin/   │  │  workdir/   │  │  cloud/             │  │
│  │  basic.py   │  │  my_plugin/ │  │  community_plugins/ │  │
│  │  note.py    │  │             │  │                     │  │
│  │  emotion.py │  │             │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**架构关键点：**

1. **插件注册**：`NekroPlugin` 类定义插件元信息和功能
2. **沙盒执行与 RPC**：AI 代码在沙盒执行，插件方法通过 RPC 调用
3. **API 交互**：插件通过丰富 API 与核心服务交互
4. **事件驱动**：插件可监听响应各种系统事件
5. **配置与存储**：插件有独立配置和存储空间

### 1.4 开发前置条件

**必备知识：**
- Python 编程基础（熟悉 async/await 和面向对象编程）
- API 概念理解
- Nekro-Agent 基本使用经验

**推荐工具：**
- VSCode 或 PyCharm IDE
- Python 3.10+
- Nekro-Agent 源码或已安装实例

---

## 2. 插件结构

### 2.1 标准目录结构

```
plugins/
├── workdir/                          # 用户开发插件目录
│   └── my_plugin/
│       ├── __init__.py               # 必须：导出 plugin 实例
│       ├── plugin.py                 # 主要插件代码
│       ├── README.md                 # 插件文档
│       └── requirements.txt          # 依赖（可选）
│
├── builtin/                          # 内置插件（系统自带）
│   ├── basic.py
│   ├── note.py
│   ├── emotion.py
│   ├── history_travel.py
│   └── draw/
│       ├── __init__.py
│       ├── handlers/
│       └── plugin.py
│
└── cloud/                            # 社区插件市场
```

### 2.2 核心文件结构

#### `__init__.py` - 插件入口

```python
# hello_plugin/__init__.py
from .plugin import plugin

__all__ = ["plugin"]
```

**关键点：**
- 必须导出名为 `plugin` 的 `NekroPlugin` 实例
- 使得 Nekro-Agent 插件加载机制能够找到并导入插件

#### `plugin.py` - 插件核心代码

```python
# hello_plugin/plugin.py
from nekro_agent.api.plugin import NekroPlugin, SandboxMethodType
from nekro_agent.api.schemas import AgentCtx

# 1. 创建插件实例
plugin = NekroPlugin(
    name="我的插件",
    module_name="my_plugin",
    description="插件描述",
    version="1.0.0",
    author="YourName",
    url="https://github.com/yourname/my_plugin",
    support_adapter=["onebot_v11", "telegram"],
)

# 2. 注册沙盒方法
@plugin.mount_sandbox_method(
    method_type=SandboxMethodType.TOOL,
    name="say_hello",
    description="返回一个问候语"
)
async def say_hello(_ctx: AgentCtx) -> str:
    return "Hello from Plugin!"
```

### 2.3 插件命名规范

| 属性 | 规范 | 示例 |
|------|------|------|
| `module_name` | 蛇形命名，唯一标识 | `my_cool_plugin` |
| `author` | 英文字母、数字、下划线 | `KroMiose` |
| `key` | 自动生成 `author.module_name` | `KroMiose.my_cool_plugin` |

---

## 3. 核心概念

### 3.1 插件实例 (`NekroPlugin`)

`NekroPlugin` 是每个插件的核心对象，用于定义元信息和挂载各种功能。

```python
from nekro_agent.api.plugin import NekroPlugin

plugin = NekroPlugin(
    name="我的酷炫插件",           # 显示名称
    module_name="my_cool_plugin",  # 模块标识（唯一）
    description="这个插件能做很多酷炫的事情。",
    version="1.0.0",
    author="开发者名称",
    url="https://github.com/developer/my_cool_plugin",
    support_adapter=["onebot_v11", "telegram"],  # 支持的适配器
    is_builtin=False,              # 是否内置插件
    is_package=False,              # 是否为包结构
)
```

**参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | str | UI 中显示的插件名称 |
| `module_name` | str | 插件唯一标识，与目录名一致 |
| `description` | str | 插件功能描述 |
| `version` | str | 版本号，推荐语义化版本 |
| `author` | str | 作者名（英文） |
| `url` | str | 插件仓库或主页地址 |
| `support_adapter` | List[str] | 支持的适配器列表 |
| `is_builtin` | bool | 是否内置插件 |
| `is_package` | bool | 是否为包结构 |

### 3.2 插件生命周期

```
加载阶段          运行阶段              卸载阶段
   │                │                    │
   ▼                ▼                    ▼
┌───────┐      ┌───────────┐       ┌───────────┐
│ Init  │ ───► │  Normal   │ ───►  │ Cleanup   │
│ (一次)│      │ Operation │       │ (一次)    │
└───────┘      └───────────┘       └───────────┘
   │                │                    │
   ▼                ▼                    ▼
初始化          接收消息/调用           释放资源
资源准备        处理事件/执行          断开连接
连接服务        响应提示词             清理状态
```

#### 3.2.1 初始化 (`@plugin.mount_init_method()`)

插件首次加载时执行，适合一次性设置工作。

```python
@plugin.mount_init_method()
async def initialize_plugin():
    """插件初始化"""
    # 创建数据目录
    data_dir = plugin.get_plugin_data_dir()
    (data_dir / "cache").mkdir(exist_ok=True)
    
    # 初始化数据库
    await init_database()
    
    # 连接外部服务
    await connect_external_service()
    
    # 初始化向量数据库集合
    await init_vector_collection()
```

**用途：**
- 资源准备（创建目录、初始化数据库）
- 状态初始化（加载默认配置）
- 外部系统连接
- 环境验证

#### 3.2.2 会话重置回调 (`@plugin.mount_on_channel_reset()`)

会话重置时执行。

```python
@plugin.mount_on_channel_reset()
async def handle_channel_reset(_ctx: AgentCtx):
    """清理会话相关数据"""
    await plugin.store.delete(
        chat_key=_ctx.chat_key,
        store_key="session_cache"
    )
```

#### 3.2.3 用户消息回调 (`@plugin.mount_on_user_message()`)

收到用户消息时执行。

```python
from nekro_agent.api.message import ChatMessage
from nekro_agent.api.signal import MsgSignal

@plugin.mount_on_user_message()
async def handle_user_message(_ctx: AgentCtx, message: ChatMessage) -> MsgSignal | None:
    """处理用户消息"""
    
    # 消息过滤
    if "禁止词" in message.content_text:
        return MsgSignal.BLOCK_ALL
    
    # 特殊命令处理
    if message.content_text.startswith("!plugin:"):
        await _ctx.send_text(f"收到命令: {message.content_text}")
        return MsgSignal.BLOCK_TRIGGER
    
    return None  # 继续正常处理
```

**返回值说明：**

| 返回值 | 说明 |
|--------|------|
| `None` / `MsgSignal.CONTINUE` | 正常处理 |
| `MsgSignal.BLOCK_TRIGGER` | 记录但阻止 AI 响应 |
| `MsgSignal.BLOCK_ALL` | 阻止记录和响应 |

#### 3.2.4 清理方法 (`@plugin.mount_cleanup_method()`)

插件卸载时执行。

```python
@plugin.mount_cleanup_method()
async def cleanup_plugin():
    """释放资源"""
    # 关闭数据库连接
    await close_database()
    
    # 清理临时文件
    await clear_temporary_files()
    
    # 断开外部连接
    await disconnect_service()
```

### 3.3 沙盒方法详解

沙盒方法是插件向 AI 提供功能的主要接口。

#### 3.3.1 注册沙盒方法

```python
@plugin.mount_sandbox_method(
    method_type=SandboxMethodType.TOOL,
    name="calculate_sum",
    description="计算两个数字的和"
)
async def my_sum_function(_ctx: AgentCtx, num1: int, num2: int) -> int:
    """计算并返回两个整数的和
    
    Args:
        num1: 第一个加数
        num2: 第二个加数
        
    Returns:
        两个数字的和
    """
    return num1 + num2
```

#### 3.3.2 沙盒方法类型 (`SandboxMethodType`)

| 类型 | 用途 | 返回值 | AI 行为 |
|------|------|--------|---------|
| `TOOL` | 直接工具调用 | 任意可序列化类型 | 使用返回值继续任务 |
| `AGENT` | 需要 AI 处理的操作 | 字符串 | 触发新一轮回复 |
| `BEHAVIOR` | 执行副作用操作 | 字符串 | 记录结果，不立即回复 |
| `MULTIMODAL_AGENT` | 多模态内容处理 | 消息段列表 | 触发新一轮回复 |

**示例：**

```python
# TOOL - AI 直接使用结果
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "get_weather", "获取天气")
async def get_weather(_ctx: AgentCtx, city: str) -> dict:
    return {"city": city, "temp": 25, "weather": "晴朗"}

# AGENT - AI 根据结果进行新一轮对话
@plugin.mount_sandbox_method(SandboxMethodType.AGENT, "search_web", "网络搜索")
async def search_web(_ctx: AgentCtx, query: str) -> str:
    results = await do_search(query)
    return f"搜索结果：{results}"

# BEHAVIOR - 执行操作并记录结果
@plugin.mount_sandbox_method(SandboxMethodType.BEHAVIOR, "send_reminder", "发送提醒")
async def send_reminder(_ctx: AgentCtx, time: str, message: str) -> str:
    await schedule_reminder(time, message)
    return f"提醒已设置：{message}"
```

#### 3.3.3 编写规范

**重要规范：**

1. **函数签名**：必须是 `async def`，第一个参数 `_ctx: AgentCtx`
2. **类型注解**：所有参数和返回值必须有类型注解
3. **文档字符串**：必须包含详细描述、参数说明、返回值说明、示例

```python
@plugin.mount_sandbox_method(
    SandboxMethodType.TOOL,
    "get_user_preference",
    "获取用户偏好设置"
)
async def get_user_preference(
    _ctx: AgentCtx,
    user_id: str,
    preference_key: str
) -> Optional[str]:
    """获取用户的偏好设置值
    
    根据用户ID和偏好键名，从插件存储中检索对应的偏好设置值。
    
    Args:
        user_id: 用户的唯一标识符
        preference_key: 偏好设置的键名 (例如: "theme", "language")
        
    Returns:
        偏好值，如果未找到则返回 None
        
    Example:
        ```python
        # 获取用户主题设置
        theme = get_user_preference(user_id="user_123", preference_key="theme")
        if theme:
            print(f"用户主题: {theme}")
        ```
    """
    value = await plugin.store.get(
        user_key=user_id,
        store_key=preference_key
    )
    return value
```

### 3.4 AgentCtx 上下文

`AgentCtx` 是插件开发中最重要的对象，封装了所有上下文信息。

#### 3.4.1 核心属性

```python
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "get_info", "获取信息")
async def get_info(_ctx: AgentCtx) -> dict:
    """获取当前上下文信息"""
    return {
        "chat_key": _ctx.chat_key,
        "channel_id": _ctx.channel_id,
        "channel_name": _ctx.channel_name,
        "channel_type": _ctx.channel_type,
        "adapter_key": _ctx.adapter_key,
    }
```

#### 3.4.2 便捷方法

**消息发送：**

```python
# 发送文本消息
await _ctx.send_text("Hello!")

# 发送图片
await _ctx.send_image(image_path)

# 发送文件
await _ctx.send_file(file_path)

# 推送系统消息（可选触发 AI 响应）
await _ctx.push_system("系统提示", trigger_agent=False)
```

**文件系统：**

```python
# 插件向 AI 传递文件
sandbox_path = await _ctx.fs.mixed_forward_file(
    file_source="https://example.com/image.png",
    file_name="image.png"
)

# AI 向插件传递文件
host_path = _ctx.fs.get_file(sandbox_path)
```

---

## 4. 配置系统

### 4.1 注册配置类

```python
from nekro_agent.api.plugin import ConfigBase, ExtraField
from pydantic import Field
from typing import List, Literal

@plugin.mount_config()
class MyPluginConfig(ConfigBase):
    """插件配置"""
    
    # API 密钥（敏感信息）
    api_key: str = Field(
        default="",
        title="API 密钥",
        description="请输入 API 密钥",
        json_schema_extra=ExtraField(is_secret=True, required=True).model_dump()
    )
    
    # 数值配置
    max_items: int = Field(
        default=10,
        title="最大项目数",
        description="单次操作的最大项目数量"
    )
    
    # 开关配置
    enable_feature: bool = Field(
        default=True,
        title="启用功能 X"
    )
    
    # 枚举选择
    mode: Literal["fast", "accurate", "balanced"] = Field(
        default="balanced",
        title="处理模式"
    )
    
    # 模型组选择
    model_group: str = Field(
        default="default-chat",
        title="使用的模型组",
        json_schema_extra=ExtraField(ref_model_groups=True, model_type="chat").model_dump()
    )
    
    # 多行文本
    system_prompt: str = Field(
        default="你是一个智能助手",
        title="系统提示词",
        json_schema_extra=ExtraField(is_textarea=True).model_dump()
    )
    
    # 列表配置
    allowed_users: List[str] = Field(
        default=[],
        title="允许的用户 ID",
        json_schema_extra=ExtraField(is_textarea=True, sub_item_name="用户ID").model_dump()
    )
```

### 4.2 ExtraField 扩展属性

```python
from nekro_agent.api.plugin import ExtraField

# 敏感信息（输入时隐藏）
is_secret=True

# 多行文本输入
is_textarea=True

# 下拉选择（模型组）
ref_model_groups=True
model_type="chat"  # chat / embedding / draw

# 隐藏配置（仅配置文件）
is_hidden=True

# 需要重启生效
is_need_restart=True

# 可在频道级别覆盖
overridable=True

# 占位提示
placeholder="请输入..."

# 必填
required=True
```

### 4.3 访问配置

```python
# 获取配置实例
config = plugin.get_config(MyPluginConfig)

# 访问配置项
api_key = config.api_key
max_items = config.max_items
```

---

## 5. 存储机制

### 5.1 KV 键值存储 (`plugin.store`)

#### 5.1.1 核心 API

```python
# 设置数据
await plugin.store.set(
    chat_key="",          # 会话标识（可选）
    user_key="",          # 用户标识（可选）
    store_key="key",      # 存储键名（必需）
    value="value"         # 存储值（必需）
)

# 获取数据
value = await plugin.store.get(
    chat_key="",
    user_key="",
    store_key="key"
)  # 不存在返回 None

# 删除数据
await plugin.store.delete(
    chat_key="",
    user_key="",
    store_key="key"
)
```

#### 5.1.2 作用域说明

| 作用域 | 参数 | 说明 |
|--------|------|------|
| 会话级 | `chat_key` | 数据与特定会话绑定 |
| 用户级 | `user_key` | 数据与特定用户绑定（跨会话） |
| 全局级 | 无 key | 数据属于插件本身 |

#### 5.1.3 完整示例

```python
from pydantic import BaseModel
from typing import Dict, Optional
import json

class Note(BaseModel):
    title: str
    content: str
    created_at: float

class UserNotes(BaseModel):
    notes: Dict[str, Note] = {}

@plugin.mount_sandbox_method(SandboxMethodType.BEHAVIOR, "add_note", "添加笔记")
async def add_note(_ctx: AgentCtx, title: str, content: str) -> str:
    """添加笔记"""
    # 获取现有数据
    data = await plugin.store.get(
        user_key=_ctx.from_user_id,
        store_key="notes"
    )
    user_notes = UserNotes.model_validate_json(data) if data else UserNotes()
    
    # 添加笔记
    new_note = Note(
        title=title,
        content=content,
        created_at=time.time()
    )
    user_notes.notes[title] = new_note
    
    # 保存
    await plugin.store.set(
        user_key=_ctx.from_user_id,
        store_key="notes",
        value=user_notes.model_dump_json()
    )
    
    return f"笔记已添加: {title}"
```

### 5.2 插件持久化目录

```python
from pathlib import Path

# 获取插件数据目录
plugin_dir: Path = plugin.get_plugin_data_dir()

# 创建子目录
cache_dir = plugin_dir / "cache"
cache_dir.mkdir(parents=True, exist_ok=True)

# 保存文件
log_file = plugin_dir / "logs" / "app.log"

# 使用 aiofiles 进行异步文件操作
import aiofiles

async def save_data(content: str):
    async with aiofiles.open(plugin_dir / "data.txt", "w") as f:
        await f.write(content)
```

---

## 6. 提示词注入

### 6.1 基础用法

```python
@plugin.mount_prompt_inject_method(
    name="user_context_injection",
    description="向 AI 注入用户上下文"
)
async def inject_user_context(_ctx: AgentCtx) -> str:
    """注入用户偏好和状态信息"""
    
    # 获取用户偏好
    prefs = await plugin.store.get(
        user_key=_ctx.from_user_id,
        store_key="preferences"
    )
    
    # 获取会话状态
    mode = await plugin.store.get(
        chat_key=_ctx.chat_key,
        store_key="mode"
    )
    
    prompt_parts = []
    
    if prefs:
        prompt_parts.append(f"用户偏好: {prefs}")
    
    if mode:
        prompt_parts.append(f"当前模式: {mode}")
    
    return "\n".join(prompt_parts)
```

### 6.2 设计原则

1. **简洁明了**：注入内容应简短清晰
2. **高度相关**：只注入与当前情境相关的信息
3. **动态生成**：根据上下文动态生成
4. **避免冲突**：注意与其他注入内容协调

---

## 7. 事件系统

### 7.1 事件类型汇总

| 装饰器 | 触发时机 | 用途 |
|--------|----------|------|
| `@mount_init_method()` | 插件加载 | 初始化资源 |
| `@mount_cleanup_method()` | 插件卸载 | 释放资源 |
| `@mount_on_channel_reset()` | 会话重置 | 清理会话数据 |
| `@mount_on_user_message()` | 收到用户消息 | 消息预处理/拦截 |
| `@mount_on_system_message()` | 系统消息 | 系统事件响应 |
| `@mount_prompt_inject_method()` | 对话开始 | 注入提示词 |

### 7.2 完整事件示例

```python
from nekro_agent.api.schemas import AgentCtx
from nekro_agent.api.message import ChatMessage
from nekro_agent.api.signal import MsgSignal

# 初始化
@plugin.mount_init_method()
async def init():
    plugin_dir = plugin.get_plugin_data_dir()
    (plugin_dir / "data").mkdir(exist_ok=True)

# 用户消息处理
@plugin.mount_on_user_message()
async def on_message(_ctx: AgentCtx, message: ChatMessage) -> MsgSignal | None:
    # 权限检查
    if not await check_permission(_ctx.from_user_id):
        return MsgSignal.BLOCK_ALL
    
    # 内容过滤
    if contains_banned_words(message.content_text):
        await _ctx.send_text("内容包含敏感词")
        return MsgSignal.BLOCK_TRIGGER
    
    return None

# 会话重置
@plugin.mount_on_channel_reset()
async def on_reset(_ctx: AgentCtx):
    await plugin.store.delete(
        chat_key=_ctx.chat_key,
        store_key="temp_state"
    )

# 清理
@plugin.mount_cleanup_method()
async def cleanup():
    await close_all_connections()
```

---

## 8. 高级功能

### 8.1 动态路由

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

@plugin.mount_router()
def create_router() -> APIRouter:
    router = APIRouter()
    
    @router.get("/")
    async def index():
        return {"message": "Welcome to plugin API"}
    
    @router.post("/users")
    async def create_user(user_data: UserCreate):
        return {"id": 1, **user_data.dict()}
    
    return router
```

**访问路径：** `/plugins/{author}.{module_name}/{path}`

### 8.2 向量数据库

```python
from nekro_agent.api import core
from qdrant_client import models

@plugin.mount_init_method()
async def init_vector_db():
    """初始化向量数据库集合"""
    client = await core.get_qdrant_client()
    
    collection_name = plugin.get_vector_collection_name()
    
    # 检查并创建集合
    collections = await client.get_collections()
    if collection_name not in [c.name for c in collections.collections]:
        await client.create_collection(
            collection_name=collection_name,
            vectors_config=models.VectorParams(
                size=1024,  # 向量维度
                distance=models.Distance.COSINE
            )
        )

@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "search_similar", "语义搜索")
async def search_similar(_ctx: AgentCtx, query: str) -> str:
    """搜索相似内容"""
    # 生成查询向量
    embedding = await generate_embedding(query)
    
    # 搜索
    client = await core.get_qdrant_client()
    results = await client.search(
        collection_name=plugin.get_vector_collection_name(),
        query_vector=embedding,
        limit=5
    )
    
    return str(results)
```

### 8.3 动态包导入

```python
from nekro_agent.api.plugin import dynamic_import_pkg

@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "fetch_url", "获取网页")
async def fetch_url(_ctx: AgentCtx, url: str) -> str:
    """使用 requests 获取网页内容"""
    
    requests = dynamic_import_pkg("requests>=2.25.0")
    
    response = requests.get(url, timeout=10)
    return f"状态码: {response.status_code}"
```

### 8.4 文件交互

```python
# 插件向 AI 传递文件
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "generate_chart", "生成图表")
async def generate_chart(_ctx: AgentCtx, data: dict) -> str:
    """生成图表并返回给 AI"""
    # 生成图片...
    chart_path = "/path/to/chart.png"
    
    # 转换为 AI 可访问的沙盒路径
    sandbox_path = await _ctx.fs.mixed_forward_file(chart_path, file_name="chart.png")
    
    return sandbox_path

# AI 向插件传递文件
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "analyze_file", "分析文件")
async def analyze_file(_ctx: AgentCtx, file_path: str) -> str:
    """分析 AI 提供的文件"""
    # 获取宿主机真实路径
    host_path = _ctx.fs.get_file(file_path)
    
    # 读取文件
    async with aiofiles.open(host_path, "r") as f:
        content = await f.read()
    
    return f"文件内容长度: {len(content)} 字符"
```

---

## 9. 最佳实践

### 9.1 代码规范

```python
# ✅ 推荐：清晰的结构和注释
async def calculate_statistics(
    _ctx: AgentCtx,
    data: List[float],
    mode: str = "mean"
) -> dict:
    """计算数据统计值
    
    Args:
        data: 输入数据列表
        mode: 计算模式 (mean/sum/max/min)
        
    Returns:
        包含统计结果的字典
    """
    import statistics
    
    if mode == "mean":
        result = statistics.mean(data)
    elif mode == "sum":
        result = sum(data)
    # ...
    
    return {"mode": mode, "result": result}

# ❌ 不推荐：缺少类型注解和文档
async def calc(d, m):
    return sum(d) if m == "sum" else 0
```

### 9.2 错误处理

```python
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "fetch_data", "获取数据")
async def fetch_data(_ctx: AgentCtx, url: str) -> str:
    """从 URL 获取数据"""
    
    # 参数验证
    if not url:
        raise ValueError("URL 不能为空")
    
    try:
        # 使用 dynamic_import_pkg 导入
        requests = dynamic_import_pkg("requests>=2.25.0")
        
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        
        return response.text[:1000]  # 限制返回长度
        
    except requests.Timeout:
        return "请求超时"
    except requests.RequestException as e:
        return f"请求失败: {e}"
```

### 9.3 性能优化

```python
# ✅ 推荐：使用异步操作
import aiofiles

async def read_large_file(file_path: str) -> str:
    async with aiofiles.open(file_path, "r") as f:
        content = await f.read()
    return content

# ✅ 推荐：批量操作
async def batch_save(items: List[dict]):
    for item in items:
        await plugin.store.set(
            store_key=item["key"],
            value=json.dumps(item["value"])
        )

# ❌ 不推荐：同步阻塞操作
def sync_read(file_path):
    with open(file_path, "r") as f:
        return f.read()  # 阻塞事件循环
```

### 9.4 安全性

```python
# ✅ 推荐：敏感信息使用 ExtraField(is_secret=True)
class Config(ConfigBase):
    api_key: str = Field(
        default="",
        json_schema_extra=ExtraField(is_secret=True).model_dump()
    )

# ✅ 推荐：用户输入验证
@plugin.mount_sandbox_method(SandboxMethodType.TOOL, "process", "处理")
async def process(_ctx: AgentCtx, user_input: str) -> str:
    # 清理输入
    clean_input = user_input.strip()
    
    # 长度限制
    if len(clean_input) > 1000:
        raise ValueError("输入过长")
    
    # 特殊字符处理
    if any(char in clean_input for char in ["\x00", "\r", "\n"]):
        raise ValueError("输入包含非法字符")
    
    return process_input(clean_input)
```

### 9.5 配置最佳实践

```python
@plugin.mount_config()
class MyPluginConfig(ConfigBase):
    """插件配置"""
    
    # 提供合理的默认值
    timeout: int = Field(default=30, ge=1, le=300)
    
    # 清晰的中英文描述
    enable_feature: bool = Field(
        default=True,
        json_schema_extra=ExtraField(
            i18n_title=i18n.i18n_text(zh_CN="启用功能", en_US="Enable Feature"),
        ).model_dump()
    )
    
    # 敏感信息保护
    api_key: str = Field(
        default="",
        json_schema_extra=ExtraField(is_secret=True).model_dump()
    )
```

---

## 10. CI/CD

### 10.1 GitHub Actions 工作流

```yaml
# .github/workflows/test.yml
name: Test Plugin

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          
      - name: Install dependencies
        run: |
          pip install -q pytest pytest-asyncio
          
      - name: Run tests
        run: |
          pytest tests/ -v
```

### 10.2 代码质量检查

```yaml
# .github/workflows/quality.yml
name: Code Quality

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install lint tools
        run: pip install ruff black mypy
      
      - name: Run ruff
        run: ruff check .
      
      - name: Run black
        run: black --check .
      
      - name: Run mypy
        run: mypy plugin.py --ignore-missing-imports
```

### 10.3 自动发布

```yaml
# .github/workflows/release.yml
name: Release

on:
  release:
    types: [created]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build package
        run: |
          pip install build
          python -m build
      
      - name: Create release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 11. 发布流程

### 11.1 插件打包结构

```
my_plugin/
├── __init__.py              # 导出 plugin
├── plugin.py                # 主代码
├── README.md                # 英文文档
├── README_CN.md             # 中文文档（推荐）
├── requirements.txt         # 依赖
├── pyproject.toml           # 包配置
└── tests/
    └── test_plugin.py
```

### 11.2 README 模板

```markdown
# My Plugin

Brief description of the plugin.

## Features

- Feature 1
- Feature 2
- Feature 3

## Installation

```bash
# Copy to plugins/workdir/
```

## Configuration

| Config | Type | Default | Description |
|--------|------|---------|-------------|
| api_key | str | "" | API key |

## Usage

```python
# Example usage
```

## License

MIT
```

### 11.3 发布到社区市场

1. 整理插件代码和文档
2. 创建 GitHub 仓库
3. 提交到 Nekro-Agent 社区市场审核
4. 通过后用户可从市场安装

---

## 12. 示例插件解析

### 12.1 basic.py - 基础交互插件

**核心功能：**
- 发送文本/图片/文件消息
- 智能防刷屏机制
- 适配器特定功能（如获取 QQ 头像）

**关键设计：**

```python
# 1. 配置定义
@plugin.mount_config()
class BasicConfig(ConfigBase):
    SIMILARITY_MESSAGE_FILTER: bool = Field(default=True)
    SIMILARITY_THRESHOLD: float = Field(default=0.7)

# 2. 消息缓存
SEND_MSG_CACHE: Dict[str, List[str]] = {}

# 3. 相似度检查
@plugin.mount_sandbox_method(...)
async def send_msg_text(...):
    # 检查重复消息
    if message_text in recent_messages:
        # 清空缓存或抛出异常
        pass
    
    # 检查相似度
    for recent_msg in recent_messages:
        similarity = calculate_text_similarity(message_text, recent_msg)
        if similarity > config.SIMILARITY_THRESHOLD:
            # 发送警告
            pass

# 4. 动态方法收集
@plugin.mount_collect_methods()
async def collect_available_methods(_ctx: AgentCtx) -> List[Callable]:
    if _ctx.adapter_key == "onebot_v11":
        return [send_msg_text, send_msg_file, get_user_avatar]
    return [send_msg_text, send_msg_file]
```

**学习要点：**
- 配置系统的完整使用
- 缓存机制实现防刷屏
- 根据适配器动态暴露不同方法

### 12.2 note.py - 笔记系统插件

**核心功能