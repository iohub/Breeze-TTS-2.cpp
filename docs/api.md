# breeze-server API 参考

`breeze-server` 是 Breeze-TTS-2.cpp 的本地推理服务：把文本合成语音（边生成边流式返回）、
管理参考音色、把一段录音换成另一个声音。本文是它的完整接口参考，覆盖 HTTP 与 WebSocket
两条通道、每个端点的参数与响应、音频格式约定、启动参数，以及这套接口**明确不提供**的能力。

它不是一个通用的 OpenAI 兼容网关：除 `POST /v1/audio/speech` 之外没有任何 OpenAI 端点
（没有 `/v1/models`、`/v1/files`、`/v1/audio/transcriptions`），也没有鉴权与限流。
需要细节时以源码为准，关键位置在每节末尾给出。

> 建议阅读顺序：先看[概览](#1-概览)与[路由索引](#2-路由索引)，再按需跳读具体接口。
> [注意事项与局限性](#6-注意事项与局限性)列出了几个最容易踩的坑：响应头不对称、
> 流式开始后无法再改状态码、`DELETE` 不删除磁盘文件。

## 目录

- [1. 概览](#1-概览)
  - [1.1 服务定位与构建](#11-服务定位与构建)
  - [1.2 监听地址与端口](#12-监听地址与端口)
  - [1.3 音频约定](#13-音频约定)
  - [1.4 并发、槽位与错误优先级](#14-并发槽位与错误优先级)
  - [1.5 安全前提](#15-安全前提)
- [2. 路由索引](#2-路由索引)
- [3. HTTP 接口](#3-http-接口)
  - [3.1 健康检查](#31-健康检查)
  - [3.2 显存管理](#32-显存管理)
  - [3.3 语音合成](#33-语音合成)
  - [3.4 音色转换](#34-音色转换)
  - [3.5 音色库](#35-音色库)
  - [3.6 静态资源与演示页](#36-静态资源与演示页)
  - [3.7 LLM 反向代理](#37-llm-反向代理)
  - [3.8 CORS 行为](#38-cors-行为)
- [4. WebSocket 协议](#4-websocket-协议)
- [5. 启动参数](#5-启动参数)
- [6. 注意事项与局限性](#6-注意事项与局限性)
- [7. 证据来源](#7-证据来源)

## 1. 概览

### 1.1 服务定位与构建

| 项目 | 值 |
| --- | --- |
| 可执行目标 | `breeze-server <model.gguf> [flags]` |
| 源码目录 | `apps/server/`（`main.cpp` 解析参数，`server.cpp` 跑服务） |
| 构建定义 | `CMakeLists.txt:289-300`，八个编译单元：`main.cpp`、`server.cpp`、`voices.cpp`、`ws.cpp`、`ws_api.cpp`、`openai.cpp`、`encode.cpp`、`llm_proxy.cpp` |
| 打开方式 | `-DBREEZE_BUILD_SERVER=ON`（默认就是 `ON`，`CMakeLists.txt:66`）；MP3 由 `-DBREEZE_MP3` 控制（默认 `ON`，`:67`），需要 `third_party/shine` 子模块 |
| HTTP 框架 | vendored **cpp-httplib 0.18.3**（`third_party/httplib.h:11`），服务主体为 `run_server`（`apps/server/server.cpp:159`），`svr.listen` 在 `apps/server/server.cpp:591` |
| WebSocket | cpp-httplib 不支持 WebSocket，因此是**独立端口**上的自研 RFC 6455 实现（`apps/server/ws.cpp:169`、`:184`、`:202`），进程内由 `httplib::Server` 线程池与 WS 接受线程各自服务 |

一个进程只加载一个模型，模型常驻显存（Q4_K 约 2.9 GB），按 `--parallel` 控制同时生成的流数量。

### 1.2 监听地址与端口

| 项 | 默认值 | 说明 |
| --- | --- | --- |
| HTTP host | `127.0.0.1` | `server.h:9`，`--host` 可改 |
| HTTP port | `8080` | `server.h:10`，`--port` 可改 |
| WebSocket port | HTTP port + 1 | `server.cpp:265`：`opts.ws_port == 0` 时取 `port + 1`；`--ws-port -1` 完全关闭 |
| 关闭 WS 的后果 | — | `/health` 的 `ws_port` 报 `0`，`--talk` 页面不可用（启动时会打印提示） |

HTTP 与 WS 都由同一个 `--host` 值绑定，但两者的处理方式不同：

- HTTP 走 `svr.listen(host, port)`，主机名会被 httplib 解析。
- WS 走 `inet_pton(AF_INET, host, ...)`，**只接受点分 IPv4 字面量**；传 `localhost` 或域名时
  `inet_pton` 失败，代码会退化成 `INADDR_ANY`（`ws.cpp:182`），也就是监听所有网卡。这是
  一个容易被忽略的安全差异，详见[注意事项](#6-注意事项与局限性)。

### 1.3 音频约定

| 项 | 值 |
| --- | --- |
| 声道 | 单声道（mono） |
| 采样率 | **24000 Hz**（取自 `model.cfg.sample_rate`；WS 的 `ready` 事件硬编码 `24000`） |
| 采样格式 | **s16le**，有符号 16 位小端，裸 PCM，**没有 WAV 头** |
| 帧率 | 12.5 帧/秒，即 1 帧 = 1920 个采样点 |
| 流式粒度 | 每次 flush 若干整帧，默认从 1 帧起步、逐步升到 25 帧（`--chunk-first` / `--chunk-max`） |

原生接口与 OpenAI 的 `pcm` 格式都直接返回这段裸 PCM。需要 WAV 就自己包头，或者请求
`response_format: "wav"`（由服务端加头）。MP3 路径会先上采样到 48 kHz 再编码，因此 MP3 响应的
`X-Sample-Rate` 报 `48000`（`encode.cpp:73-77`、`:87`）。

### 1.4 并发、槽位与错误优先级

生成、音色编码、音色转换都从同一个 `GenScheduler` 取槽位，槽位总数等于 `--parallel`：

| 场景 | 占用槽位 | 满了以后的 HTTP 行为 | 满了以后的 WS 行为 |
| --- | --- | --- | --- |
| 语音合成，`cfg_scale == 1.0` | 1 | `409 {"error":"busy"}` | 先发 `queued`，然后阻塞排队 |
| 语音合成，`cfg_scale != 1.0` 且 `--parallel > 1` | 2 | 同上 | 同上 |
| 语音合成，`cfg_scale != 1.0` 且 `--parallel == 1` | 1 | 同上 | 同上 |
| 音色转换 `POST /v1/audio/convert` | **全部** `--parallel` 个 | 同上 | — |
| `POST /v1/voices` 编码参考音频 | 1 | 同上 | — |

`GenScheduler::cost()` 只有在 `cfg_scale != 1.0 && parallel > 1` 时才返回 2
（`include/breeze/scheduler.h`），所以单槽部署里开引导不会额外占槽。

HTTP 是**立即拒绝**而不是排队，客户端应带退避重试；WebSocket 是排队，这也是对话类场景更适合
走 WS 的原因之一。HTTP 侧的错误判定顺序并不统一，写客户端时要注意：

- `POST /v1/audio/speech` 原生分支：**先取槽位**，再校验参数。所以槽位满时，即使 `text` 缺失
  也会先拿到 `409`。
- `POST /v1/audio/speech` OpenAI 分支：**先解析 JSON**（可返回 `400`/`404`），成功后才取槽位。
- `POST /v1/audio/convert`：**先取全部槽位**，再校验参数。
- 槽位不在时若模型已被卸载且装不回来，取槽位返回的是 `503` 而不是 `409`
  （`server.cpp:91-103` 的 `take_or_fail`）。

### 1.5 安全前提

**没有鉴权，没有速率限制，没有 Origin 校验**，HTTP 与 WebSocket 都是如此。`Authorization`
头只在 `/llm/` 透传时被转发，从不校验。这套接口只应绑在环回地址上，或者放到
终止 TLS 并做认证的反向代理之后。`docs/server.md:140` 对此有同样的声明。

## 2. 路由索引

共 20 条路由（含 CORS 通配 `OPTIONS`）。除 `/v1/audio/speech` 外都不是 OpenAI 端点。

| # | 方法 | 路径 | 说明 | 注册条件 | 源码 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `/health` | 存活检查、采样率、WS 端口、显存状态 | 始终 | `server.cpp:274` |
| 2 | POST | `/v1/unload` | 把模型移出显存，服务不停 | 始终 | `server.cpp:284` |
| 3 | POST | `/v1/load` | 提前把模型装回显存 | 始终 | `server.cpp:294` |
| 4 | POST | `/v1/audio/speech` | 语音合成，流式；同一路径兼收 OpenAI JSON | 始终 | `server.cpp:370` |
| 5 | POST | `/v1/audio/convert` | 音色转换，非流式 | 始终 | `server.cpp:462` |
| 6 | POST | `/v1/voices` | 注册/保存参考音色 | 始终 | `voices.cpp:187` |
| 7 | GET | `/v1/voices` | 列出已缓存与已保存的音色 | 始终 | `voices.cpp:251` |
| 8 | GET | `/v1/voices/(.+)` | 单个音色的详情 | 始终 | `voices.cpp:261` |
| 9 | PATCH | `/v1/voices/(.+)` | 改 `description` / `metadata` | 始终 | `voices.cpp:269` |
| 10 | DELETE | `/v1/voices/(.+)` | 从内存中移除音色 | 始终 | `voices.cpp:287` |
| 11 | GET | `/` | Web UI 的 `index.html` | `--webui` | `server.cpp:538` |
| 12 | GET | `/style.css` | Web UI 样式表 | `--webui` | `server.cpp:541` |
| 13 | GET | `/app.js` | Web UI 脚本 | `--webui` | `server.cpp:544` |
| 14 | GET | `/talk` | 语音通话演示页 | `--talk` | `server.cpp:551` |
| 15 | GET | `/talk.css` | 演示页样式表 | `--talk` | `server.cpp:554` |
| 16 | GET | `/talk.js` | 演示页脚本 | `--talk` | `server.cpp:557` |
| 17 | GET | `/talk/config` | 演示页需要的 `ws_port` / `tyger` / `sample_rate` | `--talk` | `server.cpp:560` |
| 18 | GET | `/llm/(.*)` | 透传到 `--llm` 指向的 OpenAI 兼容服务 | `--llm` 非空 | `llm_proxy.cpp:50` |
| 19 | POST | `/llm/(.*)` | 同上 | `--llm` 非空 | `llm_proxy.cpp:51` |
| 20 | OPTIONS | `.*` | CORS 预检，统一回 `204` | `--cors` 非空 | `server.cpp:580` |

两点源码层面的补充：

- 路由 11-19 是在 `run_server` 里按开关条件注册的，没开对应开关时这些路径落到 httplib
  的默认 `404`（`text/plain` 的 `Not Found`），不会返回 JSON 错误体。
- `--talk` 但 `--ws-port -1` 时，`/talk` 仍会注册，因为 WS 而无法工作；启动日志会给出提示
  （`server.cpp:566`）。

## 3. HTTP 接口

以下每节按「用途 → 请求 → 参数 → 响应 → 响应头 → 错误 → 示例」组织。所有 JSON 错误体统一为
`{"error":"<message>"}`，唯一例外是 OpenAI 兼容分支（见 [3.3.2](#332-openai-兼容分支)）。

### 3.1 健康检查

#### 用途

存活探针 + 状态查询。服务在加载完成前不会 `listen`，所以拿到 `200` 就说明模型已就绪；它同时
上报采样率、WS 端口、模型是否在显存、当前是否忙。

#### 请求

```http
GET /health
```

无参数、无请求体。`server.cpp:274`。

#### 响应

`200 application/json`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | 恒为 `"ok"` |
| `sample_rate` | int | 模型采样率，通常 `24000` |
| `ws_port` | int | WS 端口；关闭时为 `0` |
| `loaded` | bool | 模型是否在显存里（`false` 表示已被 unload / 尚未 lazy 加载） |
| `busy` | bool | 是否有任何槽位被占用（合成、转换、音色编码都算） |
| `idle_seconds` | int | 所有槽位空闲了多久，忙时恒为 `0` |

#### 示例

```bash
curl http://127.0.0.1:8080/health
```

```json
{"status":"ok","sample_rate":24000,"ws_port":8081,"loaded":true,"busy":false,"idle_seconds":42}
```

注意 `busy` 与 `idle_seconds` 来自调度器的槽位计数，所以「所有槽位空闲」才算 idle；
`--parallel 4` 时只有 4 个槽全空，`busy` 才会是 `false`。

> 已知局限：`ws_port` 只在**配置层**取值，即使 WS 端口实际绑定失败（日志里有
> `could not open the websocket port N`），`/health` 仍会报那个端口号。要确认 WS 可用，
> 必须真的去连一次。

### 3.2 显存管理

#### 用途

不重启进程就让模型离开显存，把 GPU 让给别的程序；以及提前把模型装回来。两者都可重复调用：
卸载已卸载的、加载已加载的，都只回报当前状态（`scheduler.cpp:71-85`）。

#### 请求

```http
POST /v1/unload
POST /v1/load
```

无参数、无请求体。注册位置 `server.cpp:284`（unload）与 `server.cpp:294`（load）。

#### 响应

| 端点 | 成功响应 | 说明 |
| --- | --- | --- |
| `POST /v1/unload` | `200 {"loaded":false}` | 权重、生成缓存、Vulkan 后端都释放；监听器、WS 会话、已缓存/已保存音色保留（音色以编码后的 codes 存在内存里） |
| `POST /v1/load` | `200 {"loaded":true}` | 装回显存并跑 warmup（除非 `--no-warmup`） |

#### 错误

| 状态 | 端点 | 响应体 | 触发条件 |
| --- | --- | --- | --- |
| `409` | `/v1/unload` | `{"error":"busy"}` | 还有槽位被占用。不会打断正在生成的请求，稍后重试即可 |
| `503` | `/v1/load` | `{"error":"the model could not be loaded"}` | 装载失败，通常是别的进程占着显存 |

`503` 之后，任何需要模型的请求都会继续返回 `503`，直到某次装载成功。错误体由
`server.cpp:91-103` 的 `take_or_fail` 与同风格的 `set_content` 生成。

#### 示例

```bash
curl -X POST http://127.0.0.1:8080/v1/unload
```

```json
{"loaded":false}
```

```bash
curl -X POST http://127.0.0.1:8080/v1/load
```

```json
{"loaded":true}
```

### 3.3 语音合成

`POST /v1/audio/speech` 是**同一路径承载两套协议**的端点，也是唯一 OpenAI 兼容的端点。

| 分支 | 判定条件 | 请求编码 | 响应形态 |
| --- | --- | --- | --- |
| 原生表单分支 | `Content-Type` 头**不包含**子串 `json`（含完全没有该头） | `multipart/form-data` 或 `application/x-www-form-urlencoded`，URL query 参数同样生效 | `audio/pcm` 裸流，chunked |
| OpenAI 兼容分支 | `Content-Type` 头包含子串 `json` | `application/json` | 由 `response_format` 决定 `audio/pcm` / `audio/mpeg` / `audio/wav`；`stream_format:"sse"` 时为 `text/event-stream` |

判定就是一次子串查找，大小写敏感，实现见 `server.cpp:371-374`：

```cpp
if (req.get_header_value("Content-Type").find("json") != std::string::npos) {
    speak_openai(req, res);
    return;
}
```

也就是说 `application/json; charset=utf-8`、`text/json` 都进 OpenAI 分支，而
`application/x-www-form-urlencoded`、`multipart/form-data` 与「无 Content-Type」都进原生分支。

#### 3.3.1 原生表单分支

##### 用途

声音设计（只给描述）、声音克隆（给参考音频）、以及「克隆 + 表演指导」三种模式，边生成边推流。

##### 参数

参数用 `field()` 读取（`voices.h:14`）：先查 multipart 文件部件，再查普通参数，**URL query
参数与表单字段等价**。所以小参数可以直接写进 query string。

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `text` | string | **必填** | 要读的文本，UTF-8。缺失或空 → `400` |
| `instruction` | string | `Speak clearly and naturally.` | 声音描述 / 表演指导 |
| `ref_text` | string | 空 | `ref_audio` 的逐字稿。**有 `ref_audio` 时必须给**：只有 `ref_text` 非空时参考音频才会被采纳（`generation.cpp:324-345`），否则这次请求实际按声音设计生成，而控制台仍会打印 `clone` |
| `ref_audio` | file | 无 | 参考 WAV。任意采样率与声道数，服务端重采样到单声道模型采样率。与 `ref_text` 一起参与哈希缓存，同一段音频 + 同一份文本复用已编码结果 |
| `voice_id` | string | 无 | 已缓存或已保存的音色 id，替代上传音频；未知 id → `404` |
| `cfg_scale` | float | `1.0` | 无分类器引导强度，`1.0` 关闭 |
| `seed` | int | `42` | 随机种子 |
| `temperature` | float | `0`（用模型默认） | 采样温度 |
| `top_k` | int | `0`（用模型默认） | |
| `top_p` | float | `0`（用模型默认） | |
| `repetition_penalty` | float | `0`（用模型默认） | |
| `max_new_tokens` | int | `0`（用模型默认，750 帧） | 每段文本的帧数上限，12.5 帧/秒 |
| `split_chars` | int | `--split-chars`（默认 600） | 长文本切分长度；`0` 表示整段一次生成 |

所有采样类字段把 `0` 解释为「用模型默认值」，不是真的设为 0。未知字段被忽略。

##### 响应

`200` + **流式** chunked 响应体：

| 响应头 | 值 |
| --- | --- |
| `Content-Type` | `audio/pcm` |
| `Transfer-Encoding` | `chunked` |
| `X-Sample-Rate` | `24000`（模型采样率） |
| `X-Sample-Format` | `s16le` |
| `Cache-Control` | `no-store` |

响应头设置于 `server.cpp:440-442`，chunked provider 在 `server.cpp:450-451`。响应体是
**无头的 s16le 单声道 PCM**，不是 WAV。首块通常在几百毫秒内到达（RTX 3060 上约 200–350 ms，
带参考编码时再加约 0.6 s），之后大致每个 chunk 一次 flush，默认 `--chunk-max 25` 即约 2 秒音频。

##### 错误

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `400` | `{"error":"could not read ref_audio"}` | 上传了 `ref_audio`，但 WAV 解析失败且没有可用的 codes（防止坏音频静默变成声音设计） |
| `400` | `{"error":"text is required"}` | `text` 缺失或为空 |
| `404` | `{"error":"unknown voice_id"}` | `voice_id` 不是已缓存/已保存的音色 |
| `409` | `{"error":"busy"}` | 所有槽位被占 |
| `503` | `{"error":"the model could not be loaded"}` | 模型不在显存且装不回来 |

判定顺序：槽位 → `ref_audio` → `voice_id` → `text`（见 `server.cpp:375-437`）。

##### 示例

声音设计：

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  --form-string "text=Welcome aboard. Your journey begins now." \
  --form-string "instruction=A warm, thoughtful young woman with a clear, calm delivery." \
  --form-string "cfg_scale=1" \
  --form-string "seed=42" \
  -o speech.pcm
```

用 `--form-string` 而不是 `-F` 传文本字段：`curl` 会把以 `(` 开头的值当作 multipart 分组的
开始，`-F "text=(sigh) ..."` 会发出空字段。

克隆（上传参考音频）：

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  --form-string "text=It is good to hear your voice again." \
  -F "ref_audio=@reference.wav" \
  --form-string "ref_text=This is the exact transcript of the reference audio." \
  -o clone.pcm
```

用已保存的音色，并把参数放 query string：

```bash
curl -X POST "http://127.0.0.1:8080/v1/audio/speech?voice_id=harbour&seed=7" \
  --form-string "text=The harbour was quiet that morning." \
  -o harbour.pcm
```

边播边收：

```bash
curl -sN -X POST http://127.0.0.1:8080/v1/audio/speech \
  --form-string "text=Hello there." \
  | ffplay -f s16le -ar 24000 -ac 1 -nodisp -autoexit -
```

#### 3.3.2 OpenAI 兼容分支

##### 用途

让本来就对接 OpenAI TTS 的应用（SillyTavern、官方 SDK）不加改动地指过来。解析入口是
`openai.cpp:87` 的 `parse_openai_speech`，由 `server.cpp:304` 的 `speak_openai` 调用。

##### 参数

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `input` | string | **必填** | 要读的文本。缺失或空 → `400 input is required` |
| `voice` | string 或 `{"id": ...}` | 无 | 已保存/已缓存音色的 id；也接受新式的对象形式。留空则用 `instructions` 做声音设计 |
| `instructions` | string | `Speak clearly and naturally.` | 声音描述（设计时）或表演指导（克隆时） |
| `response_format` | string | `mp3` | 只能是 `mp3`、`opus`、`aac`、`flac`、`wav`、`pcm` 之一（`openai.cpp:117-122`） |
| `stream_format` | string | `audio` | `audio` 直接返回文件流，`sse` 包成 Server-Sent Events（`openai.cpp:125-127`） |
| `model` | string | 忽略 | 接受但不使用，兼容总要发一个的客户端 |
| `speed` | number | 忽略 | 模型没有语速控制 |
| `cfg_scale` | number | `1.0` | 原生字段作为附加字段被接受 |
| `seed` | number | `42` | 同上 |
| `temperature` | number | `0` | 同上 |
| `top_k` | number | `0` | 同上 |
| `top_p` | number | `0` | 同上 |
| `repetition_penalty` | number | `0` | 同上 |
| `max_new_tokens` | number | `0` | 同上 |
| `split_chars` | number | `--split-chars` | 同上 |

数值字段既接受 JSON number，也接受「看起来像数字的字符串」（`num_field`，`openai.cpp:29`），
因为有些客户端会把数字加引号。`voice` 两侧的空白会被 trim，方便 SillyTavern 那种用逗号拼
音色列表的写法。

##### 响应

`Content-Type` 由 `make_encoder` 决定（`encode.cpp:172-194`）：

| `response_format` | 实际发送 | `Content-Type` | `X-Sample-Rate` |
| --- | --- | --- | --- |
| `pcm` | 裸 s16le PCM，24 kHz | `audio/pcm` | `24000` |
| `mp3` | MP3，80 kbps 单声道，48 kHz | `audio/mpeg` | `48000` |
| `wav` | 16 bit 单声道 WAV，24 kHz | `audio/wav` | `24000` |
| `opus` / `aac` / `flac` | **WAV** | `audio/wav` | `24000` |
| 任意格式，但 shine 缺失或 `-DBREEZE_MP3=OFF` | **WAV** | `audio/wav` | `24000` |

响应头只有两个（`server.cpp:329-330`）：

| 响应头 | 值 |
| --- | --- |
| `X-Sample-Rate` | 实际编码采样率（MP3 为 `48000`） |
| `Cache-Control` | `no-store` |

**这一分支不设置 `X-Sample-Format`**，与原生分支不对称，见
[注意事项](#6-注意事项与局限性)。所有格式都是边生成边发：WAV 头的长度字段写死为最大值，
播放器读到流结束；MP3 的编码器状态跨 chunk 保留。

##### SSE 帧

`stream_format: "sse"` 时 `Content-Type` 改为 `text/event-stream`（`server.cpp:331`），
每个 chunk 变成一条 `speech.audio.delta`，结束时补一条 `speech.audio.done`：

```text
data: {"type":"speech.audio.delta","audio":"//NIxAAAAAAAAAAAAFhpbmcAAAAPAAAA..."}

data: {"type":"speech.audio.delta","audio":"PAAAABAAAAB..."}

data: {"type":"speech.audio.done","usage":{"input_tokens":0,"output_tokens":0,"total_tokens":0}}
```

生成函数在 `openai.cpp:157`（delta，base64 编码）与 `openai.cpp:161`（done，token 计数恒为 0）。
把 delta 的 base64 依次解码拼接，得到的就是同一 `response_format` 下普通请求返回的完整文件；
选 `wav` 或 `mp3` 时，第一条 delta 里就带着容器头。

##### 错误

错误体是 OpenAI 形状（`openai.cpp:153`）：

```json
{"error":{"message":"input is required","type":"invalid_request_error","param":null,"code":null}}
```

| 状态 | `type` | 触发条件 |
| --- | --- | --- |
| `400` | `invalid_request_error` | 请求体不是 JSON 对象、`input` 缺失、某字段类型不对或 `response_format` 取值未知 |
| `404` | `invalid_request_error` | `voice` 不是已保存/已缓存音色 |
| `409` | `server_error` | 所有槽位被占，message 为 `busy, every generation slot is taken` |
| `503` | `server_error` | 模型装不回来，message 为 `the model could not be loaded` |

常见 `400` 消息：`request body must be a json object`、`input must be a string`、
`input is required`、`voice must be a string`、
`response_format must be one of mp3, opus, aac, flac, wav or pcm`、
`stream_format must be audio or sse`、`cfg_scale must be a number`（字段名会替换）。

##### 示例

要一个 WAV：

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"Hello from Breeze.","voice":"harbour","response_format":"wav"}' \
  -o speech.wav
```

声音设计（`voice` 留空）：

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"We need to discuss what happened last night.",
       "instructions":"Speak slowly with a restrained, serious tone.",
       "response_format":"pcm","cfg_scale":1.0,"seed":7}' \
  -o direction.pcm
```

走 SSE：

```bash
curl -N -X POST http://127.0.0.1:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input":"Streaming over server sent events.","response_format":"pcm","stream_format":"sse"}'
```

### 3.4 音色转换

#### 用途

把一段现成录音（`source`）保留其内容与节奏，换成另一个人的音色重新发声。**非流式**：整段生成
完才一次性返回。这个端点会**占满全部 `--parallel` 个槽位**（`server.cpp:464`），因为转换不参与
批处理，需要独占 GPU。

#### 请求

```http
POST /v1/audio/convert
Content-Type: multipart/form-data; boundary=...
```

必须是 multipart：`source` 与 `ref_audio` 都是文件部件。

#### 参数

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `source` | file | **必填** | 要转换的 WAV。任意采样率/声道，服务端重采样；读不出来 → `400` |
| `ref_audio` | file | 无 | 目标音色的参考 WAV；与 `voice_id` 二选一 |
| `voice_id` | string | 无 | 用已缓存/已保存的音色作参考，替代 `ref_audio`；未知 → `404` |
| `ref_text` | string | 空 | 参考音频的逐字稿。给 `voice_id` 时可省略（用音色里存的那份）；否则必填 |
| `text` | string | 空 | `source` 的逐字稿。留空则由服务端自己编码来源音频 |
| `temperature` | float | `0.3` | 采样温度 |
| `top_k` | int | `1` | 采样 top-k |
| `cfg_scale` | float | `1.0` | 引导强度 |
| `keep_acoustic` | int | `0` | 保留原始声学细节的开关，`0` 关闭 |
| `seed` | int | `42` | 随机种子 |

注意默认值与语音合成不同：`temperature` 是 `0.3`、`top_k` 是 `1`，转换的确定性更强。

#### 响应

一次性 `200`，`Content-Type: audio/pcm`（`server.cpp:529`），body 是完整的裸 s16le 单声道 PCM：

| 响应头 | 值 |
| --- | --- |
| `Content-Type` | `audio/pcm` |
| `X-Sample-Rate` | `24000` |
| `X-Sample-Format` | `s16le` |

这里**不设** `Cache-Control`（与语音合成不同），也没有 `Transfer-Encoding: chunked`。

#### 错误

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `400` | `{"error":"a source wav file is required"}` | `source` 缺失或 WAV 解析失败 |
| `400` | `{"error":"ref_audio or voice_id is required"}` | 两者都没给 |
| `400` | `{"error":"ref_text is required"}` | 既没有可用 `voice_id` 的文本，也没显式给 `ref_text` |
| `404` | `{"error":"unknown voice_id"}` | `voice_id` 未知 |
| `409` | `{"error":"busy"}` | 槽位没全空（这个接口要全部槽位） |
| `503` | `{"error":"the model could not be loaded"}` | 模型装不回来 |
| `500` | `{"error":"conversion failed"}` | 生成过程中抛异常，详情只打在 stderr |

#### 示例

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/convert \
  -F "source=@take.wav" \
  -F "ref_audio=@reference.wav" \
  --form-string "ref_text=This is the exact transcript of the reference audio." \
  --form-string "text=The line I actually said in the recording." \
  --form-string "temperature=0.3" \
  --form-string "top_k=1" \
  -o converted.pcm
```

用已保存音色，省掉一次参考编码：

```bash
curl -X POST http://127.0.0.1:8080/v1/audio/convert \
  -F "source=@take.wav" \
  --form-string "voice_id=harbour" \
  -o converted.pcm
```

### 3.5 音色库

#### voice 对象

五个 `/v1/voices` 路由共享同一个序列化结构（`voices.cpp:45-59`）：

```json
{
  "id": "harbour",
  "frames": 50,
  "seconds": 4.00,
  "encode_ms": 641,
  "saved": true,
  "ref_text": "This is the exact transcript of the reference audio.",
  "description": "Warm, low, unhurried.",
  "metadata": {"language": "en", "role": "narrator"}
}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 音色 id。命名保存的就是文件名；未命名的是 `v_<16 位十六进制>`，由音频字节与逐字稿哈希得到 |
| `frames` | int | 编码后的帧数，12.5 帧/秒 |
| `seconds` | number | 时长，`frames × 每帧采样数 / 采样率`，保留两位 |
| `encode_ms` | number | 编码耗时（毫秒）；从磁盘加载的音色是 `0` |
| `saved` | bool | 背后是否有 `.breeze` 文件（重启后仍在） |
| `ref_text` | string | 参考音频的逐字稿 |
| `description` | string | 自由文本描述，上限 4096 字节 |
| `metadata` | object | 键值**全为字符串**的扁平对象；上限 32 个字段，值 ≤1024 字节 |

音色表上限 64 条（`voices.h:54`）。满了以后淘汰最旧的**未命名**条目；已保存的音色永不淘汰
（`voices.cpp:152-166`）。

#### `POST /v1/voices`

##### 用途

编码一段参考音频并登记为音色。给了 `name` 就同时写到 `--voices-dir` 下的 `<name>.breeze`，
重启后仍在；不给就是纯内存缓存，随进程消失。

##### 参数

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `ref_audio` | file | **必填** | 参考 WAV，任意采样率/声道 |
| `ref_text` | string | **必填** | 参考音频的逐字稿 |
| `name` | string | 无 | 存盘文件名，只允许字母、数字、`_`、`-`，≤64 字符 |
| `description` | string | 空 | 描述，≤4096 字节 |
| `metadata` | string | 空 | **JSON 对象字符串**，如 `{"language":"en"}` |

`description` 与 `metadata` 既可以作为表单字段，也可以放在 JSON 请求体里
（`read_meta_request`，`voices.cpp:68-89`）。但 `POST` 始终需要 multipart 的 `ref_audio`，
所以纯 JSON body 的 `POST` 会先撞上 `400 ref_audio and ref_text are required`；只想改元数据
请用 `PATCH`。

##### 响应

`200 application/json`，返回 voice 对象（`voices.cpp:248`）。

##### 错误

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `400` | `{"error":"ref_audio and ref_text are required"}` | 缺 `ref_audio` 文件部件，或 `ref_text` 为空 |
| `400` | `{"error":"name can only use letters, digits, dash and underscore"}` | `name` 非法 |
| `400` | `{"error":"request body must be a json object"}` | JSON body 不是对象 |
| `400` | `{"error":"metadata must be a json object"}` | `metadata` 字符串不是 JSON 对象 |
| `400` | `{"error":"could not read ref_audio"}` | WAV 解析后没有任何采样点 |
| `400` | `{"error":"description is longer than 4096 bytes"}` 等 | 元数据校验失败（见下） |
| `409` | `{"error":"busy"}` | 编码需要 1 个槽位，全被占 |
| `503` | `{"error":"the model could not be loaded"}` | 模型装不回来 |
| `500` | `{"error":"could not write the voice file"}` | `.breeze` 落盘失败 |

元数据校验规则（`apply_meta`，`voices.cpp:91-119` 与 `src/voice.cpp:113-131`）：

- key：1–64 字节，只允许字母、数字、`.`、`-`、`_`，且不能叫 `description`。
- value：必须是字符串，≤1024 字节；传 `null` 或空串表示**删除**该键。
- `description` 传 `null` 表示清空。

##### 重复提交的行为

同名/同哈希再次提交时不会重复编码，或者会覆盖文件：

| 情况 | 行为 |
| --- | --- |
| 未命名（`name` 省略）且哈希已存在 | 只把新的 `description`/`metadata` 合并进去，直接返回已有条目，**不重新编码** |
| 命名且已存在 | 沿用旧 `description`/`metadata`（除非本次显式提供），重新编码并重写 `.breeze` |

##### 示例

```bash
curl -X POST http://127.0.0.1:8080/v1/voices \
  -F "ref_audio=@reference.wav" \
  --form-string "ref_text=This is the exact transcript of the reference audio." \
  --form-string "name=harbour" \
  --form-string "description=Warm, low, unhurried." \
  --form-string 'metadata={"language":"en","role":"narrator"}'
```

只登记到内存（拿返回的 `id` 去合成）：

```bash
curl -X POST http://127.0.0.1:8080/v1/voices \
  -F "ref_audio=@reference.wav" \
  --form-string "ref_text=This is the exact transcript of the reference audio."
```

#### `GET /v1/voices`

返回 **JSON 数组**，顺序为「启动时从磁盘加载的音色」在前，之后按注册先后
（`voices.cpp:251-259`）：

```bash
curl http://127.0.0.1:8080/v1/voices
```

```json
[{"id":"harbour","frames":50,"seconds":4.00,"encode_ms":0,"saved":true,
  "ref_text":"...","description":"Warm, low, unhurried.","metadata":{"language":"en"}}]
```

无错误码，空表返回 `[]`。

#### `GET /v1/voices/{id}`

返回单个 voice 对象（`voices.cpp:261-267`）。

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `200` | voice 对象 | — |
| `404` | `{"error":"unknown voice_id"}` | id 不在表里 |

```bash
curl http://127.0.0.1:8080/v1/voices/harbour
```

#### `PATCH /v1/voices/{id}`

改 `description` 与 `metadata`，支持 JSON body 或表单字段（`voices.cpp:269-285`）。语义是
**合并**：只更新给出的键，`null` 或空串删除该键。

```bash
curl -X PATCH http://127.0.0.1:8080/v1/voices/harbour \
  -H "Content-Type: application/json" \
  -d '{"description":"Warm and dry.","metadata":{"language":"en","role":null,"accent":"rp"}}'
```

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `200` | 更新后的 voice 对象 | — |
| `400` | `{"error":"..."}` | body 不是 JSON 对象、`description` 不是字符串、元数据超限或 key 非法 |
| `404` | `{"error":"unknown voice_id"}` | id 不在表里 |
| `500` | `{"error":"could not write the voice file"}` | 该音色已保存到磁盘，但重写 `.breeze` 失败 |

已保存的音色改完会立刻重写磁盘文件，所以修改是持久的；纯内存音色只改内存。

#### `DELETE /v1/voices/{id}`

从内存中移除音色（`voices.cpp:287-299`）。

```bash
curl -X DELETE http://127.0.0.1:8080/v1/voices/harbour
```

```json
{"deleted":"harbour","file_kept":true}
```

| 字段 | 说明 |
| --- | --- |
| `deleted` | 被移除的 id |
| `file_kept` | `true` 表示磁盘上的 `<id>.breeze` **仍然存在**（该音色原本是 saved）；`false` 表示它本来只在内存里 |

**磁盘文件不会被删除。** 源码注释写得很直接（`voices.cpp:292`：*the file stays put, deleting
one over http would be a nasty way to lose a voice*）。想彻底删掉一个保存过的音色，得自己
删 `--voices-dir` 下的 `.breeze` 文件；否则下次启动它会被重新加载回来。

| 状态 | 响应体 | 触发条件 |
| --- | --- | --- |
| `404` | `{"error":"unknown voice_id"}` | id 不在表里 |

### 3.6 静态资源与演示页

两套页面都是**编译期内嵌**的：`cmake/embed_webui.cmake` 把 `webui/` 下的文件转成字节数组，
生成 `breeze_webui_assets.h`，所以没有可部署的静态目录，改完前端要重新构建才生效。

#### `--webui`

| 路由 | 内容 | `Content-Type` | 源码 |
| --- | --- | --- | --- |
| `GET /` | `webui/index.html` | `text/html` | `server.cpp:538` |
| `GET /style.css` | `webui/style.css` | `text/css` | `server.cpp:541` |
| `GET /app.js` | `webui/app.js` | `application/javascript` | `server.cpp:544` |

页面与 API 同源，不需要 CORS；它调用的就是 `/v1/audio/speech`。

#### `--talk`

| 路由 | 内容 | `Content-Type` | 源码 |
| --- | --- | --- | --- |
| `GET /talk` | `webui/talk.html` | `text/html` | `server.cpp:551` |
| `GET /talk.css` | `webui/talk.css` | `text/css` | `server.cpp:554` |
| `GET /talk.js` | `webui/talk.js` | `application/javascript` | `server.cpp:557` |
| `GET /talk/config` | 页面需要的运行时配置 | `application/json` | `server.cpp:560` |

`GET /talk/config` 的响应：

```json
{"ws_port":8081,"tyger":"ws://127.0.0.1:8765/ws","sample_rate":24000}
```

| 字段 | 说明 |
| --- | --- |
| `ws_port` | 这里返回的是**原始配置值**，`--ws-port -1` 时会返回 `-1`（而 `/health` 报 `0`） |
| `tyger` | `--tyger` 的值，JSON 转义后原样返回 |
| `sample_rate` | 模型采样率 |

这个页面还需要 `--llm` 指向的聊天服务与外部 tyger-server 才能完整工作；只开 `--talk` 而没有
WS 时，启动会打印 `the voice call demo needs the websocket, drop --ws-port -1`。

### 3.7 LLM 反向代理

#### 用途

`--llm <url>` 非空时（`--talk` 会自动把它设为 `http://127.0.0.1:1234/v1`），服务把
`/llm/<path>` 透传到 `<url>/<path>`，让浏览器不必自己处理本地 LLM 服务的 CORS。

#### 请求

```http
GET  /llm/(.*)
POST /llm/(.*)
```

注册位置 `llm_proxy.cpp:50-51`。目标地址在启动时被拆成 origin 与路径前缀（`split_url`，
`llm_proxy.cpp:8`），例如 `http://127.0.0.1:1234/v1` → origin `http://127.0.0.1:1234`、
prefix `/v1`。

#### 行为

| 项 | 行为 |
| --- | --- |
| 方法 | 仅 `GET` 与 `POST`；`PATCH`/`DELETE` 等落到 httplib 默认 `404` |
| 转发头 | **只转发 `Authorization`**；无 `Content-Type` 时补 `application/json` |
| 未转发 | query string、其他请求头、cookies 全部丢弃（路径只取 `matches[1]`） |
| 响应 | 一律 `Content-Type: text/event-stream`，chunked 逐块转发上游内容 |
| 超时 | 连接 5 s，读 300 s（`llm_proxy.cpp:28-29`） |
| 上游不可达 | 返回 `200`，body 为 `{"error":{"message":"could not reach the llm at <origin>"}}` |

鉴权是**透传**，不是校验：客户端发什么 `Authorization` 就往上游发什么，服务端自己不检查。

#### 示例

```bash
curl http://127.0.0.1:8080/llm/models
```

```bash
curl -X POST http://127.0.0.1:8080/llm/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer local" \
  -d '{"model":"any","messages":[{"role":"user","content":"hi"}],"stream":true}'
```

### 3.8 CORS 行为

`--cors` 非空时启用（`server.cpp:570`），由两部分组成：

1. **post-routing 统一加头**（`server.cpp:572-578`），对每个响应生效，包括错误响应与 chunked
   流：

   | 响应头 | 值 |
   | --- | --- |
   | `Access-Control-Allow-Origin` | `*`（`--cors` 无参数或传 `*`），否则回显匹配的 `Origin` |
   | `Access-Control-Expose-Headers` | `X-Sample-Rate, X-Sample-Format` |
   | `Vary` | `Origin`，仅在非 `*` 模式下添加 |

2. **`OPTIONS .*` 通配预检**（`server.cpp:580-588`）：一律 `204`，带

   | 响应头 | 值 |
   | --- | --- |
   | `Access-Control-Allow-Methods` | `GET, POST, PATCH, DELETE, OPTIONS` |
   | `Access-Control-Max-Age` | `86400` |
   | `Access-Control-Allow-Headers` | 回显请求的 `Access-Control-Request-Headers`（有才回） |

不在白名单里的 origin 依然拿到 `204`，只是没有 `Access-Control-Allow-Origin`，浏览器会自行
拒绝。origin 比较是**逐字节相等**，scheme 与端口都要对上。

启动参数两种写法（`main.cpp:73-78`）：单独 `--cors` 允许任意 origin；跟一个以 `-` 开头的参数
时视为没有给出值（因为 origin 一定以 scheme 开头）：

```bash
breeze-server model.gguf --cors
breeze-server model.gguf --cors http://localhost:5173,https://app.example.com
```

CORS 只作用于 HTTP。WebSocket 不受浏览器同源策略约束，这个开关对 WS 端口没有任何影响。

## 4. WebSocket 协议

### 4.1 为什么是另一个端口

cpp-httplib 不支持 WebSocket，也不把升级后的 socket 交出来，所以这里是**直接实现在独立监听器
上的 RFC 6455**（`ws.cpp:169` 的 `WsServer::start`，`ws.cpp:184` 的 `bind/listen`，
`ws.cpp:202` 的握手）。它不是 HTTP 路由，`/ws` 之类的路径并不存在——连上 TCP 端口就是 WS。

| 项 | 值 |
| --- | --- |
| 默认端口 | HTTP 端口 + 1（`server.cpp:265`） |
| 改端口 | `--ws-port N` |
| 关闭 | `--ws-port -1` |
| 绑定地址 | 同 `--host`，但只认点分 IPv4；其他写法退化为 `INADDR_ANY` |
| 端口发现 | `GET /health` 的 `ws_port`，或 `--talk` 页面的 `GET /talk/config` |
| 握手 | 读到空行为止，只要求请求里有 `Sec-WebSocket-Key`（不校验 `Origin`/`Upgrade`/`Sec-WebSocket-Version`），回 `101` + `Sec-WebSocket-Accept` |
| 子协议 | 不支持协商，忽略 `Sec-WebSocket-Protocol` |
| Origin | **不校验** |
| 单条消息上限 | 8 MB（`ws.cpp:26`），超出直接断连 |
| 控制帧 | 处理 `ping`/`pong`/`close`；收到 `close` 即结束会话 |

### 4.2 帧约定

| 帧类型 | 内容 |
| --- | --- |
| **文本帧** | JSON 控制消息，必须是一个 JSON 对象 |
| **二进制帧** | 裸 PCM 音频：s16le 单声道 **24 kHz**，与 HTTP 响应体完全一致 |

WebSocket 在帧层面已经区分文本与二进制，所以音频不需要信封、不需要 base64。客户端发来的二进制
帧会被忽略（`ws_api.cpp:209`）。

### 4.3 客户端 → 服务端

| `type` | 字段 | 说明 | 源码 |
| --- | --- | --- | --- |
| `start` | `voice_id`、`instruction`、`ref_text`、`cfg_scale`、`seed`、`temperature`、`top_k`、`split_chars` | 开一个会话。参考音色只编码一次，全会话复用。会清空队列与缓冲 | `ws_api.cpp:166`、`:217` |
| `text` | `text` | 追加文本。凑成完整句子的部分立刻开始说，剩下的留在缓冲里 | `ws_api.cpp:230` |
| `flush` | `text` | 把缓冲加上这段文本作为一段立刻说出，不管有没有句末标点 | `ws_api.cpp:230` |
| `instruction` | `instruction` | 改变此后入队的每一段的表演方式 | `ws_api.cpp:225` |
| `cancel` | — | 中断当前段并丢弃缓冲 | `ws_api.cpp:242` |
| `end` | `text` | 追加最后的文本，说完所有内容后发 `done` | `ws_api.cpp:230` |

`start` 的字段默认值与 HTTP 原生分支一致（`cfg_scale` 1.0、`seed` 42、`temperature` 0、
`top_k` 0、`instruction` 为 `Speak clearly and naturally.`）。两点差异必须注意：

1. **WS 的 `start` 不支持 `top_p`、`repetition_penalty`、`max_new_tokens`**，也不支持直接上传
   `ref_audio` —— 参考音频要先 `POST /v1/voices` 换一个 `voice_id`。
2. `split_chars` 在 WS 上**不能关**：`0` 或负数会被改成 600（`ws_api.cpp:189-191`），因为按句
   流式输出需要一个长度上限。

### 4.4 服务端 → 客户端

所有事件的 JSON 由 `event()` 拼装（`ws_api.cpp:103`），格式为 `{"type":"...", ...}`。

| `type` | 附带字段 | 时机 | 源码 |
| --- | --- | --- | --- |
| `ready` | `sample_rate`、`format` | 连接建立后立刻发，值为 `24000` / `"s16le"` | `ws_api.cpp:204` |
| `started` | `voice_id` | `start` 处理完，会话打开 | `ws_api.cpp:197` |
| `speaking` | `text` | 某一段即将开始生成 | `ws_api.cpp:144` |
| `queued` | — | 槽位已满，正在排队（不是拒绝） | `ws_api.cpp:131` |
| `instruction_set` | — | 新的表演方式从此后生效 | `ws_api.cpp:229` |
| `cancelled` | — | 当前段被打断 | `ws_api.cpp:155` |
| `done` | — | 队列里所有内容都说完 | `ws_api.cpp:120`、`:161` |
| `error` | `message` | 请求有问题，或模型装不回来 | 见下 |

`error` 的 `message` 取值：

| `message` | 触发条件 |
| --- | --- |
| `messages have to be a json object` | 文本帧不是合法 JSON 对象 |
| `send start first` | 还没发 `start` 就发了别的消息 |
| `unknown voice_id` | `start` 里的 `voice_id` 不存在 |
| `unknown type` | `type` 不认识 |
| `the model could not be loaded` | 轮到这一段时模型装不回来，这一段被跳过 |

### 4.5 最小交互示例

```text
-> {"type":"start","voice_id":"harbour","seed":7}
<- {"type":"started","voice_id":"harbour"}
-> {"type":"text","text":"The harbour was quiet that morning. "}
<- {"type":"speaking","text":"The harbour was quiet that morning."}
<- <binary audio>
-> {"type":"instruction","instruction":"Speak in an urgent, alarmed whisper."}
<- {"type":"instruction_set"}
-> {"type":"end","text":"Then the alarm went off."}
<- {"type":"speaking","text":"Then the alarm went off."}
<- <binary audio>
<- {"type":"done"}
```

对应的最小 Python 客户端：

```python
import asyncio, json, websockets

async def speak():
    async with websockets.connect("ws://127.0.0.1:8081", max_size=None) as ws:
        print(await ws.recv())                     # ready
        await ws.send(json.dumps({"type": "start", "voice_id": "harbour"}))
        await ws.send(json.dumps({"type": "end", "text": "Hello there."}))
        audio = bytearray()
        async for msg in ws:
            if isinstance(msg, bytes):
                audio.extend(msg)
            elif json.loads(msg)["type"] in ("done", "error"):
                break
        return bytes(audio)                        # s16le, 24 kHz, mono

asyncio.run(speak())
```

### 4.6 会话语义与并发

- **输入缓冲**：文本按句末标点切分（`.`、`!`、`?`、`;` 以及 `。！？；`）；若缓冲超过
  `split_chars` 仍没有句末标点，就退化为按空格切。`flush` 与 `end` 强制吐出全部缓冲。
- **锚点限制**：会话还没有参考音色时，第一段会充当后面所有内容的参考，所以它的预算被压到 200
  字符（`ws_api.cpp:236`），太长的首段会让模型后面开始跳句。
- **队列与槽位**：每段单独取槽位，槽位满时先发 `queued` 再阻塞等待，因此多条连接会在段与段之间
  交错，而不是一条连接霸占整段对话。
- **`cancel` 的边界**：只有确实正在说话时才置位取消标志；空闲时发 `cancel` 只清空队列，不会
  吞掉下一段。
- **断连**：socket 断开后会话退出、置取消标志，正在生成的段随之中止。

WS 同样**没有鉴权、没有 Origin 校验、没有速率限制**。绑到 `0.0.0.0` 等于把 GPU 交给整个网络，
务必放在反向代理之后（`docs/websocket.md` 的 Security 一节也是这么写的）。

## 5. 启动参数

```text
breeze-server <model.gguf> [--host H] [--port P] [--webui] [--cpu] [--te-cpu]
                           [--chunk-first N] [--chunk-max N] [--verbose]
                           [--voices-dir PATH] [--ws-port P] [--split-chars N]
                           [--cors [ORIGINS]] [--no-warmup] [--parallel N]
                           [--talk] [--llm URL] [--tyger URL]
                           [--idle-unload SECONDS] [--lazy] [--before-load CMD]
```

帮助文本在 `main.cpp:11-44`，解析在 `main.cpp:52-78`。

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `--host` | `127.0.0.1` | 绑定地址。用 `0.0.0.0` 才接受远程连接（注意 WS 的 IPv4-literal 限制） |
| `--port` | `8080` | HTTP 端口 |
| `--ws-port` | HTTP 端口 + 1 | WS 端口；`-1` 关闭 |
| `--webui` | 关 | 在 `/` 提供浏览器 UI |
| `--cors` | 关 | 允许跨源调用；无值或 `*` 表示任意 origin，否则逗号分隔白名单 |
| `--parallel` | `1` | 同时生成的流数量；必须 ≥1。引导流在 `>1` 时算两个槽 |
| `--chunk-first` | `1` | 首个流式块的最小帧数；GPU 忙时服务端会自动抬高 |
| `--chunk-max` | `25` | 块大小递增到的上限；两者设成一样就是不递增的固定块 |
| `--split-chars` | `600` | 长文本切分长度；`0` 表示整段一次生成（请求可覆盖） |
| `--voices-dir` | `voices` | 启动时加载 `.breeze` 的目录，也是保存音色的目录 |
| `--cpu` | 关 | 强制 CPU 后端（不使用 Vulkan） |
| `--te-cpu` | 关 | 文本编码器跑在 CPU，省约 1 GB 显存，代价是每段文本多约 0.7 s；不要与 `--parallel` 同用 |
| `--no-warmup` | 关 | 跳过启动时的预热生成 |
| `--lazy` | 关 | 启动时就把模型留在显存外，首个请求再加载 |
| `--idle-unload` | `0` | 空闲这么多秒后自动卸载模型；`0` 表示永不，必须 ≥0 |
| `--before-load` | 无 | 每次加载前执行的 shell 命令（含启动那次）。命令失败只记录日志，不阻止加载 |
| `--talk` | 关 | 提供 `/talk` 语音通话演示页 |
| `--llm` | 空；`--talk` 时为 `http://127.0.0.1:1234/v1` | 演示页使用的 OpenAI 兼容服务，经 `/llm/` 透传 |
| `--tyger` | `ws://127.0.0.1:8765/ws` | 演示页使用的 tyger-server 地址 |
| `--verbose` / `-v` | 关 | 每个请求后附加分阶段耗时 |

参数校验（`main.cpp:79-93`）：`--parallel < 1`、`--idle-unload < 0`、
`--chunk-first/--chunk-max < 1` 都会直接报错退出（退出码 `1`），未知参数同样退出码 `1`。
`--split-chars` 为负会被静默改成 `0`。

`-h` / `--help`（或不给任何参数）打印上面的 usage 文本，但只在作为 `argv[1]` 时才生效
（`main.cpp:10`）；写在模型路径后面会被当成未知参数并以退出码 `1` 结束。

```bash
breeze-server breeze-tts-2-q4_k.gguf --port 8137 --webui
```

## 6. 注意事项与局限性

### 6.1 不提供的接口

本服务只做 TTS 及其配套能力。以下端点**不存在**，访问会得到 httplib 的默认 `404`（`text/plain`
的 `Not Found`，不是 JSON）：

| 缺失的端点 | 说明 |
| --- | --- |
| `GET /v1/models` | 全仓 grep `v1/models` 命中 0；OpenAI 官方 SDK 里凡是先 `models.list()` 的流程都会失败 |
| `POST /v1/audio/transcriptions` | 没有 ASR。需要转写请用 tyger-server 之类的独立服务 |
| `POST /v1/files`、`GET /v1/files` | 没有文件对象；参考音频走 `multipart` 或 `/v1/voices` |
| `GET/POST /v1/audio/voices` | 音色用 `/v1/voices`，不是 OpenAI 的形状 |
| `GET /status`、`GET /metrics`、`GET /openapi.json` | 只有 `/health`；没有 Prometheus 指标，也没有 OpenAPI 描述 |

OpenAI 的内置音色名（`alloy`、`nova` 等）也不存在，用它们会得到 `404`。想兼容就把同名音色用
`POST /v1/voices` 存成 `alloy`，之后就能当普通 `voice_id` 用。

### 6.2 响应头不对称

| 端点 | `X-Sample-Rate` | `X-Sample-Format` | `Cache-Control` |
| --- | --- | --- | --- |
| `POST /v1/audio/speech`（原生） | 设 | 设 | 设 |
| `POST /v1/audio/speech`（OpenAI） | 设 | **不设** | 设 |
| `POST /v1/audio/convert` | 设 | 设 | **不设** |

写通用客户端时不要假设 `X-Sample-Format` 一定在：OpenAI 分支只有 `X-Sample-Rate`
（`server.cpp:329` 与 `server.cpp:440-441` 的差异）。从 OpenAI 分支拿到的裸 PCM 仍是
s16le，只是头里没说。

`Access-Control-Expose-Headers` 声明的是 `X-Sample-Rate, X-Sample-Format`，跨源时想读别的
自定义头需要自己加代理。

### 6.3 流式一旦开始就无法回退

chunk 一旦发出，HTTP 状态码就定死了。若生成中途失败，服务端只能提前断开连接，客户端会看到
**被截断的响应体**（OpenAI 分支的 provider 在 `server.cpp:340-368`，原生分支在
`server.cpp:450-459`，任何一层失败后都直接 `sink.done()`）。
因此：把不完整的 body 当作错误处理，不要只看状态码。

同理，OpenAI 的 SSE 分支在失败时也可能只是少一条 `speech.audio.done`，客户端应当给它设超时。

### 6.4 `DELETE /v1/voices/{id}` 不删磁盘文件

见 [3.5](#35-音色库)。`file_kept: true` 就是「文件还在」的意思，重启后音色会回来。要真删就删
`--voices-dir` 下对应的 `.breeze`。

### 6.5 WS 的绑定地址陷阱

`--host` 同时用于 HTTP 与 WS，但 WS 侧只接受点分 IPv4（`inet_pton`）。写 `--host localhost`、
`--host myhost.lan` 时 HTTP 正常，WS 却会退化成 `INADDR_ANY`——**监听所有网卡**，而且不会有任何
警告。只想本机访问就坚持用 `127.0.0.1`。

### 6.6 状态字段的两处不一致

| 场景 | `/health` 的 `ws_port` | `/talk/config` 的 `ws_port` |
| --- | --- | --- |
| 正常 | 端口号 | 端口号 |
| `--ws-port -1` | `0` | `-1` |

另外 `/health` 报的是**配置值**：WS 端口绑定失败时它依然会报那个端口。真正要确认可达，得连一次。

### 6.7 LLM 代理不是通用反代

`/llm/` 只转发 `GET`/`POST`、只转发 `Authorization`、**丢弃 query string**、并且无条件把响应
标成 `text/event-stream`。上游返回普通 JSON 也能用（内容照样逐块转发），但客户端不能依赖
`Content-Type` 判断格式；带 `?api-version=...` 之类查询参数的接口会失败。

### 6.8 其它容易踩的点

- **`POST /v1/audio/convert` 要独占全部槽位**，`--parallel 4` 时它也要等 4 个槽全空，期间任何
  合成请求都会 `409`。
- **原生合成分支先取槽位再校验参数**，所以并发压力大时参数错误也会先看到 `409`。
- **`POST /v1/voices` 的 JSON body 只能携带元数据**，`ref_audio` 必须是 multipart 文件部件；
  纯 JSON 会得到 `400`。
- **WAV 头长度字段写死为 `0xFFFFFFFF`**（`encode.cpp:42-60`），是流式 WAV 的常见做法：播放器
  读到流结束为止。要精确长度的文件请自己封装。
- **`--cors` 的 origin 是逐字节比较**，`http://localhost:5173` 与
  `http://127.0.0.1:5173` 是两个不同的 origin。
- **没有请求体大小上限检查**（httplib 的 `PAYLOAD_MAX_LENGTH` 保持默认的极大值），上传大 WAV
  时服务端会整段收进内存。

## 7. 证据来源

本文所有接口细节均来自对 `apps/server/` 的**静态源码分析**（未启动服务实测），端口、路由、
参数默认值、错误消息与响应头都逐条对照源码确认。关键文件：

| 文件 | 覆盖内容 |
| --- | --- |
| `apps/server/main.cpp` | 启动参数与校验、usage 文本（`:11-44`、`:52-93`） |
| `apps/server/server.h` | `ServerOptions` 默认值（host `:9`、port `:10`、ws_port `:16`） |
| `apps/server/server.cpp` | 全部核心 HTTP 路由（`:274`、`:284`、`:294`、`:370`、`:462`、静态页 `:538-565`）、CORS（`:570-589`）、`take_or_fail`（`:91`）、WS 端口推导（`:265`）、`listen`（`:591`） |
| `apps/server/voices.cpp` / `voices.h` | `/v1/voices` 五条路由（`:187`、`:251`、`:261`、`:269`、`:287`）、voice 对象结构（`:45`）、`field()`（`voices.h:14`） |
| `apps/server/openai.cpp` | OpenAI 请求解析（`:87`）、SSE 帧（`:157`、`:161`）、错误体（`:153`） |
| `apps/server/encode.cpp` | `response_format` 到 `Content-Type` 的映射（`:172`）、MP3 上采样（`:73-77`） |
| `apps/server/llm_proxy.cpp` | `/llm/` 透传、超时与转发的头（`:23-48`） |
| `apps/server/ws.cpp` / `ws_api.cpp` | RFC 6455 帧层与握手（`ws.cpp:169`、`:202`）、消息与事件（`ws_api.cpp:103`、`:166`、`:204`、`:217-251`） |
| `include/breeze/scheduler.h`、`src/scheduler.cpp` | 槽位与 `cost()` 语义、`load`/`unload`/`idle_seconds` |
| `CMakeLists.txt:289-314` | `breeze-server` 的编译单元、线程与 shine 依赖 |
| `third_party/httplib.h:11` | cpp-httplib 版本 `0.18.3` |

配套文档：[server.md](server.md)（面向使用的服务说明）、[websocket.md](websocket.md)
（WS 会话设计）、[voices.md](voices.md)（音色文件与缓存策略）、
[voice-conversion.md](voice-conversion.md)（音色转换原理）。本文只做接口参考，不复述那些
背景与调优内容。
