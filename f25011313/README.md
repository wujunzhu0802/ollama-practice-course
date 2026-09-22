# Ollama 模型下载、API 调用区别与扩展能力分析

学号：f25011313  
资料核查日期：2026 年 9 月 22 日  
课程提交入口：[ai-opensource-practice-course](https://gitcode.com/nicezack/ai-opensource-practice-course)

## 阅读说明：模型、接口和应用

Ollama API、OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages 是不同的接口形式，并不是四个模型。

- **模型**负责理解输入和生成输出，例如在 Ollama 中运行的 Qwen。
- **API**规定程序如何发送提示词、传递工具信息、接收结果。
- **应用**管理会话、检索资料、执行工具和展示答案。

Ollama 提供原生 API，也实现部分 OpenAI、Anthropic 兼容接口。因此，同一个本地模型可以通过不同格式被调用。使用 OpenAI SDK 不一定访问 OpenAI 云端，使用 Anthropic SDK 也不一定调用 Claude；实际由服务地址、模型名称和服务端实现共同决定。[Ollama API 说明](https://docs.ollama.com/api/introduction)

## 一、Ollama 模型下载、调用与对外服务

### 1.1 安装与下载

Ollama 是运行和管理模型的服务程序，模型权重需要另外下载。本作业用 `qwen3:0.6b` 做轻量文本示例，模型库标示约 523 MB；这不是运行时总内存，实际还需要推理程序、上下文缓存和操作系统内存。小模型适合验证流程，复杂问答与工具选择可更换更大模型。参考：[模型页面](https://ollama.com/library/qwen3:0.6b)、[Windows 安装说明](https://docs.ollama.com/windows)。

Windows 先从 [Ollama 官网](https://ollama.com/download/windows) 安装，再重新打开 PowerShell：

```powershell
ollama --version
ollama pull qwen3:0.6b
ollama list
ollama run qwen3:0.6b "请用一句中文介绍自己。 /no_think"
ollama ps
```

`pull` 下载模型，`list` 列出本地模型，`run` 发起对话，`ps` 查看已加载模型。下载完成与推理成功是两项独立检查。Windows 桌面程序通常已启动后台服务；只有后台未运行时才执行 `ollama serve`，否则会端口冲突。默认服务地址是 `http://127.0.0.1:11434`。[官方快速开始](https://docs.ollama.com/quickstart)

### 1.2 本机 API 调用

下面用 PowerShell 发送 UTF-8 JSON；显式关闭流式输出，便于第一次观察完整返回：

```powershell
$body = @{
  model = 'qwen3:0.6b'
  messages = @(@{ role = 'user'; content = '用一句话解释人工智能。 /no_think' })
  stream = $false
} | ConvertTo-Json -Depth 8
$reply = Invoke-RestMethod -Method Post `
  -Uri 'http://127.0.0.1:11434/api/chat' `
  -ContentType 'application/json; charset=utf-8' `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
$reply.message.content
```

原生 `/api/chat` 返回的正文位于 `message.content`。直接通过 HTTP 调用时，原生聊天接口默认流式返回逐行 JSON（NDJSON），不能把整个流当作一个 JSON 对象解析；本作业所有基础示例统一使用 `stream: false`。[Ollama Chat API](https://docs.ollama.com/api/chat)

### 1.3 对外服务：局域网与公网

**局域网实验。** 退出正在运行的 Ollama 托盘程序，在新的 PowerShell 窗口中执行：

```powershell
$env:OLLAMA_HOST = '0.0.0.0:11434'
ollama serve
```

此设置只对当前终端及其子进程有效。用 `ipconfig` 找到服务器的局域网 IPv4 地址，其他电脑把请求地址改成 `http://服务器IPv4:11434/api/chat`。`0.0.0.0` 是监听地址，不是客户端应填写的目标地址；对方填写 `localhost` 只会访问对方自己的电脑。Windows 防火墙应仅向可信客户端或可信子网放行该端口，且只用专用网络配置。恢复本机模式时，停止手动服务，清除该终端的 `OLLAMA_HOST`，再启动桌面程序。[Ollama FAQ](https://docs.ollama.com/faq)

**公网服务。** 采用“客户端 → HTTPS 反向代理（鉴权、限流）→ 回环地址 Ollama → 模型”的结构。保留 Ollama 监听 `127.0.0.1:11434`，公网仅开放代理的 HTTPS 端口。公网 IP、域名解析、证书、端口放行和一台持续运行的主机都需要实际配置；单独设置 `OLLAMA_HOST` 不会自动生成公网链接。

下方提供的 Nginx 配置示例，是 **Linux Nginx、代理和 Ollama 同机** 的部署模板。替换 `llm.example.com`、证书路径和密码文件后，将其放入 Nginx 的 `http` 配置上下文；用 `htpasswd` 创建密码文件，再运行 `nginx -t`，成功后加载配置。模板仅允许推理 POST 路由，默认拒绝模型下载、创建和删除接口，并关闭代理缓冲以支持流式响应。[Nginx 鉴权文档](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html)、[代理文档](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

部署后先发送不带凭据的 POST，预期 401；再用 `curl.exe -u student` 发送同一请求，按提示输入代理密码，预期返回模型结果。请求 `/api/pull` 应为 404。代理示例使用 HTTP Basic 认证，OpenAI SDK 中的占位 `api_key='ollama'` 不能代替代理密码；正式 SDK 应用可采用能验证 Bearer Token 的网关，或用自定义 HTTP 客户端配置代理 Basic 认证。

**本地 Ollama 默认不验证 API Key。** 示例里的 `ollama` 是 SDK 占位值，不是密码。`OLLAMA_ORIGINS` 管理浏览器跨域来源，也不是身份认证。Ollama 云端服务另需真实密钥，本作业的调用地址均指向本机。[认证说明](https://docs.ollama.com/api/authentication)

### 1.4 公网代理配置示例

```nginx
# Linux Nginx template, included inside the http {} context.
# Replace example domain, certificates and password file before deployment.
# Ollama must run on the SAME host at 127.0.0.1:11434.
limit_req_zone $binary_remote_addr zone=ollama_rate:10m rate=5r/m;
limit_conn_zone $binary_remote_addr zone=ollama_conn:10m;

server {
    listen 443 ssl;
    server_name llm.example.com;
    ssl_certificate /etc/letsencrypt/live/llm.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/llm.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    client_max_body_size 2m;

    # Only inference routes: never expose /api/pull, /api/create, /api/delete.
    location ~ ^/(api/(chat|generate|embed)|v1/(chat/completions|responses|messages|embeddings))$ {
        auth_basic "Ollama course service";
        auth_basic_user_file /etc/nginx/ollama.htpasswd;
        limit_except POST { deny all; }
        limit_req zone=ollama_rate burst=3 nodelay;
        limit_req_status 429;
        limit_conn ollama_conn 2;
        proxy_pass http://127.0.0.1:11434;
        proxy_http_version 1.1;
        proxy_set_header Host localhost:11434;
        proxy_set_header Connection "";
        proxy_set_header Authorization "";
        proxy_set_header X-Api-Key "";
        proxy_buffering off;
        proxy_read_timeout 300s;
    }

    location / { return 404; }
}
```

### 1.5 实验验收

依次检查：`/api/version` 返回版本；`ollama list` 出现模型；本机调用有非空正文；可信局域网客户端调用成功；公网代理验证未授权拒绝、授权成功和管理路由隔离。首轮请求可能包含模型加载时间，不能用一次总耗时断言某种接口更快。

## 二、四种调用方式的区别

以下以“调用同一台 Ollama 服务中的同一个模型”为比较前提。涉及厂商云端独有能力时，另行说明。

### 1. Ollama API：面向本地模型的原生接口

**核心特点：** 直接使用 Ollama 自己的请求格式，便于控制推理参数与模型运行状态。

- 聊天路径为 `POST /api/chat`；单次文本生成另有 `/api/generate`。
- 聊天输入主要是 `model` 和 `messages`，生成正文位于 `message.content`。
- 可使用 `options` 传递原生推理参数，使用 `keep_alive` 控制模型保持加载的时间。
- 除生成接口外，还有下载、查看、创建模型及生成嵌入向量等接口。

典型请求结构：

```json
{
  "model": "qwen3:0.6b",
  "messages": [
    {"role": "user", "content": "请解释什么是机器学习。"}
  ],
  "stream": false
}
```

**适合场景：** 从头编写本地模型程序、需要控制模型加载或使用 Ollama 原生参数的项目。

**局限：** 与其他厂商接口不完全相同，迁移时需要调整字段和响应解析。接口支持某项能力，也不代表所有模型都支持它。

参考：[Ollama Chat API](https://docs.ollama.com/api/chat)。

### 2. OpenAI Chat Completions：以对话消息为中心

**核心特点：** 把多轮对话表示为 `messages` 列表，围绕聊天消息进行交互。

- 请求路径为 `POST /v1/chat/completions`。
- 输入使用 `messages`；系统指令可通过 system 消息传入。
- 输出包含 `choices`，通常从 `choices[0].message.content` 读取正文。
- 工具调用请求通常放在 `message.tool_calls` 中；应用执行后通过 tool 消息回传结果。

接入本地 Ollama 时，OpenAI SDK 的 `base_url` 通常设置为：

```text
http://localhost:11434/v1
```

**适合场景：** 已经基于 OpenAI 聊天接口开发的聊天机器人、问答系统或框架，希望以较少改动接入 Ollama。

**局限：** 在常规消息交互中，应用需要管理并发送相关历史。切换服务地址可以复用接口格式，但仍需检查模型名称、参数、工具和多模态支持。

参考：[Ollama 的 OpenAI 兼容说明](https://docs.ollama.com/api/openai-compatibility)。

### 3. OpenAI Responses：以输入和输出项为中心

**核心特点：** 不只表示聊天消息，还用类型化条目表示工具调用、工具结果等交互内容。

- 请求路径为 `POST /v1/responses`。
- 输入通常使用 `input`，系统说明可放在 `instructions`。
- 原始响应使用 `output` 数组，其中可以包含不同类型的条目。
- OpenAI Python SDK 提供 `response.output_text` 便捷属性；读取原始 HTTP JSON 时，应从输出条目中提取文本，不能假设存在同名顶层字段。
- OpenAI 官方服务提供会话状态管理及多种内置工具，适用于更复杂的应用流程。

**与 Chat Completions 的主要区别：** Chat Completions 把交互组织为消息；Responses 把消息、函数调用、函数结果等组织为独立条目，便于表达不同阶段的工作。Responses 并不意味着应用的所有自定义函数都会被服务器自动执行。

**在 Ollama 中的重要限制：** 当前兼容文档说明 Responses 仅支持无状态模式，不支持用 `previous_response_id` 或 `conversation` 保存和续接会话。OpenAI 云端的内置工具也不能因为路径相同就被视为 Ollama 已提供的功能。

**适合场景：** 已使用 Responses 的项目，或希望按输出条目组织消息与工具流程的应用；使用 Ollama 时要先核查所需功能是否受支持。

参考：[OpenAI Responses 迁移指南](https://developers.openai.com/api/docs/guides/migrate-to-responses)、[Ollama 兼容范围](https://docs.ollama.com/api/openai-compatibility)。

### 4. Anthropic Messages：以内容块组织消息

**核心特点：** 消息内容由不同类型的内容块组成，正文和工具请求需要按类型分别处理。

- 请求路径为 `POST /v1/messages`。
- 输入包括 `model`、`messages` 和 `max_tokens`；系统指令放在顶层 `system`，而不是写成 messages 中的 system 角色。
- 输出 `content` 是内容块数组，读取正文时应筛选 `type="text"`。
- 工具请求使用 `tool_use` 内容块，工具结果使用 `tool_result` 内容块。
- 工具定义使用 `input_schema` 描述参数，与 OpenAI 常用的 function 定义格式不同。

Anthropic SDK 接入 Ollama 时，基础地址通常为 `http://localhost:11434`，由 SDK 添加 `/v1/messages`。

**适合场景：** 已使用 Anthropic SDK 或 Messages 格式的应用，希望接入兼容的 Ollama 模型。

**局限：** Ollama 只实现部分兼容能力。例如当前文档列出提示缓存、批处理和 PDF 文档内容块等限制。使用该协议不会让本地模型获得 Claude 模型本身的能力。

参考：[Ollama 的 Anthropic 兼容说明](https://docs.ollama.com/api/anthropic-compatibility)。

### 5. 从实际开发角度比较

**输入与返回结构不同。** 原生 Ollama 和 Chat Completions 都常用 messages 输入，但正文的返回位置不同；Responses 使用 input 和 output 条目；Anthropic Messages 使用内容块。切换接口时不能只改 URL，还要修改对应的解析逻辑。

**多轮上下文的管理方式不同。** 本地 Ollama 的这些调用通常需要应用保存并回传相关历史。OpenAI 官方 Responses 提供服务端会话机制，但不能直接套用到 Ollama 的无状态兼容实现。

**流式处理不同。** 原生 Ollama HTTP 生成接口通常采用逐行 JSON，即 NDJSON；兼容接口一般采用 SSE。即使都设置 `stream=true`，增量数据的位置、事件类型和结束标记仍可能不同，应使用对应 SDK 或解析器。

**工具调用格式不同。** 四种接口都可在相应服务与模型支持的前提下表达工具调用，但函数名称、参数和结果回传格式不同。自定义工具仍需应用实际执行。

**认证与服务商不同。** 本地 Ollama 默认无需认证，SDK 中的 `api_key="ollama"` 只是占位值。厂商云端或另加的网关需使用其真实认证方式。协议相同不代表认证规则相同。[Ollama 认证说明](https://docs.ollama.com/api/authentication)

**接口名称不能直接决定效果和速度。** 模型、量化、上下文、采样设置、推理控制和硬件都会影响结果。同一模型换接口不必然变得更聪明，也不能据接口名称判断谁最快；比较时应统一条件并重复测试。

### 6. 如何选择

- **需要 Ollama 原生控制：** 选择 Ollama API。
- **已有 OpenAI 聊天项目：** 优先复用 Chat Completions。
- **已有 Responses 工作流：** 使用 Responses，并核查 Ollama 的无状态与工具限制。
- **已有 Anthropic 应用：** 使用 Messages，并适配内容块与兼容边界。

选择的依据是项目已有代码、所需功能和服务端兼容程度。API 兼容主要减少接入改动，不保证模型行为完全一致。

## 三、还可以给模型什么“外挂”？

这里的“外挂”主要指由应用接入的外部资料、工具和工作流程。模型负责理解和选择，应用负责提供数据、执行动作与管理权限。

### 1. RAG 知识库：补充课程和业务资料

**作用：** 让模型结合指定资料回答，例如课程讲义、产品手册和企业知识库。

**实现过程：** 文档分段 → 嵌入模型生成向量 → 检索相关片段 → 将原文与问题一起交给模型 → 回答并标明来源。

**边界：** RAG 通常不修改模型权重。检索不到资料或资料本身有误时，答案仍可能不可靠；应保留出处，并允许模型说明证据不足。单独调用嵌入接口不等于已经完成 RAG。

参考：[Ollama Embeddings](https://docs.ollama.com/capabilities/embeddings)。

### 2. Function Calling：接入计算器和业务工具

**作用：** 让模型借助外部程序完成精确计算、天气查询、数据库查询等操作。

**实现过程：** 应用提供工具名称和参数说明 → 模型提出调用请求 → 应用校验参数并执行 → 将工具结果回传 → 模型组织答案。

例如让模型计算“12345 × 6789”，应用可通过计算器得到 83810205，再让模型解释结果。

**边界：** 模型提出调用不代表执行已经发生。应用应限制可调用的工具、校验参数，并为操作失败设置退出或重试策略。

参考：[Ollama Tool Calling](https://docs.ollama.com/capabilities/tool-calling)。

### 3. MCP：统一连接不同系统

**作用：** 使用标准协议接入文件、数据库及其他业务系统，减少为每个工具单独编写连接逻辑。

**实现方式：** 具备 MCP 客户端能力的宿主应用连接 MCP 服务器，获得工具、资源或提示模板，再将相关信息提供给模型。

**与 Function Calling 的关系：** Function Calling 解决模型如何表达“调用什么工具、传什么参数”；MCP 解决应用如何发现和连接外部能力。两者可以组合使用。

**边界：** MCP 不是直接安装进模型权重的插件。单独运行 Ollama 通常还需要宿主应用，才能把 MCP 服务与模型调用衔接起来。

参考：[MCP 架构说明](https://modelcontextprotocol.io/docs/learn/architecture)。

### 4. 联网搜索：补充实时信息

**作用：** 获取模型训练数据之外的新资料，例如新闻、最新文档和网页内容。

**实现方式：** 应用调用搜索或网页读取工具，整理结果及来源，再交给模型分析。

**边界：** 本地模型不会因为能够生成文本就自动联网。搜索结果也需要核验日期、来源和内容质量，网页中的文字不能自动成为执行指令。

### 5. 长期记忆：保存可复用信息

**作用：** 保存用户明确需要记录的偏好、项目进度或历史摘要，让后续对话更连贯。

**实现方式：** 将信息写入数据库，下一轮检索相关记录后加入上下文。

**边界：** 持久化记忆与一次请求中的历史消息不同，也不等于模型重新训练。应用需要提供更新、删除和纠错机制，避免持续使用过时信息。

### 6. 多模态工具：连接图片和语音

**作用：** 扩展文字之外的输入和输出方式。

- OCR：把图片中的文字转为文本。
- 语音识别：把录音转为文字。
- 语音合成：把模型回答读出来。
- 视觉模型：直接处理兼容格式的图片输入。

**边界：** 给纯文本模型接 OCR，可以让它处理识别后的文字，但不能据此说它已经具备完整的图像理解能力；直接视觉推理仍需对应模型支持。

参考：[Ollama Vision](https://docs.ollama.com/capabilities/vision)。

### 7. 工作流与 Agent：组织多步任务

**作用：** 把“理解任务、检索资料、调用工具、检查结果、生成回复”连接成一个执行流程。

**实现方式：** 应用定义步骤、状态、循环与终止条件，根据模型输出选择下一步操作。

**边界：** Agent 的行动能力来自模型与应用的组合。步骤越多，越需要处理错误、设置调用预算和记录执行情况；外部写入或删除权限不能仅凭模型判断授予。

### 8. 提示模板、结构化输出与模型定制

这三类扩展解决不同问题：

- **提示模板或技能说明：** 规定回答角色、步骤和任务规范，提高行为的一致性。
- **结构化输出：** 用 JSON Schema 等约束输出形状，方便程序提取字段；结构正确不代表事实正确。
- **微调或 LoRA：** 调整模型在特定任务上的行为或风格，涉及训练数据、适配器和模型兼容性。

Ollama 的 Modelfile 可以配置系统提示、参数及兼容的适配器，但仅设置系统提示并不等于完成微调。

参考：[Ollama Structured Outputs](https://docs.ollama.com/capabilities/structured-outputs)、[Modelfile 文档](https://docs.ollama.com/modelfile)。

## 四、综合分析

四种 API 的核心差别，是交互格式、状态管理、工具表达和生态兼容，而不是四套固定的模型能力。

模型“外挂”则把资料、计算和外部操作带入应用。以课程助手为例，可以由 RAG 查找讲义、计算工具核验数值、记忆系统保存学习进度，再通过适合项目的 API 调用模型生成带来源的答案；如果需要连接多个系统，可以由 MCP 宿主统一接入。

本文包含三项任务：模型下载、调用与对外服务的操作说明；四种 API 调用方式的区别；模型可接入的扩展能力。文中的命令和配置为实践步骤与示例，本次未进行真实模型下载、推理和公网部署，不将预期结果当作实测记录。技术特性依据所列官方资料整理，具体支持范围仍取决于服务版本和所选模型。

