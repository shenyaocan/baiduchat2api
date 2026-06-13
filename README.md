# Baidu Chat Reverse Engineering (wenxin2api)

逆向百度文心助手 (chat.baidu.com) 的纯算法实现，支持 OpenAI 兼容 API。

## 逆向成果

### 1. 核心 API 发现
- **主聊天 API**: `POST https://chat.baidu.com/aichat/api/conversation`
- **响应格式**: `text/event-stream` (SSE)
- **Token 获取**: 从页面 HTML 中内联的 `aiTabFrameBaseData` JSON 提取

### 2. Token 生成算法 (已逆向)
```
chat_token = base64("{token}|{MD5(query)}|{timestamp}|{lid}")-{lid}-3
```
- `token` 和 `lid` 从页面初始化数据获取
- `MD5` 使用标准 spark-md5 算法对查询字符串哈希
- `timestamp` 为当前毫秒时间戳

### 3. 支持的模型 (3个)
| OpenAI 模型名 | Baidu 模型 | 说明 |
|---|---|---|
| `baidu-smart` | `smartMode` | 默认智能模式 |
| `baidu-deepseek` | `deepseek` | DeepSeek 深度思考 |
| `baidu-ds-v4` | `ds-v4` | DeepSeek-V4 Pro |

### 4. Thinking (深度思考) 支持
- 通过 `deep_search=True` 启用深度思考
- SSE 响应中解析 `thinking` / `thinking_content` / `reasoning` 字段
- 转换为 OpenAI 的 `reasoning_content` 字段

### 5. 工具调用支持
- 请求中传入 OpenAI 兼容的 `tools` 后，服务端会自动追加系统提示词
- 上游模型按 XML 输出标准工具结构：
```xml
<tool_calls>
  <tool_call>
    <name>get_weather</name>
    <arguments>{"city":"北京"}</arguments>
  </tool_call>
</tool_calls>
```
- 服务端会解析 XML，并返回 OpenAI 兼容的 `message.tool_calls`

### 6. 文件结构
```
├── baidu_chat.py      # 核心逆向客户端 (纯算法)
├── main.py            # OpenAI 兼容 API 服务器
├── config.toml        # 配置文件
├── requirements.txt   # 依赖
└── config/            # 逆向分析产物 (JS 源码等)
```

## 使用方式

### 1. 安装依赖
```bash
pip install -r requirements.txt
```

### 2. 配置 Cookie (可选但推荐)
编辑 `config.toml`，填入从浏览器复制的 Cookie：
```toml
[cookies]
value = "BAIDUID=xxx; BIDUPSID=xxx; ..."
```

### 3. 启动 OpenAI 兼容服务器
```bash
python main.py --config config.toml
# 或指定端口
python main.py --port 8000
```

公网部署时建议在 `config.toml` 配置自定义密钥：
```toml
[auth]
api_keys = ["sk-your-secret-key"]
```

配置后，请求必须携带：
```bash
-H "Authorization: Bearer sk-your-secret-key"
```

Docker 一键部署：
```bash
docker compose up -d --build
```

多 Cookie 负载均衡：
```toml
[cookies]
values = [
  "BAIDUID=xxx; BIDUPSID=xxx; ...",
  "BAIDUID=yyy; BIDUPSID=yyy; ..."
]
```

服务端会为每个 Cookie 建立独立客户端，并按当前并发数均衡分发。某个 Cookie 不可用时会立即刷新当前会话并重试，仍失败则自动切到下一个 Cookie。

上下文压缩与新窗口：
```toml
[context]
fresh_conversation = true
max_chars = 12000
max_messages = 16
max_message_chars = 2000
```

每次请求都会使用新的百度对话窗口；OpenAI `messages` 会在本地压缩后作为单次 prompt 发送，避免百度侧上下文串线。

### 4. API 调用示例
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-secret-key" \
  -d '{
    "model": "baidu-deepseek",
    "messages": [{"role": "user", "content": "1+1等于几"}],
    "stream": true
  }'
```

### 5. 工具调用示例
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-secret-key" \
  -d '{
    "model": "baidu-smart",
    "messages": [{"role": "user", "content": "北京天气怎么样"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather by city",
        "parameters": {
          "type": "object",
          "properties": {"city": {"type": "string"}},
          "required": ["city"]
        }
      }
    }],
    "stream": false
  }'
```

返回中的工具调用：
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "",
      "tool_calls": [{
        "id": "call_xxx",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\":\"北京\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

### 6. CLI 直接调用
```bash
python baidu_chat.py "1+1等于几" --model deepseek
python baidu_chat.py "hello" --model smart
```

## 技术细节

### 逆向分析过程
1. **Phase 1 - 侦察**: 使用浏览器分析 chat.baidu.com 网络请求，识别出 `aichat/api/conversation` SSE 接口
2. **Phase 2 - 静态分析**: 下载并分析 `search-js.js`, `chat-main-pc.js`, `common.js`, `vendors.js` 等 Vite 构建产物
3. **Phase 3 - 动态验证**: Hook fetch API 捕获请求体，确认 `chat_token` 格式和参数结构
4. **Phase 4 - 算法提取**: 从 minified JS 中提取 `getToken$1` 函数，确认使用 `spark-md5` 进行标准 MD5 哈希

### 关键发现
- `token` 和 `lid` 存储在页面 HTML 的 `<script name="aiTabFrameBaseData">` 中
- Token 算法：`base64(token_val | MD5(query_str) | timestamp | lid) - lid - 3`
- 模型通过 `usedModel.modelName` 和 `isDeepseek` header 控制
- SSE 事件类型：`basedata` (初始数据), `ping` (心跳), `message` (内容块)

### 注意事项
- 需要保持 Cookie 有效（特别是登录态相关的 Cookie）
- `token` 和 `lid` 在会话期间有效，过期后需重新获取页面
- 深度思考模型会返回更长的响应，建议增加 timeout
- 由于百度接口可能迭代更新，token 算法需持续验证

## 免责声明
本项目仅供学习交流使用，请遵守百度相关服务条款。不得用于商业用途或非法用途。
