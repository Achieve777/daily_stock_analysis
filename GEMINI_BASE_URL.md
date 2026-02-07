# GEMINI_BASE_URL 功能说明

## 概述

本次更新为项目添加了对 `GEMINI_BASE_URL` 环境变量的支持，允许使用第三方兼容 Gemini API 的服务，而不仅限于 Google 官方 API。

## 修改内容

### 1. 环境变量配置 (.env.example)

在 `.env.example` 文件的 Gemini 配置部分添加了新的配置项：

```bash
GEMINI_API_KEY=
# Gemini API Base URL（可选，用于第三方兼容 Gemini 的 API 服务）
# 留空则使用官方 API 地址
# GEMINI_BASE_URL=https://your-gemini-compatible-api.com/v1beta
GEMINI_MODEL=gemini-3-flash-preview
GEMINI_MODEL_FALLBACK=gemini-2.5-flash
```

### 2. 配置管理模块 (src/config.py)

#### 2.1 添加配置字段

在 `Config` dataclass 中添加了 `gemini_base_url` 字段：

```python
# === AI 分析配置 ===
gemini_api_key: Optional[str] = None
gemini_base_url: Optional[str] = None  # Gemini API Base URL (用于第三方兼容 API)
gemini_model: str = "gemini-3-flash-preview"  # 主模型
gemini_model_fallback: str = "gemini-2.5-flash"  # 备选模型
```

#### 2.2 加载环境变量

在 `_load_from_env()` 方法中添加了环境变量读取：

```python
gemini_api_key=os.getenv('GEMINI_API_KEY'),
gemini_base_url=os.getenv('GEMINI_BASE_URL'),
gemini_model=os.getenv('GEMINI_MODEL', 'gemini-3-flash-preview'),
```

### 3. AI 分析器模块 (src/analyzer.py)

#### 3.1 更新 _init_model() 方法

修改了 Gemini 模型初始化逻辑，支持自定义 Base URL：

```python
# 配置 API Key 和 Base URL
configure_kwargs = {"api_key": self._api_key}

# 如果配置了自定义 Base URL，则使用它
if config.gemini_base_url:
    configure_kwargs["client_options"] = {
        "api_endpoint": config.gemini_base_url
    }
    logger.info(f"使用自定义 Gemini API Base URL: {config.gemini_base_url}")

genai.configure(**configure_kwargs)
```

#### 3.2 更新 _switch_to_fallback_model() 方法

确保切换到备选模型时也使用正确的 Base URL：

```python
# 重新配置 genai (确保使用正确的 base_url)
configure_kwargs = {"api_key": self._api_key}
if config.gemini_base_url:
    configure_kwargs["client_options"] = {
        "api_endpoint": config.gemini_base_url
    }
genai.configure(**configure_kwargs)
```

## 使用方法

### 场景 1: 使用官方 Gemini API（默认）

在 `.env` 文件中只需配置 API Key：

```bash
GEMINI_API_KEY=your_api_key_here
# GEMINI_BASE_URL 留空或不配置
```

### 场景 2: 使用第三方兼容 Gemini 的 API

在 `.env` 文件中同时配置 API Key 和 Base URL：

```bash
GEMINI_API_KEY=your_api_key_here
GEMINI_BASE_URL=https://your-gemini-compatible-api.com/v1beta
```

## 兼容性说明

- **向后兼容**: 如果不配置 `GEMINI_BASE_URL`，系统将继续使用官方 Gemini API，保持与旧版本的完全兼容
- **第三方 API 要求**: 第三方 API 服务必须完全兼容 Google Generative AI SDK 的接口规范
- **配置优先级**: Base URL 配置在模型初始化和切换备选模型时都会生效

## 日志输出

当使用自定义 Base URL 时，系统会在日志中输出相关信息：

```
INFO: 使用自定义 Gemini API Base URL: https://your-api.com/v1beta
INFO: Gemini 模型初始化成功 (模型: gemini-3-flash-preview)
```

## 测试

提供了测试脚本 `test_gemini_base_url.py` 用于验证配置是否正确加载：

```bash
python3 test_gemini_base_url.py
```

## 相关文件

- `.env.example` - 环境变量配置模板
- `src/config.py` - 配置管理模块
- `src/analyzer.py` - AI 分析器模块
- `test_gemini_base_url.py` - 功能测试脚本

## 注意事项

1. `GEMINI_BASE_URL` 是可选配置项，留空或不配置则使用官方 API
2. 配置的 Base URL 应该是第三方服务的完整 API endpoint 地址
3. 第三方 API 必须与 Google Generative AI SDK 的 `client_options.api_endpoint` 参数兼容
4. 建议在使用第三方 API 前先测试其兼容性
