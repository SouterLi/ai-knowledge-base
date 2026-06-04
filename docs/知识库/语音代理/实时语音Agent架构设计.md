# 实时语音 Agent 架构设计

## 主题选择记录

- **本次序号**：第 27 篇。
- **README 位置**：README 目录表第 27 篇，位于「多模态 LLM 图像与文档理解」之后、「开源大模型私有化部署与推理服务」之前，归入「专项进阶」卷。
- **选题理由**：README 已经预留「实时语音 Agent 架构设计」主题，但仓库中缺少对应文档。面试中语音 Agent 常被追问实时链路、打断、低延迟、状态同步和工具调用安全，本篇补齐这一专项进阶主题。
- **去重判断**：已有「LLM 流式响应与异步任务架构」侧重文本流式和长任务，「Agent 规划执行与可靠性治理」侧重通用 Agent 循环。本篇聚焦 **实时音频输入输出、全双工交互、VAD/ASR/TTS、Barge-in 打断和语音场景的端到端延迟治理**，不与已有主题重复。

## 核心概念

### 1. 实时语音 Agent 是什么

实时语音 Agent 是把「听、想、说、做」连成闭环的 AI 应用：用户通过麦克风持续输入音频，系统实时识别语音、理解意图、调用工具或业务系统，并以流式语音回答。它和普通文本 Chatbot 最大的差异不是输入格式变成了音频，而是交互变成了 **低延迟、可打断、强状态同步** 的实时系统。

面试中要先讲清楚边界：语音 Agent 不等于简单的「ASR 转文字 + LLM + TTS」。生产级语音 Agent 还要处理说话人停顿、噪声、抢话打断、半句话意图、音频播放取消、工具执行确认、通话记录、隐私合规和失败降级。

### 2. 两类主流架构

| 架构 | 链路 | 优点 | 风险 |
| --- | --- | --- | --- |
| 级联式语音 Agent | VAD → ASR → LLM/Agent → TTS | 模块可替换、可观测性强、适合企业落地 | 端到端延迟叠加，模块间状态同步复杂 |
| 原生实时多模态模型 | 音频直接进模型，音频或文本流式输出 | 延迟低、自然度高、打断体验好 | 黑盒更强，工具与审计、成本、私有化可控性要重点评估 |

实际落地常用混合方案：实时模型负责自然对话和快速响应，业务工具调用、权限校验、审计和高风险动作仍由后端 Agent 编排层控制。

### 3. 全双工与半双工

- **半双工**：用户说完后系统再说，类似语音助手的按住说话。实现简单，但交互不自然。
- **全双工**：用户和系统可以同时有音频流，系统说话时用户能插话打断。更像真人通话，但需要处理回声消除、VAD、播放取消和会话状态回滚。

语音 Agent 面试高频点是 Barge-in，即用户在系统播报过程中插话，系统必须及时停止 TTS、取消未完成生成，并基于新的用户输入继续对话。

### 4. 延迟指标

实时语音体验不是只看总耗时，而要拆成多段：

| 指标 | 含义 | 面试关注点 |
| --- | --- | --- |
| VAD 延迟 | 判断用户开始/结束说话的耗时 | 太短会截断，太长会显得迟钝 |
| ASR Partial 延迟 | 产生中间识别文本的时间 | 决定能否边听边理解 |
| TTFT | Agent 首个文本 token 时间 | 决定系统是否快速反应 |
| TTFB Audio | 首个可播放音频帧时间 | 用户真正感知到的响应速度 |
| Barge-in 延迟 | 用户插话到系统停播的时间 | 直接影响自然度 |

核心表达：**语音 Agent 的优化目标是首响快、可打断、不中断上下文，而不是只追求最终回答生成得快**。

## 核心知识点

### 1. 端到端链路

```text
麦克风采集 → 音频帧编码 → WebRTC/WebSocket → VAD → 流式 ASR
→ 增量意图理解 → Agent 状态机 → 工具调用/知识检索 → 流式生成
→ 流式 TTS → 播放队列 → 打断/取消 → 日志与评估回流
```

这条链路里每个环节都要支持流式，否则一个同步阻塞点就会破坏实时体验。比如 ASR 如果只在整句结束后返回，LLM 再快也做不到自然抢答；TTS 如果只能整段合成，首个音频包会很慢。

### 2. 通信协议选型

| 协议 | 适用 | 说明 |
| --- | --- | --- |
| WebRTC | 浏览器实时音视频、弱网、回声消除 | 首选实时通话协议，天然支持音频轨、抖动缓冲和 NAT 穿透 |
| WebSocket | App/服务端双向事件流 | 实现简单，适合自定义音频帧和控制事件 |
| SSE | 文本单向流 | 适合文本流式，不适合实时双向音频 |

语音 Agent 一般不让前端直连模型。后端实时网关要负责鉴权、会话管理、限流、模型路由、工具权限、审计和降级。

```json
{
  "type": "input_audio_frame",
  "session_id": "s_123",
  "sequence": 42,
  "codec": "pcm16",
  "sample_rate": 16000,
  "payload": "base64..."
}
```

控制事件必须和音频帧分离，例如 `speech_started`、`speech_stopped`、`assistant_audio_delta`、`barge_in`、`tool_call_requested`、`error`、`done`。只传裸音频会导致状态难以排查。

### 3. VAD 与轮次切分

VAD（Voice Activity Detection）用于判断用户是否正在说话。它不是简单地检测音量，因为真实环境有背景噪声、键盘声、多人说话和静音停顿。

常见策略：

- 前端轻量 VAD：快速感知用户开始说话，用于打断播放。
- 服务端 VAD：结合更稳定的模型和上下文，判断一句话是否结束。
- 端点检测：用静音时长、语义完整性和 ASR 置信度共同判断是否可以提交给 Agent。

面试要强调取舍：结束阈值太短会把一句话切碎，太长会让系统反应慢。客服、会议、车载场景的阈值通常不同，需要用真实录音评估。

### 4. 流式 ASR 与增量理解

流式 ASR 会不断输出 partial transcript，最终再给 final transcript。Agent 可以利用 partial 做提前准备，但不能把 partial 当成稳定事实直接执行高风险动作。

```python
async def handle_asr_events(asr_stream, agent_state):
    async for event in asr_stream:
        if event.type == "partial":
            # 中文注释：partial 只用于意图预热，不直接触发写操作
            agent_state.update_live_transcript(event.text)
            await agent_state.prefetch_candidates(event.text)
        elif event.type == "final":
            # 中文注释：final 文本稳定后再进入正式 Agent 轮次
            await agent_state.commit_user_turn(event.text)
```

典型面试坑：候选人只说「ASR 识别完再调大模型」。更好的回答是：partial 用于预取知识、预测意图、提前加载工具上下文；final 用于确认轮次和执行动作。

### 5. Agent 状态机

语音 Agent 不适合完全依赖自由循环，因为实时通话需要明确状态。常见状态包括：

```text
IDLE → LISTENING → THINKING → SPEAKING → INTERRUPTED
                  ↘ TOOL_PENDING → CONFIRMING → SPEAKING
```

状态机要解决三个问题：

1. 当前谁在说话：用户、助手，还是双方重叠。
2. 当前输出是否还能继续：如果被打断，TTS 和 LLM 生成都要取消。
3. 当前业务动作是否已经生效：读操作可重试，写操作要幂等，高风险写操作要确认。

```python
from enum import Enum

class VoiceState(str, Enum):
    IDLE = "idle"
    LISTENING = "listening"
    THINKING = "thinking"
    SPEAKING = "speaking"
    INTERRUPTED = "interrupted"

class SessionState:
    def __init__(self):
        self.state = VoiceState.IDLE
        self.current_response_id = None
        self.cancel_event = None

    async def on_user_speech_started(self):
        if self.state == VoiceState.SPEAKING:
            # 中文注释：用户插话时先取消仍在播放和生成的回答
            self.state = VoiceState.INTERRUPTED
            if self.cancel_event:
                self.cancel_event.set()
        self.state = VoiceState.LISTENING
```

### 6. Barge-in 打断设计

Barge-in 是语音 Agent 的核心体验。正确做法不是只让前端静音，而是端到端取消：

- 前端停止播放当前音频队列。
- 后端取消 TTS 合成任务。
- 后端取消或截断 LLM 流式生成。
- 会话状态记录「上一轮被打断」，避免模型继续引用未播完内容。
- 如果工具调用已经开始，要区分可取消、不可取消和需要补偿的动作。

```python
async def cancel_response(session, response_id):
    # 中文注释：打断时要同时处理播放、TTS、LLM 和工具任务
    await session.player.stop(response_id)
    await session.tts.cancel(response_id)
    await session.llm.cancel(response_id)

    if session.current_tool_call and session.current_tool_call.can_cancel:
        await session.current_tool_call.cancel()

    session.history.mark_interrupted(response_id)
```

面试表达重点：**用户没听到的内容不能默认进入长期对话事实**。如果模型生成了十句话但只播了两句，后续上下文要知道实际播放边界。

### 7. TTS 流式合成与播放队列

TTS 不应该等完整答案生成后再合成。更好的方式是按句子、短语或语义片段流式合成：

```text
LLM delta → 句子切分 → TTS chunk → audio delta → 播放队列
```

需要注意：

- 句子太短：TTS 韵律差，频繁请求增加成本。
- 句子太长：首个音频慢，打断后浪费更多。
- 播放队列要支持清空、暂停、恢复和 response_id 校验，避免旧音频串到新轮次。

```python
async def stream_tts(llm_tokens, tts_client, audio_sink):
    buffer = ""
    async for token in llm_tokens:
        buffer += token
        if buffer.endswith(("。", "？", "！")) and len(buffer) >= 12:
            # 中文注释：按语义片段合成，兼顾首响和自然度
            async for audio in tts_client.synthesize_stream(buffer):
                await audio_sink.send(audio)
            buffer = ""
```

### 8. 工具调用与高风险确认

语音 Agent 很容易被设计成「用户一句话直接办事」，但生产环境必须分层：

| 动作类型 | 示例 | 处理方式 |
| --- | --- | --- |
| 只读 | 查询订单、查询余额 | 可直接执行，但要校验身份和租户 |
| 低风险写 | 修改昵称、创建提醒 | 可执行，但要记录幂等键 |
| 高风险写 | 转账、退款、取消订单、发送外部消息 | 必须语音复述关键信息并让用户明确确认 |

```python
def build_idempotency_key(session_id: str, turn_id: str, tool_name: str) -> str:
    # 中文注释：防止网络重试或模型重复调用导致业务重复执行
    return f"{session_id}:{turn_id}:{tool_name}"
```

高风险确认不要只问「确认吗」。应复述对象、金额、影响和不可逆风险，例如：「我将为订单 123 申请退款 98 元，提交后可能无法撤销。请明确说确认退款。」

### 9. 延迟优化

语音 Agent 的延迟优化要并行化，而不是等每一步完成：

- ASR partial 到达时预取用户画像、知识库和候选工具。
- LLM 输出一小段就触发 TTS，而不是等完整答案。
- TTS 音色模型预热，常用开场白可缓存。
- 对工具调用设置超时和降级话术。
- 弱网下动态降低采样率或切到文本输入。

可用粗略预算表达：

```text
VAD 100~300ms + ASR partial 200~600ms + LLM TTFT 300~1000ms
+ TTS 首包 200~800ms + 网络/播放 100~300ms
```

不要死背数字，面试中更重要的是说明每段如何观测、如何优化、如何在业务 SLA 下取舍。

### 10. 可观测性与评估

语音 Agent 的日志不能只存最终文本，要能回放一通会话：

- 音频事件时间线：speech_started、speech_stopped、barge_in、assistant_audio_delta。
- ASR partial/final、置信度、修改历史。
- LLM 请求版本、Prompt 版本、工具调用参数和结果。
- TTS 首包时间、播放完成比例、被打断位置。
- 用户体验指标：打断成功率、平均抢话停播时间、转人工率、任务完成率。

评估要覆盖：

1. 识别准确率：ASR 字错率、关键词识别。
2. 对话质量：意图理解、任务完成、上下文保持。
3. 实时体验：首响、停顿、打断、尾音截断。
4. 安全合规：身份确认、越权、敏感信息播报。

### 11. 隐私与安全

语音比文本更敏感，因为原始音频包含声纹、环境声和旁人信息。生产设计要考虑：

- 明示录音和用途，按合规要求保留或删除。
- 原始音频、转写文本、日志分级存储和脱敏。
- 声纹识别不能替代强身份认证。
- 防止语音提示注入，例如用户播放一段录音要求系统忽略规则。
- 外放场景避免直接播报完整身份证、手机号、余额等敏感信息。

## 高频面试问题与标准答案

**Q1：实时语音 Agent 和普通文本 Agent 最大区别是什么？**
我会先从交互形态说起。文本 Agent 主要是一次输入、一次输出，重点在规划、工具调用和结果质量；实时语音 Agent 是连续音频流，重点还包括低延迟、轮次切分、打断、播放状态和音频链路稳定性。它不是简单套一层 ASR 和 TTS，而是要把 VAD、流式 ASR、Agent 状态机、流式 TTS、Barge-in 和工具安全放到同一个会话状态里治理。

**Q2：如果让你设计一个语音客服 Agent，整体架构怎么说？**
我会采用分层架构：前端通过 WebRTC 或 WebSocket 上传音频帧，实时网关做鉴权、限流和会话管理；音频进入 VAD 和流式 ASR，ASR partial 用来预取知识和候选工具，final 文本进入 Agent 编排层；Agent 根据意图调用知识库或业务工具；回答通过 LLM 流式生成，再按语义片段做 TTS 流式合成并下发播放。全链路记录事件时间线、模型版本、工具调用和播放进度，方便复盘。

**Q3：为什么不能只用“ASR 完整识别后再调用 LLM”？**
这样能做出功能，但体验会偏慢。语音场景用户感知的是停顿，ASR 等整句结束、LLM 再生成、TTS 再整段合成，延迟会叠加。我更倾向于用 partial transcript 做增量理解和预取，比如提前召回知识、加载订单信息，但真正执行工具和确认轮次要等 final transcript，避免 partial 识别错误触发副作用。

**Q4：Barge-in 打断怎么实现？**
打断不能只在前端把声音停掉。用户开始说话后，前端要立即清空播放队列，后端要取消当前 TTS 和 LLM 流式生成，并把上一轮回答标记为 interrupted。如果工具调用已经发出，还要判断是否可取消；不可取消的写操作要靠幂等和补偿处理。还有一个细节是上下文要记录用户实际听到了哪部分，不能把没播完的内容当成用户已经知道的事实。

**Q5：语音 Agent 的延迟应该怎么优化？**
我会先拆指标，而不是笼统说优化接口。链路包括 VAD、ASR partial、LLM TTFT、TTS 首包、网络和播放。优化手段包括 ASR partial 预取、模型和音色预热、LLM 流式输出、TTS 分片合成、常见话术缓存、工具超时降级、WebRTC 降低弱网抖动。最后要用 P50/P95 和 Barge-in 延迟看真实体验，不只看平均耗时。

**Q6：WebRTC 和 WebSocket 选哪个？**
如果是浏览器或移动端实时通话，我优先考虑 WebRTC，因为它在音频采集、回声消除、抖动缓冲、弱网和 NAT 穿透上更成熟。WebSocket 更适合自定义协议、服务端内部链路或简单 App 场景，实现成本低。无论选哪个，都要把音频帧和控制事件分开设计，否则打断、错误恢复和排障都会困难。

**Q7：VAD 阈值怎么定？**
我不会拍一个固定值。VAD 要结合场景调，比如客服通话可以稍微等用户说完，车载或助手场景要更敏捷。阈值太短会把停顿切成多轮，太长会显得迟钝。生产上会结合静音时长、ASR 置信度和语义完整性，再用真实录音评估截断率、误触发率和平均响应延迟。

**Q8：语音 Agent 调用工具时有什么风险？**
最大风险是语音识别错、用户表达模糊、模型重复调用或被打断后状态不一致。我的做法是按风险分层：只读工具做身份和租户校验；低风险写操作加幂等键；高风险写操作必须复述关键字段并要求用户明确确认。尤其是金额、订单、外部发送这类动作，不能只靠模型一句“用户应该是这个意思”就执行。

**Q9：怎么评估一个语音 Agent 做得好不好？**
我会分四层评估。第一层是音频和 ASR，比如字错率、关键词识别、端点检测准确率。第二层是实时体验，比如首响、TTS 首包、Barge-in 停播时间、尾音截断。第三层是任务效果，比如意图识别、工具成功率、一次解决率、转人工率。第四层是安全合规，比如身份确认、越权和敏感信息播报。只看最终回答文本是不够的。

**Q10：用户打断后，上一轮生成到一半的内容要不要放进上下文？**
不能简单全放。语音里要区分模型生成了什么、TTS 合成了什么、用户实际听到了什么。后续上下文至少要知道上一轮在什么位置被打断。否则模型可能说“刚才我已经解释过”，但用户其实没听到。我的做法是记录 response_id、播放进度和 interrupted 标记，只把已播放或业务上必须保留的部分进入对话事实。

**Q11：原生实时多模态模型能不能替代 ASR + LLM + TTS 级联架构？**
部分场景可以，尤其是追求自然对话和低延迟的助手类产品。但企业落地我不会直接说完全替代，因为级联架构在审计、模块替换、私有化、成本控制、工具权限和错误定位上更清晰。更稳妥的方案是按场景选择：开放闲聊和轻任务可以用实时模型，高风险业务动作仍通过后端 Agent 编排和工具治理层。

**Q12：语音场景有哪些隐私合规点？**
语音比文本更敏感，因为原始音频可能包含声纹、环境声和旁人信息。需要明示录音用途和保留周期，原始音频与转写文本分级存储，日志脱敏，敏感字段播报时做掩码。声纹可以作为辅助识别，但不能替代强认证。还有一个容易忽略的点是外放场景，Agent 不应该直接读出完整身份证、手机号、余额这类敏感信息。
