# 插件开发指南

## 概述

本指南介绍如何为小智ESP32语音助手开发自定义插件。插件系统允许开发者扩展语音助手的功能，添加新的服务能力。

## 插件系统架构

插件系统基于以下组件：
1. **插件注册** - 使用装饰器自动注册插件函数
2. **插件加载** - 系统启动时自动加载所有插件
3. **插件执行** - 统一工具处理器调度插件执行
4. **配置管理** - 在配置文件中管理插件参数

## 创建新插件的步骤

### 1. 创建插件文件

在 `plugins_func/functions/` 目录下创建新的Python文件，文件名应与插件功能相关。

```python
import requests
from plugins_func.register import register_function, ToolType, ActionResponse, Action
from config.logger import setup_logging

# 设置日志
TAG = __name__
logger = setup_logging()

# 定义函数描述（供LLM理解如何调用）
PLUGIN_FUNCTION_DESC = {
    "type": "function",
    "function": {
        "name": "your_plugin_name",
        "description": "插件功能的详细描述，清楚说明插件的作用和使用场景",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "参数1的描述",
                },
                "param2": {
                    "type": "integer",
                    "description": "参数2的描述",
                },
            },
            "required": ["必需参数列表"],
        },
    },
}

# 使用装饰器注册函数
@register_function("your_plugin_name", PLUGIN_FUNCTION_DESC, ToolType.SYSTEM_CTL)
def your_plugin_name(conn, param1: str = None, param2: int = 0):
    """
    插件函数实现
    
    Args:
        conn: 连接对象，包含配置信息和上下文
        param1: 字符串参数示例
        param2: 整数参数示例
        
    Returns:
        ActionResponse: 包含执行结果的响应对象
    """
    try:
        # 从配置中获取插件参数
        config = conn.config.get("plugins", {}).get("your_plugin_name", {})
        api_key = config.get("api_key", "")
        default_setting = config.get("default_setting", "default_value")
        
        # 实现插件功能逻辑
        # 例如调用外部API、处理数据等
        result = "执行结果"
        
        # 返回成功响应
        return ActionResponse(Action.REQLLM, result, None)
        
    except Exception as e:
        # 记录错误日志
        logger.error(f"插件执行失败: {e}")
        # 返回错误响应
        return ActionResponse(Action.ERROR, None, f"插件执行失败: {str(e)}")
```

### 2. 插件函数返回类型说明

插件函数应返回 `ActionResponse` 对象，包含以下动作类型：

- `Action.REQLLM` - 执行完插件后请求LLM生成回复
- `Action.RESPONSE` - 直接回复用户，不经过LLM
- `Action.ERROR` - 发生错误时返回
- `Action.NONE` - 不执行任何操作

### 3. 插件工具类型说明

注册插件时需要指定工具类型：

- `ToolType.SYSTEM_CTL` - 系统控制类插件（如播放音乐、退出等）
- `ToolType.IOT_CTL` - IoT设备控制类插件（需要conn参数）
- `ToolType.WAIT` - 需要等待执行结果的插件
- `ToolType.CHANGE_SYS_PROMPT` - 修改系统提示词的插件

### 4. 在配置文件中添加插件配置

在 `data/.config.yaml` 文件的 `plugins` 部分添加插件配置：

```yaml
plugins:
  your_plugin_name:
    # 插件所需的配置参数
    api_key: "your_api_key"
    default_setting: "value"
    timeout: 30
```

### 5. 在意图识别中启用插件

在 `data/.config.yaml` 文件的意图识别配置中添加插件到函数列表：

```yaml
Intent:
  function_call:
    type: function_call
    functions:
      - your_plugin_name  # 添加你的插件名称
```

## 插件开发最佳实践

### 1. 错误处理
始终使用try-except块捕获异常，并返回适当的错误信息。

### 2. 日志记录
使用系统提供的日志功能记录插件执行过程中的重要信息。

### 3. 配置管理
从连接对象的配置中获取插件参数，确保配置的灵活性。

### 4. 参数验证
验证输入参数的有效性，避免因参数错误导致插件执行失败。

### 5. 缓存机制
对于频繁调用且数据变化不频繁的功能，考虑实现缓存机制提高性能。

## 示例插件

以下是一个简单的问候插件示例：

```python
from plugins_func.register import register_function, ToolType, ActionResponse, Action
from datetime import datetime

GREETING_FUNCTION_DESC = {
    "type": "function",
    "function": {
        "name": "get_greeting",
        "description": "根据当前时间提供问候语",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string",
                    "description": "用户的姓名",
                },
            },
        },
    },
}

@register_function("get_greeting", GREETING_FUNCTION_DESC, ToolType.SYSTEM_CTL)
def get_greeting(conn, name: str = None):
    current_hour = datetime.now().hour
    
    if current_hour < 12:
        greeting = "早上好"
    elif current_hour < 18:
        greeting = "下午好"
    else:
        greeting = "晚上好"
        
    if name:
        message = f"{greeting}，{name}！很高兴为您服务。"
    else:
        message = f"{greeting}！很高兴为您服务。"
        
    return ActionResponse(Action.REQLLM, message, None)
```

## 测试插件

1. 重启服务使插件生效
2. 通过语音或文本测试插件功能
3. 检查日志确认插件正常工作
4. 验证错误处理机制

## 常见问题

### 1. 插件未加载
- 确认插件文件位于正确的目录
- 检查插件文件名是否符合Python模块命名规范
- 确认使用了正确的装饰器注册函数

### 2. 配置无法读取
- 确认配置项名称与代码中的一致
- 检查配置文件格式是否正确
- 验证配置项是否在正确的配置节下

### 3. 参数传递错误
- 确认函数参数与描述中的定义一致
- 检查参数类型是否正确
- 验证必需参数是否提供