# MathTalkTV 产品需求文档 (PRD)

**版本**: v1.0 MVP
**日期**: 2026-01-12
**开发周期**: 1天

---

## 0. 产品概述

### 0.1 一句话描述
MathTalkTV：初中生观看数学录播视频时，可随时进入"倾听模式"进行多轮语音问答，AI老师用语音+画板实时解答，结束后一键回到视频继续学习。

### 0.2 MVP 核心目标
- ✅ Web端视频播放（暂停/播放/倍速/全屏/进度条）
- ✅ 一键进入/退出倾听模式
- ✅ 多轮语音问答（ASR → LLM流式 → TTS播放）
- ✅ AI基于"当前视频节点"生成上下文相关解答（MVP砍掉向量召回补充节点）
- ✅ 画板累积显示文本对话+LaTeX公式（可滚动查看历史）
- ✅ 打断机制：随时退出倾听，立即回到视频
- ✅ AI回答后显示"继续提问/返回视频"按钮供学生选择

### 0.3 MVP 不做（后续迭代）
- ❌ 账号体系/付费
- ❌ 跨视频记忆
- ❌ 向量召回补充节点（只用当前时间点定位节点）
- ❌ 多轮上下文摘要压缩（直接传最近3轮原文）
- ❌ 复杂画板动作（箭头/高亮/自由绘画/几何图形，MVP仅支持文本+LaTeX）
- ❌ IDE代码执行
- ❌ 移动端适配

---

## 1. 用户角色与使用流程

### 1.1 角色
- **学生**（初中生，匿名临时会话）

### 1.2 主流程（Happy Path）

```
1. 学生打开视频页面，点击播放
2. 观看中遇到疑问 → 点击"提问"按钮
   - 视频自动暂停
   - 进入"倾听模式"

3. 倾听模式下：
   - 学生直接说话（无需按住），系统自动ASR识别
   - AI流式生成回答（边说边播放TTS）
   - 画板累积显示：问题 + 回答 + 公式（可滚动查看历史）

4. AI回答完毕后：
   - 显示两个按钮："继续提问" / "返回视频"
   - 点击"继续提问"：保持倾听，可继续追问
   - 点击"返回视频"：退出倾听，视频从暂停点继续播放

5. 多轮对话（在倾听模式下）：
   - 学生可追问"那这个公式怎么推导？"
   - 系统自动携带历史上下文（最近3轮原文）
```

### 1.3 异常流程

| 异常情况 | 处理方式 |
|---------|---------|
| ASR识别失败 | 显示"没听清，请再说一遍" |
| 倾听模式下长时间无声音 | 保持倾听状态，不自动退出 |
| LLM生成失败 | 显示兜底话术："这个问题有点复杂，建议重新观看视频或换个方式问我" |
| TTS合成失败 | 仅显示文本 + 提示"语音暂时不可用" |
| 学生在AI回答中途点击"返回视频" | 立即终止ASR/LLM/TTS链路，清空音频队列，播放视频 |
| 学生提问超纲内容 | 判断是否为数学相关：<br>- 是数学：正常回答，引导"这个知识点初中不要求，我简单讲讲..."<br>- 非数学：提示"我只能回答数学问题哦" |
| 节点生成失败（Whisper/AI识别失败） | 降级为"通用数学答疑"（无视频节点上下文） |

---

## 2. 功能需求详细说明

### 2.1 视频管理（P0）

#### 2.1.1 视频上传
- **输入**：用户上传MP4视频（≤500MB，≤30分钟）
- **处理流程**（同步，用户等待）：
  ```
  1. 上传到本地存储
  2. 调用Whisper API提取带时间戳字幕
     输出格式：[{text, start, end}, ...]
     如失败：标记视频status='failed'，降级为通用答疑模式
  3. AI识别知识点节点边界
     输入：完整字幕
     输出：[{order, startTime, endTime, title, summary}, ...]
     如失败：标记视频status='ready_no_nodes'，降级为通用答疑模式
  4. 存储到数据库（不做向量化，MVP砍掉此步骤）
  ```
- **前端展示**：进度条 + "预计需要X分钟"

#### 2.1.2 视频播放
- **基础功能**：
  - 播放/暂停
  - 倍速：0.75x / 1x / 1.25x / 1.5x / 2x
  - 全屏
  - 进度条拖动
  - 音量调节
- **状态控制**：
  - 点击"提问"按钮 → 自动暂停，记录 `pauseTime`
  - 点击"返回视频" → 从 `pauseTime` 继续播放

---

### 2.2 倾听模式（P0）

#### 2.2.1 进入/退出
- **入口**：视频下方固定"提问"按钮（麦克风图标）
- **状态变化**：
  - 未倾听：按钮显示"提问"
  - 倾听中：界面显示"倾听中..."状态，学生可随时说话
  - AI回答后：显示"继续提问" / "返回视频"两个按钮
- **超时机制**：无超时，保持倾听直到学生主动点击"返回视频"

#### 2.2.2 语音采集（ASR）
- **技术方案**：讯飞实时语音转写 WebSocket
- **录音参数**：
  - 采样率：16kHz
  - 声道：单声道
  - 格式：PCM（浏览器 MediaRecorder 捕获后转码）
- **流程**：
  ```
  1. 前端通过 getUserMedia 获取麦克风流
  2. 实时切片（建议160ms/chunk）
  3. WebSocket推送到后端
  4. 后端转发给讯飞ASR
  5. 讯飞返回识别文本（流式 + 最终确认）
  6. 前端实时显示识别文本（灰色临时 → 黑色确认）
  ```
- **VAD（语音活动检测）**：
  - 检测到2秒静音 → 判定本次提问结束
  - 发送完整问题给LLM

---

### 2.3 AI问答（P0）

#### 2.3.1 上下文召回（MVP简化版）
**输入**：
- `pauseTime`：当前视频暂停时间点
- `question`：学生问题文本
- `history`：最近3轮对话原文 `[{role: 'user', content}, {role: 'assistant', content}]`

**召回逻辑（MVP简化）**：
```python
# 仅步骤1：时间定位当前节点
current_node = find_node_by_time(video_id, pauseTime)
# 如果节点不存在（视频处理失败），则 current_node = None

# MVP不做向量召回，直接构建上下文
context = {
    "current_node": current_node,  # 可能为None（降级为通用答疑）
    "history": last_3_turns_raw  # 直接传原文，不做摘要压缩
}
```

#### 2.3.2 LLM生成
**Prompt结构**（示例）：
```
你是一位初中数学老师，正在为学生讲解视频内容。

## 当前视频节点
{如果current_node存在：}
标题：{current_node.title}
内容：{current_node.summary}
{如果current_node为None：}
（无视频节点信息，进入通用答疑模式）

## 历史对话（最近3轮原文）
{history}

## 学生当前问题
{question}

## 要求
1. 用初中生能懂的语言
2. 每次只讲一个点，不要展开过多
3. 回答控制在20-40秒（约80-160字）
4. 如需板书，输出JSON格式：{"type": "latex", "content": "..."}
5. 讲完后不要主动引导，等待学生选择"继续提问"或"返回视频"
6. 如果问题超纲，说明"这个初中不要求，我简单讲..."
7. 如果问题非数学，回复"我只能回答数学问题哦"
```

**输出格式**（流式）：
```json
{
  "type": "text",
  "content": "这个公式是二次函数的顶点公式"
}
{
  "type": "latex",
  "content": "y = a(x - h)^2 + k"
}
{
  "type": "text",
  "content": "其中(h, k)就是顶点坐标，它是通过配方法推导出来的"
}
```

#### 2.3.3 时长控制
- **目标**：20-40秒
- **最长限制**：60秒（前端计时，超时强制截断 + 收尾话术）
- **收尾话术**："这个知识点比较多，建议你先理解这部分，可以继续问我或重新看视频"

---

### 2.4 语音播报（TTS）（P0）

#### 2.4.1 技术方案
- **服务商**：讯飞语音合成
- **音色**：女声、温柔、语速适中（可配置）
- **格式**：PCM/WAV，采样率16kHz

#### 2.4.2 流式播放
**后端处理**：
```python
1. LLM每输出一个完整句子（按标点切分：。！？）
2. 立即发送给讯飞TTS合成
3. 返回音频二进制数据（可能需分片）
4. 通过WebSocket推送给前端
```

**前端播放**：
```javascript
1. 接收到音频片段 → 加入播放队列
2. 使用 <audio> 元素或 Web Audio API 逐段播放
3. 队列模式：上一段播完自动播下一段
4. 显示播放状态："AI正在回答..."
```

#### 2.4.3 停止机制
**触发条件**：
- 学生点击"返回视频"
- 学生提出新问题（打断）

**处理流程**：
```
1. 前端：停止当前音频播放，清空队列
2. WebSocket发送 STOP 指令
3. 后端：关闭讯飞TTS连接，终止LLM stream
4. 清理所有异步任务
```

---

### 2.5 画板（P0 简化版）

#### 2.5.1 布局
```
+------------------+----------------------+
|                  |  [对话历史区域]       |
|   视频播放区      |  ┌──────────────────┐|
|                  |  │ 👤 学生：这个公式  ||
|   (16:9)         |  │    怎么来的？      ||
|                  |  │                   ||
|                  |  │ 🤖 AI：我们来看... ||
|                  |  │    [LaTeX公式]     ||
|                  |  │    它是通过配方法...||
|                  |  │                   ||
|                  |  │ [继续提问] [返回视频]||
|   [提问] 按钮     |  │ (滚动查看历史)    ||
|                  |  └──────────────────┘|
+------------------+----------------------+
```

**显示逻辑**：
- 当前视频会话的所有对话累积显示
- 可滚动查看历史消息（类似聊天记录）
- 每轮对话包含：学生问题 + AI回答（文本+LaTeX混排）

#### 2.5.2 支持的元素类型
**MVP仅支持**：
1. **文本消息**：
   ```json
   {"type": "text", "role": "user|assistant", "content": "..."}
   ```

2. **LaTeX公式**：
   ```json
   {"type": "latex", "content": "y = ax^2 + bx + c"}
   ```
   - 使用 KaTeX 渲染
   - 显示为独立块（居中，浅灰背景）

**后续迭代再支持**：
- `arrow`：箭头指向
- `highlight`：高亮某行
- 手写轨迹动画

#### 2.5.3 渲染逻辑
```javascript
// 每收到一条消息
onMessage(msg) {
  if (msg.type === 'text') {
    appendText(msg.role, msg.content);
    sendToTTS(msg.content);  // 同时触发语音播放
  } else if (msg.type === 'latex') {
    appendLatex(msg.content);
    // LaTeX不播报语音，仅显示
  }
}
```

---

### 2.6 会话记忆（P0）

#### 2.6.1 存储范围
- **仅限当前视频页面会话**
- 页面刷新/关闭 → 清空
- 不跨视频记忆

#### 2.6.2 上下文管理
**存储结构**：
```javascript
{
  videoId: "xxx",
  sessionId: "uuid",
  turns: [
    {
      timestamp: 1234567890,
      pauseTime: 125.5,  // 秒
      user: "这个公式怎么来的？",
      assistant: "我们来看二次函数的顶点公式...",
      nodes: [current_node_id, related_node_ids]
    }
  ]
}
```

**传递给LLM**：
- 只传最近3轮的 `user` 和 `assistant` 文本
- 不传节点详情（避免token浪费，节点信息由召回逻辑提供）

---

### 2.7 打断与取消（P0 必须）

#### 2.7.1 触发场景
| 场景 | 用户操作 | 系统行为 |
|------|---------|---------|
| AI正在回答 | 点击"返回视频"按钮 | 立即停止TTS播放 + 终止LLM → 播放视频 |
| AI回答完毕 | 点击"返回视频"按钮 | 退出倾听模式 → 播放视频 |
| AI回答完毕 | 点击"继续提问"按钮 | 保持倾听模式，等待新问题 |
| 倾听中等待提问 | 学生直接说话 | 开始ASR识别 → LLM回答 |
| ASR正在识别 | 点击"返回视频" | 关闭ASR连接 → 播放视频 |

#### 2.7.2 后端清理任务
**必须做到**：
```python
def cancel_all():
    # 1. 关闭ASR WebSocket
    asr_client.close()

    # 2. 终止LLM stream（调用API的 cancel 方法或直接断开连接）
    llm_stream.cancel()

    # 3. 关闭TTS合成任务
    tts_client.close()

    # 4. 清空待发送队列
    message_queue.clear()

    # 5. 通知前端："已停止"
```

---

## 3. 技术架构

### 3.1 总体架构图

```
┌─────────────────────────────────────────────────────────┐
│                       前端 (React)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │视频播放器 │  │倾听按钮   │  │画板组件   │  │WS客户端 │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
└────────────────────────┬────────────────────────────────┘
                         │ WebSocket (ASR + TTS)
                         │ HTTP (视频上传/节点查询)
┌────────────────────────▼────────────────────────────────┐
│                   后端 (Python FastAPI)                  │
│  ┌─────────────────────────────────────────────────────┐│
│  │  API 层                                              ││
│  │  - /upload (视频上传+处理)                            ││
│  │  - /ws/chat (WebSocket: ASR+LLM+TTS)                 ││
│  │  - /videos/:id/nodes (查询节点)                       ││
│  └─────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────┐│
│  │  业务逻辑层                                           ││
│  │  - VideoProcessor (Whisper处理)                      ││
│  │  - NodeExtractor (AI识别节点)                        ││
│  │  - ContextRetriever (时间定位+向量召回)               ││
│  │  - ConversationManager (会话管理)                    ││
│  └─────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────┐│
│  │  外部服务集成                                         ││
│  │  - 讯飞ASR WebSocket                                 ││
│  │  - 讯飞TTS HTTP                                      ││
│  │  - LLM API (uiuiapi / DeepSeek备用)                 ││
│  │  - Whisper API (OpenAI)                             ││
│  └─────────────────────────────────────────────────────┘│
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                    数据存储层                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │ SQLite       │  │ Chroma/向量库 │  │ 本地文件存储   │ │
│  │ (视频元数据) │  │ (节点向量)    │  │ (MP4/字幕)     │ │
│  └──────────────┘  └──────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 3.2 技术选型

| 模块 | 技术方案 | 理由 |
|------|---------|------|
| **前端框架** | React 18 + TypeScript | 组件化、类型安全 |
| **视频播放器** | Video.js / React-Player | 开箱即用，支持倍速/全屏 |
| **画板渲染** | KaTeX (LaTeX) + 自定义React组件 | 轻量、渲染快 |
| **WebSocket** | 原生 WebSocket API | 简单、实时双向通信 |
| **后端框架** | FastAPI (Python 3.10+) | 异步、WebSocket支持好、开发快 |
| **ASR** | 讯飞实时语音转写 | 中文识别准、延迟低(<500ms) |
| **TTS** | 讯飞语音合成 | 音色自然、流式支持、首字延迟<1s |
| **LLM** | uiuiapi (主) / DeepSeek (备) | 成本可控、切换灵活 |
| **Whisper** | OpenAI Whisper API | 准确度高、支持时间戳 |
| **关系数据库** | SQLite | 单机足够、零配置 |
| **文件存储** | 本地磁盘 | MVP阶段不需要OSS |

**MVP砍掉的技术**：
- ❌ Embedding模型（text-embedding-3-small）
- ❌ 向量数据库（ChromaDB）

### 3.3 核心模块设计

#### 3.3.1 视频处理流程（MVP简化）
```python
# video_processor.py
class VideoProcessor:
    async def process_video(self, video_path: str) -> dict:
        try:
            # 1. 提取字幕
            subtitles = await self.whisper_extract(video_path)
        except Exception as e:
            # Whisper失败，标记状态
            return {"video_id": video_id, "status": "failed"}

        try:
            # 2. AI识别节点
            nodes = await self.extract_nodes(subtitles)
        except Exception as e:
            # 节点识别失败，降级为无节点模式
            return {"video_id": video_id, "status": "ready_no_nodes"}

        # 3. 存储（MVP不做向量化）
        video_id = self.save_to_db(video_path, subtitles, nodes)

        return {"video_id": video_id, "nodes_count": len(nodes), "status": "ready"}
```

#### 3.3.2 实时对话流程（含AI回答后按钮）
```python
# ws_chat.py
@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await websocket.accept()
    session = ConversationSession()

    try:
        async for message in websocket.iter_json():
            if message["type"] == "start_listening":
                # 开启ASR监听
                await session.start_asr(websocket)

            elif message["type"] == "audio_chunk":
                # 转发给讯飞ASR
                await session.process_audio(message["data"])

            elif message["type"] == "question_complete":
                # 触发LLM回答
                context = await retrieve_context(
                    video_id=message["videoId"],
                    pause_time=message["pauseTime"],
                    question=message["question"],
                    history=session.history  # 最近3轮原文
                )

                # 流式生成 + TTS
                async for chunk in generate_answer(context):
                    if chunk["type"] == "text":
                        await websocket.send_json(chunk)
                        # 同时合成TTS
                        audio = await tts_synthesize(chunk["content"])
                        await websocket.send_bytes(audio)
                    elif chunk["type"] == "latex":
                        await websocket.send_json(chunk)

                # AI回答完毕，通知前端显示按钮
                await websocket.send_json({
                    "type": "answer_complete",
                    "message": "show_buttons"  # 前端显示"继续提问/返回视频"
                })

            elif message["type"] == "continue_asking":
                # 学生点击"继续提问"，保持倾听
                await websocket.send_json({"type": "status", "message": "listening"})

            elif message["type"] == "stop":
                # 学生点击"返回视频"，取消所有任务
                await session.cancel_all()

    except WebSocketDisconnect:
        await session.cleanup()
```

#### 3.3.3 上下文召回（MVP简化）
```python
# context_retriever.py
class ContextRetriever:
    def retrieve(self, video_id: str, pause_time: float, question: str):
        # 仅步骤1：时间定位当前节点
        current_node = self.find_node_by_time(video_id, pause_time)
        # 如果视频没有节点（处理失败），current_node = None

        return {
            "current_node": current_node  # 可能为None
        }
```

---

## 4. 数据结构设计

### 4.1 数据库表结构（SQLite）

#### videos 表
```sql
CREATE TABLE videos (
    id TEXT PRIMARY KEY,
    filename TEXT NOT NULL,
    filepath TEXT NOT NULL,
    duration REAL,  -- 秒
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status TEXT  -- 'processing' | 'ready' | 'ready_no_nodes' | 'failed'
);
```

#### nodes 表
```sql
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,
    video_id TEXT NOT NULL,
    order_num INTEGER,
    start_time REAL,
    end_time REAL,
    title TEXT,
    summary TEXT,
    FOREIGN KEY (video_id) REFERENCES videos(id)
);
```

#### subtitles 表
```sql
CREATE TABLE subtitles (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    video_id TEXT NOT NULL,
    text TEXT,
    start_time REAL,
    end_time REAL,
    FOREIGN KEY (video_id) REFERENCES videos(id)
);
```

### 4.2 会话数据结构（内存）

```python
class ConversationSession:
    def __init__(self):
        self.session_id = str(uuid.uuid4())
        self.video_id = None
        self.history = []  # List[Turn]，最多保留3轮
        self.asr_client = None
        self.tts_client = None
        self.llm_task = None

class Turn:
    timestamp: float
    pause_time: float
    user_text: str
    assistant_text: str
    node_id: Optional[str]  # 当前使用的节点ID（可能为None）
```

**MVP移除**：
- ❌ 向量数据库结构（ChromaDB）

---

## 5. 接口定义

### 5.1 HTTP API

#### POST /api/upload
**功能**：上传视频并同步处理

**请求**：
```
Content-Type: multipart/form-data

file: <binary>
```

**响应**：
```json
{
  "video_id": "uuid",
  "status": "ready" | "ready_no_nodes" | "failed",
  "nodes_count": 12,
  "duration": 1800.5
}
```

#### GET /api/videos/:id
**功能**：获取视频信息

**响应**：
```json
{
  "id": "uuid",
  "filename": "一元二次方程.mp4",
  "duration": 1800.5,
  "nodes": [
    {
      "order": 1,
      "start_time": 0,
      "end_time": 120.5,
      "title": "引入：什么是一元二次方程"
    }
  ]
}
```

### 5.2 WebSocket 协议

**连接地址**：`ws://localhost:8000/ws/chat?video_id={id}`

#### 客户端 → 服务器

**1. 开始倾听**
```json
{
  "type": "start_listening",
  "pause_time": 125.5
}
```

**2. 音频数据**
```json
{
  "type": "audio_chunk",
  "data": "<base64_pcm>"
}
```

**3. 问题完成（VAD检测到静音）**
```json
{
  "type": "question_complete",
  "question": "这个公式怎么来的？",
  "pause_time": 125.5
}
```

**4. 停止/打断**
```json
{
  "type": "stop"
}
```

**5. 继续提问（AI回答后）**
```json
{
  "type": "continue_asking"
}
```

#### 服务器 → 客户端

**1. ASR识别结果（流式）**
```json
{
  "type": "asr_partial",
  "text": "这个公式"
}
{
  "type": "asr_final",
  "text": "这个公式怎么来的？"
}
```

**2. AI回答（流式）**
```json
{
  "type": "text",
  "role": "assistant",
  "content": "我们来看二次函数的顶点公式"
}
{
  "type": "latex",
  "content": "y = a(x - h)^2 + k"
}
```

**3. TTS音频（二进制）**
```
WebSocket Binary Frame: <audio_pcm_data>
```

**4. 状态通知**
```json
{
  "type": "status",
  "message": "thinking" | "speaking" | "stopped" | "listening"
}
```

**5. AI回答完毕（触发按钮显示）**
```json
{
  "type": "answer_complete",
  "message": "show_buttons"
}
```

---

## 6. 非功能需求

### 6.1 性能指标

| 指标 | 目标值 |
|------|--------|
| 视频上传+处理时长 | ≤ 视频时长的0.5倍（10分钟视频 → 5分钟处理完） |
| ASR延迟（说完到显示文字） | < 500ms |
| LLM首token延迟 | < 1.5s |
| TTS首字播放延迟 | < 1s |
| 打断响应时间 | < 300ms |

### 6.2 可靠性
- WebSocket断连自动重连（最多3次）
- LLM/ASR/TTS失败时显示友好错误提示
- 视频处理失败时标记状态，允许重新上传

### 6.3 可用性
- 界面简洁，初中生可无指导上手
- 倾听状态明确："倾听中..." / "AI回答中..." / "继续提问/返回视频"按钮
- 所有异步操作显示loading状态
- 画板可滚动查看历史对话

---

## 7. MVP开发计划（1天）

### 7.1 时间分配

| 时间段 | 任务 | 负责人 | 产出 |
|--------|------|--------|------|
| 0-2h | 项目初始化 + 数据库设计 | 后端 | FastAPI项目结构 + SQLite表 |
| 0-2h | 前端页面框架搭建 | 前端 | 视频播放器 + 倾听按钮 + 画板布局 |
| 2-4h | 视频上传+Whisper处理 | 后端 | /upload接口 + 字幕提取 |
| 2-4h | WebSocket连接+音频采集 | 前端 | 麦克风录音 + WS发送 |
| 4-6h | ASR集成（讯飞） | 后端 | 实时转写 |
| 4-6h | 画板渲染（文本+LaTeX） | 前端 | KaTeX组件 + 滚动历史 |
| 6-8h | 节点识别（无向量化） | 后端 | AI节点提取 + 数据库存储 |
| 6-8h | TTS播放队列 + 按钮交互 | 前端 | 音频流式播放 + "继续提问/返回视频"按钮 |
| 8-10h | LLM问答+上下文召回（简化版） | 后端 | 完整对话流程（仅当前节点+最近3轮原文） |
| 8-10h | 打断机制 | 前后端 | 停止按钮 + 任务清理 |
| 10-12h | 联调 + Bug修复 | 全员 | 完整流程跑通 |

### 7.2 关键里程碑
- **4h**：视频上传+字幕提取可用
- **8h**：单轮问答可用（ASR → LLM → TTS）+ AI回答后显示按钮
- **10h**：多轮对话+打断机制可用
- **12h**：MVP完整功能测试通过

---

## 8. 风险与应对

| 风险 | 影响 | 应对方案 |
|------|------|----------|
| Whisper API调用慢（>1分钟） | 视频上传体验差 | 前端显示进度条 + "可能需要X分钟" |
| 讯飞ASR识别率低 | 问答体验差 | 提供"重新提问"按钮 + 显示识别文本让学生确认 |
| LLM回答超纲/不准确 | 教学质量差 | Prompt加强限制 + 人工review高频问题 |
| TTS播放卡顿 | 语音断断续续 | 增大音频缓冲 + 降低采样率到8kHz |
| 1天时间不够 | 功能不完整 | ✅已砍：向量召回、上下文摘要、复杂画板动作 |

---

## 9. 后续迭代方向（V2）

- [ ] 向量召回补充节点：问"前面讲过吗"时召回相关节点
- [ ] 多轮上下文智能摘要：用LLM压缩历史对话
- [ ] 画板增强：箭头指向、高亮、手写动画、几何图形
- [ ] 账号体系：学习记录、收藏错题
- [ ] 跨视频记忆：知识图谱关联
- [ ] 题目练习：看完视频后推送相关练习题
- [ ] 移动端适配
- [ ] 多人协作：老师可批注视频节点

---

## 附录

### A. Prompt模板参考

#### A.1 节点识别Prompt
```
你是一位初中数学教学专家。现在给你一个数学教学视频的完整字幕（带时间戳），请识别出知识点节点的边界。

## 字幕内容
{subtitles}

## 任务
1. 识别每个独立知识点的开始和结束时间
2. 为每个节点起标题（5-10字）
3. 总结节点核心内容（30-50字）

## 输出格式（JSON）
[
  {
    "order": 1,
    "start_time": 0,
    "end_time": 120.5,
    "title": "引入：什么是一元二次方程",
    "summary": "从实际问题引入一元二次方程的定义，解释各项的含义"
  }
]

## 要求
- 每个节点时长建议1-3分钟
- 节点边界应在教师讲解段落的自然停顿处
- 标题要体现知识点名称
```

#### A.2 问答Prompt（见2.3.2）

### B. 测试用例

#### B.1 基础流程测试
1. 上传视频 → 等待处理 → 查看节点列表
2. 播放视频 → 点击"提问" → 说话提问 → 听AI回答 → 点击"返回视频"
3. 多轮对话：提问 → AI回答后点击"继续提问" → 追问 → 再追问 → 点击"返回视频"
4. 画板滚动：多轮对话后，滚动查看历史消息

#### B.2 异常测试
1. ASR识别失败：在嘈杂环境提问
2. 打断测试：AI回答到一半点击"返回视频"
3. 超时测试：进入倾听后长时间不说话（应保持倾听状态）
4. 超纲测试：提问"微积分怎么算"
5. 节点失败降级：测试Whisper失败/节点识别失败时的通用答疑模式

---

**文档结束**

如有疑问请联系产品负责人。