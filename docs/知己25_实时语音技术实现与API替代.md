<!-- PUBLIC-SHARING EDITION: 精简私有工程标识、实际内部端口；教学内容与临的署名补充保留。 -->
# 知己 25 VOICE：实时双向语音技术实现、API 替代与成本模型

> **技术分享版 / 2026-10-11 / R1**  
> 读者：希望复现类似实时 AI 电话体验的开发者。  
> **事实等级**：本文区分「已在 25 冻结工程实现」「外部 API 官方能力」「建议的实现」「尚未全栈验证」。不是生产部署手册，也不构成厂商性能承诺。

## 0. TL;DR

25 的思路不是「语音识别 → 大模型 → 语音合成」一次性串行请求，而是一个**始终在线的音频管道 + 独立话轮判定 + 可取消的 LLM/TTS 流 + 音频播放权状态机**。

- **实时媒体**：浏览器 `getUserMedia` → `AudioWorklet` → 16 kHz / mono / PCM16 / **20 ms 每帧** → 双向 WebSocket；后端音频经过一个唯一的 `PlaybackGate` 才能下发；客户端环形缓冲实际出声。
- **大脑与嘴解耦**：ASR partial/final、VAD 和 Turn Policy 共同判断是否提交用户回合；LLM `delta` 经 `SpeakableChunker` 转成可念短语，TTS 不必等待整轮文本完毕。
- **打断优先**：音量/人声只能先形成疑似插话证据；Broker 才能确认并撤销上一段 `generation_id`；客户端与服务端双重丢弃过期音频，防止「已经叫停还接着说」。
- **模型与供应商可换**：ASR / TTS / LLM 通过适配器替换，**Broker / Turn authority / PlaybackGate 不跟着供应商一起换**。
- **预算结论（2026-10-11，公开标价，示例）**：一小时连续通话，若输入上云 ElevenLabs Scribe Realtime、输出用 ElevenLabs Flash/Turbo、文本大脑按 GPT-4.1 mini 的假设 token 用量，则 **约 $0.681 / 小时**，不含网络、托管、税、订阅门槛、重复生成和额外服务；保持本地 ASR+TTS 时，此示例的云端费用只剩 LLM **$0.051 / 小时**。示例不是知己真实账单。

---

## 1. 现有工程的可核对依据

本报告依据 **25 P6 Web MVP 冻结提交 `内部冻结快照（源码未公开）`（2026-10-04）** 中的源码和协议，而不是把未落地的研究当已上线产品。

| 组件 / 证据 | 冻结源码中的相对路径 | 状态与边界 |
|---|---|---|
| 媒体传输合同 | `contracts/p6_webmedia_transport_r1.md` | 冻结的 NONLIVE Web MVP 媒体协议；浏览器上行 WS PCM、下行 WS PCM。 |
| 麦克风采集 | `client/js/capture.js`、`client/worklet/capture-worklet.js` | 浏览器实时采音、16 kHz 20 ms PCM 分帧；AEC/NS/AGC。 |
| 下行播放 | `client/js/playback.js`、`client/worklet/playback-worklet.js` | PCM ring buffer、预缓冲、播放回执、撤销与 flush。 |
| 调度中心 | `server/broker/lab-broker.js`、`server/broker/turn-evidence-aggregator.js` | 汇总证据、提交回合、驱动生成。 |
| 话轮判定 | `server/adapters/turn/turn-policies.js` | 最小静音、ASR final、未决上传、续句保护等。 |
| 流式可念文本 | `server/broker/speakable-chunker.js` | 强/弱断句、最大长度压力切分、generation 撤销。 |
| 本地声线 | `server/adapters/tts/local-clone-tts.js` | `LOCAL_CLONE_SLOT` 连接 MLX Qwen3-TTS 0.6B Base 候选的本地服务；`frames-v1` 音频流及结束核对。 |
| 插话判定 | `server/broker/barge-in-controller.js` | 疑似 → 确认 → 撤销；回声与 ASR 证据参与判断。 |
| 唯一下行安全闸 | `server/broker/playback-gate.js` | generation / ownership / sequence 三重校验，不让 stale 或 duplicate 音频出声。 |
| 外围服务 | `server/adapters/resident/*`、`server/adapters/asr/*`、`server/transport/ws-transport.js` | 与特定模型解耦的适配与传输。 |

**尚未据此证明**：最新 25×30 单一产品总装、正式 iPhone Safari 真实链全通、长期并发指标、视频数字人的生产接入。后续的 Resident Acoustic Embodiment（多轨 Speech/Vocalization/Foley/Spatial）虽有非生产软件验收记录，但也不应被理解成「所有真实设备/声效已经端到端 PASS」。

---

## 2. 总体体系结构

```text
┌───────────── 浏览器 / 30 UI ─────────────┐
│  Mic getUserMedia → Capture AudioWorklet  │
│       PCM16 16k/mono, 20ms/frame          │
│       ↑                 ↓                 │
│  Speaker ← ring player  WS binary frames  │
└──────────────┬────────────────────────────┘
               │ 单会话、双向 WS
     ┌─────────▼───── 25 Voice Broker ──────────────────────────┐
     │  Uplink frames → VAD evidence ─┐                       │
     │                 → streaming ASR ├→ Turn Policy → Commit  │
     │                                ┘             │         │
     │  BargeInController / EchoGuard ←── playback  │         │
     │  GenerationRegistry + PlaybackOwnership       │         │
     │                                                ▼         │
     │  Resident/LLM adapter → token deltas → SpeakableChunker  │
     │                                            │             │
     │                                     TTS adapter         │
     │                                            │ PCM frames  │
     │                                        PlaybackGate      │
     └────────────────────────────────────────────┬──────────────┘
                                                  ▼
                                         WS downlink → speaker

控制与存储边界：37 Call/Media Session 为呼叫权威；31 管唯一通话记录；
30 管 UI；25 管音频生命周期。旁挂听觉/SOM 不取得 Turn/Call 权威。
```

### 2.1 为什么选择 WebSocket PCM，而不是语音文件上传？

- `MediaRecorder` 每一整句上传适合「对讲机式」应用，但需要额外的整句等待，插话与低延迟不自然。
- P6 NONLIVE 选择 **WebSocket PCM16**，协议原样重用 25 已验收的音频帧；未额外引入 WebRTC SFU。以后可将传输替换为 WebRTC，前提是**不得重写**回合归属与播放权合同。
- 原始 16k×16bit×1 声道 ≈ **256 kbit/s ≈ 32 KB/s 单方向音频负载**；双向都满速约 **64 KB/s**，不含 TLS/WS 头、心跳与控制消息。浏览器网络和后端算力成本不是 API 字符费。

### 2.2 实际 P6 线协议关键字节

```text
上行：0x01 | media_session_id | sequence | timestamp | pcm16
下行：0x02 | generation_id | call_id | sequence | timestamp | sample_rate | pcm16
控制：bind / unbind / heartbeat / media_state / audio_played /
     client_speech_detected / revoke_flush_acked / ...
```

真实报文还携带长度字段（例如 idLen `u16`、seq `u32`、时间戳 `f64`），详见上述冻结合同。客户端报「播放完成」是 **PlaybackAck**，不等于文字已经写入 31 的 **PersistAck**。

---

## 3. 关键技术如何实现

### 3.1 采集 / 播放线程

1. `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true, autoGainControl: true } })` 获取麦克风。
2. `AudioWorkletProcessor` 对设备采样率（常见 48 kHz）做轻量降采样，凑满 **320 个 16k 样本**发一次 20 ms PCM16。
3. JS 网络层将上行报文送到 WS；Worklet 不直接判断是否结束通话，也不直接做云 API 请求。
4. 下行 PCM 投入播放 Worklet 的 ring buffer。客户端对 **新旧 generation / call_id** 做额外过滤，撤销时执行 flush；旧样本不能在 reconnect 后复活。
5. 把音频标记为 `PLAYED` 需要 Worklet 实际消费样本，**不是**服务端把帧写入 socket 就算播放。

> 工程注意：源版播放器含 **300 ms** 预缓冲、**200 ms** 恢复缓冲等参数，属于该版本在音质与 underrun 间的折中；它们会占用用户感知延迟，**不要把服务器的 `first_audio` 指标当作用户真的听到了声音**。

### 3.2 ASR / VAD / Turn 三者不合并

- **VAD**：只回答「当前有没有声音像人声」。它不负责解释字义，也不能单独宣告用户话轮结束。
- **ASR**：持续给 partial（临时字幕）与 final（较稳定的识别结果）。final 到达时序、上传在途数和结果更新时间都要保留。
- **Turn Policy**：综合证据决定 `WAIT` 或 `CANDIDATE`；只有 Broker 允许真正 `turn_committed`。

冻结实验配置示例：`min_silence_ms=500`、`final_settle_ms=250`、`pending_grace_ms=4000`。**这些是具体版本的刺激/实验参数，并非可直接复制的最优通用阈值**。用户说「我觉得……（停顿）还是那个声音好」时，续句守卫和未决上传可避免把一段语义错误拆成两轮。

### 3.3 文字流的“可念短语”切分

`SpeakableChunker` 消费模型 `delta`，只输出本轮新增且未撤销的文字；优先在 `。！？` 等强断句输出，其次在 `，、空格` 等弱断句输出，必要时按最大长度截断。每段带 `chunk_seq`，供 TTS 适配器启动增量任务。

**典型并行方式**：LLM 继续写第二段时，TTS 同时合成第一段；客户端同时播放已经到达的 PCM。极短分段会伤害自然度并增加 API 往返，分段过长又拉高首声延迟，所以阈值要用真人中文试听与首声指标 A/B 调整。

### 3.4 语音合成如何保持真正流式

本地 `LocalCloneTts` 的接口行为：

```text
startTask(generation_id, text, onChunk, onDone, onFailed)
  POST /speak { text, gid, rate:16000, framed:true, ... }
  ← x-lin-protocol: frames-v1
  ← A: PCM16 bytes, 可连续多帧
  ← E: JSON { samples }, 明确的正常结束
  ← X: 失败
cancel(): AbortController.abort() + POST /stop?gid=...
```

只有收齐 `E`、PCM 样本数匹配、没有残留帧，才可宣布该句合成完成。网络突然断开时，已经到达的音频不能被误认为「该完整话轮成功」。

**与 VoiceStudio 本次 mini 实验的差别**：VoiceStudio 当前试用的 `/v1/audio/speech` 路径是先拿到整段生成结果、再分块传送；它不等价于真正的增量模型推理。曾在隔离测试中出现首音频约 **4.28 秒**，不能把这一数字归因于模型本身或当作官方通用性能。VoiceStudio 仍适合离线试音 / QC / 声线研究。

### 3.5 打断与过期音频：系统最容易被低估的难点

```text
AI 正在播 → VAD/ASR/客户端观察到用户发声
          → SPEECH_DETECTED / BARGE_IN_CANDIDATE
          → 轻量 duck / hold，**尚不永久取消**
          → Broker 经声学持续性 + ASR + 回声等证据确认
          → CONFIRMED_BARGE_IN
          → revoke 旧 generation 的播放所有权
          → cancel 正在跑的 LLM/TTS；flush 客户端缓冲
          → 旧 generation 的迟到 PCM 经过 PlaybackGate 即 DROP
```

**状态机不变量**：

```text
admit(audio_unit) =
  generation_eligible(unit.generation_id)
  && playback_ownership == OWNED(unit.generation_id)
  && sequence_not_consumed(unit.sequence)
```

`generation_id` 是播放世代号，`call_session_id` / `media_session_id` 是另一层通话与媒体生命周期身份；不能混为一谈。`candidate` 阶段只是怀疑，不能提前永久撤销；`confirmed` 后要让上游异步迟到数据永久失效。断网重连、旧音频重放、挂断到新通话也是同一条规则。

---

## 4. 最适合朋友复现的 Adapter 边界

下面是**建议的最小 TypeScript 伪接口（教学用，不宣称与源码逐字一致）**：

```ts
interface VoiceFrame {
  callSessionId: string;
  mediaSessionId: string;
  generationId: string;
  sequence: number;
  sampleRate: 16000;
  pcm16: Uint8Array;
}

interface AsrAdapter {
  start(options: { sampleRate: number; language: string },
        emit: (event: { type: 'partial'|'final'; text: string; ts: number }) => void
  ): { sendPcm(pcm16: Uint8Array): void; stop(): Promise<void> };
}

interface TtsAdapter {
  synthesize(args: {
    text: string;
    voiceId: string;
    generationId: string;
    signal: AbortSignal;
    onChunk(pcm16: Uint8Array, sampleRate: number): void;
  }): Promise<{ samples: number }>;
}

interface ResidentAdapter {
  streamReply(turn: string, context: unknown, signal: AbortSignal):
    AsyncIterable<string>;
}

// 适配器可以换，但以下规则不允许放进第三方 SDK 里：
// TurnAuthority -> GenerationRegistry -> PlaybackOwnership -> PlaybackGate
```

适配器应显式处理：连接超时、任务取消、采样率转换、输出计量、供应商断流、限流、返回文本与最终播放状态；不要让供应商 SDK 持有应用的通话权威或长期记忆。

---

## 5. 哪些部分可以用 ElevenLabs（ELE）或其他 API 替换？

下表中的「可换」意味着换一个 **adapter**，不是零代码把某个 endpoint 填到 URL 字段就行。

| 位置 | 25 现有方向 | ELE 可替换什么 | 其他常见替代 | 是否保留自研控制 |
|---|---|---|---|---|
| 用户语音转文字 ASR | 本机 macOS Speech / 可换流式 ASR | **Scribe v2 Realtime**，WebSocket partial/final | Soniox `stt-rt-v5`、Deepgram Nova、其他实时 STT | **保留** Turn/Broker、证据来源与 ASR final 语义 |
| AI 回答生成 LLM | Resident API / Gateway 适配层 | ELE Agents / Speech Engine 可作为**另一个全托管产品方案**；**不是**通用文本大脑的无差别直替 | OpenAI、Anthropic、MiniMax、Qwen、Ollama / llama.cpp 本地模型 | **保留** persona、tool authority、memory 权限、流式文本合同 |
| AI 语音 TTS | 本地 Qwen3-TTS / `LOCAL_CLONE_SLOT` | **Flash v2.5 / Turbo 适合实时**；v4 Turbo 走**另一条实时 Text-to-Dialogue WebSocket**，不能拿 v4 ID 强塞旧 TTS WS | Cartesia Sonic、Soniox TTS、本地 Kokoro / Piper / Qwen3-TTS | **保留**切句、取消、音频世代、播放安全 |
| 克隆声线 / 配音质检 | VS + 本地参考声线 | ELE Voice Cloning、TTS、Voice Changer，可作为候选试音 | Cartesia、VoiceStudio、本地可合法克隆模型 | **保留**身份一致性、声音授权与 A/B Gate |
| 非文字声音分类 | SOM/声音观测旁路方案 | Scribe 的 audio tags **能部分补充**，不应当作完整声学状态判断器 | YAMNet/Core ML、本地分类器 | **保留**观察与心理推断隔离，默认可关闭 |
| Foley/环境声资产 | AudioScene / FoleyTrack | ELE Sound Effects **离线生成素材**，再本地预解码播放 | 其他合法授权音效素材库、自制录音 | **保留**场景事件、时序混音、限幅及撤销 |
| 摄像头/数字人视频 | 25 媒体 + 30 UI 集成候选 | ELE TTS **不是数字人视频服务** | Vivix、视频渲染/Avatar 服务 | **保留** 37 通话权威与独立视频轨开关 |
| 打断、话轮最终提交、通话所有权 | 25/37 状态机 | **不能当作单个 ELE API 平替**；全托管 Agents 会重做产品边界 | LiveKit/Pipecat 可以协助编排，但非天然符合 25 合同 | **必须保留或重新证明等价性** |

### 5.1 ELE TTS 的两个实时 API，最容易接错

- **Text-to-Speech WebSocket**：`wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input`。适合连续发送 LLM 生成的 `text`；低延迟候选可用 `eleven_flash_v2_5`。官方明确该端点**不支持 Eleven v3 / v4**。[S3][S4]
- **Text-to-Dialogue WebSocket**：实时 v3/v4 系列走单独的协议与流结构，不可照搬上面的字段和 voice 绑定方式。[S5]
- **正确桥接**：ELE 输出格式/采样率/声道 → 统一 decode/resample 成本地 PCM16；附上原 generation/sequence，统一进入 PlaybackGate；用户打断时取消 ELE WS / generation，并丢弃网络上晚到的残余音频。不要假设关闭 WS 就等于所有下行已经清零。
- **测量区分**：ELE 官网宣传约 75ms（Flash）、约 100ms（v4 Turbo）等**模型/服务指标不等于端到端首声**；实际包含网络 RTT、输入文本缓冲、解码、播放器预缓冲和中文声线影响。[S2][S3]

### 5.2 用一个厂商全包的代价

ELE `Speech Engine`、Agents，或 OpenAI Realtime Speech-to-Speech，理论上能把 ASR/LLM/TTS 合成一体，开发速度会快。但若目标是复制 25 体验，必须额外证明：

1. 是否能拿到 ASR partial、final、VAD/turn 证据供自定义 Broker 使用？
2. 能否精确 cancel 且回收排队输出，是否暴露音频与播放 generation？
3. 能否复用既有 Resident 身份、记忆权限和 37 呼叫控制？
4. 云端音频处理的数据保留/区域/许可是否满足要求？
5. 厂商自带的 turn taking 与最终写入会不会变成第二权威？

**因此全托管方案适合快速做独立 MVP，不建议无验收直接替换 25 的核心控制层。**

---

## 6. 官方公开价格核对（2026-10-11；美元）

> **不是某个账户的报价。** 价格、地区、税、套餐、并发限额、订阅/按量资格和促销经常变动，下单前打开厂商链接复核。以下是 2026-10-11 检索到的官方页面价格。[S1][S6][S7]

| 厂商 / 服务 | 官方公开价格 | 典型计量方式 | 注意 |
|---|---:|---|---|
| ElevenLabs TTS **Flash/Turbo** | **$0.04 / 1,000 字符** | 输入文字字符 | 当前 API 按量标价；克隆声线类型/特性需再核。 |
| ElevenLabs TTS **v4 Turbo** | **常规 $0.04 / 1,000 字符** | 输入文字字符 | 截至 **10 月 12 日限时促销 $0.011/1k**；v4 需对应的实时接口，不能混用。 |
| ElevenLabs TTS **v4** | **常规 $0.08 / 1,000 字符** | 输入文字字符 | 截至 **10 月 12 日促销 $0.022/1k**；不把促销价当长期预算。 |
| ElevenLabs TTS **v3 Conversational** | **$0.04 / 1,000 字符** | 输入文字字符 | 支持低延迟对话但不是老版 Flash WS。 |
| ElevenLabs STT **Scribe v2 Realtime** | **$0.39 / 传入音频小时** | 连续音频时长 | 长连接发全部麦克风声音时，AI 讲话/静默也可能占用计费流；特定功能可能加价。 |
| ElevenLabs **Speech Engine** | **$0.08 / 分钟** | Speech Engine 服务分钟 | **另类托管路径**，不能和逐项 ASR+TTS 不加区别地相加；LLM/其他费用须看具体产品条款。 |
| ElevenLabs **Sound Effects** | **$0.12 / 生成音频分钟** | 离线资产生成 | 建议生成后缓存素材，不在通话里按每声实时重新合成。 |
| Soniox **实时 STT v5** | **约 $0.12 / 音频小时** | 供应商按 tokens，$0.12/h 是其官方折合参考 | 会随语言、上下文、输出 token 等变化，不是恒定包价。 |
| Soniox **流式 TTS** | **约 $0.70 / 生成音频小时** | 供应商按 tokens | 声线、中文与许可需单独试听核对。 |
| OpenAI **GPT-4.1 mini 文本 LLM** | **$0.40 / 1M 输入 token，$1.60 / 1M 输出 token** | input/output token | 仅举例计算文本大脑；知己实际运行的 Resident/模型未必选此档。 |
| 本地 **Qwen3-TTS + 本地 ASR** | **$0 按次云 API 费** | 自有硬件推理 | 仍有设备采购、折旧、电费、运维和许可成本；不是「完全免费」。 |

**计费口径要分清**：TTS 常以**生成文字字符数**计费，STT 常以**发送音频时间/使用 token**计费，LLM 以**输入+输出 token**计费，托管 Agents 可能以会话分钟计费。不能统一用「通话分钟 × 一个单价」套所有组件。

---

## 7. 一小时通话实际怎么估算？

**统一的教学假设（不是已量测负载）**：

- 电话接通 **60 分钟**。为支持插话，STT 云通道持续收约 60 分钟的音频。
- AI 语音有效输出约 30 分钟，按 **6,000 个 TTS 计费字符**估算；真实汉字数、标点与字符计数请以账单为准。
- LLM：总输入 **80,000 tokens**、输出 **12,000 tokens**，示例选 OpenAI GPT-4.1 mini **非缓存输入价**。
- 未含 retry、clone premium、请求/连接最低收费、ASR 附加能力、视频 API、CDN、云服务器、流量、税、额外工具/搜索费。

LLM 示例：`80,000 / 1,000,000 × $0.40 + 12,000 / 1,000,000 × $1.60 = $0.0512`。

| 方案 | ASR | TTS | LLM（示例） | 1 h 合计 |
|---|---:|---:|---:|---:|
| **A 本地优先**：本地 ASR + Qwen3-TTS + 文本云大脑 | $0 | $0 | $0.0512 | **$0.0512** |
| **B ELE 只替换嘴**：本地 ASR + ELE Flash/Turbo | $0 | $0.2400 | $0.0512 | **$0.2912** |
| **C ELE 两端全云**：ELE Scribe Realtime + ELE Flash/Turbo | $0.3900 | $0.2400 | $0.0512 | **$0.6812** |
| **D 经济云 ASR**：Soniox 实时 ASR + ELE Flash/Turbo | 约 $0.1200 | $0.2400 | $0.0512 | **约 $0.4112** |
| **E 本地嘴 + 云耳朵**：Soniox 实时 ASR + Qwen3-TTS | 约 $0.1200 | $0 | $0.0512 | **约 $0.1712** |

**30 分钟等比例示例**（ASR 发 30 分钟音频、输出 3,000 字符、LLM 40k/6k tokens）：A `$0.0256`；B `$0.1456`；C `$0.3406`；D `≈$0.2056`；E `≈$0.0856`。

**月预算**：例如每月共通话 30 小时，若负载比例近似，可用 `每小时单价 × 30`：A ~$1.54、B ~$8.74、C ~$20.44、D ~$12.34、E ~$5.14。**这只是边际 API 用量**，不等于各厂商最终付款额：月订阅套餐、额度包、最低充值、音量溢出与实时延迟保障可能使实际账单不同。

### 7.1 常见费用坑

- **ElevenLabs 当前 v4 Turbo 72% 促销只到 2026-10-12**：若按促销 `$0.011/1k` 算，该例 1h TTS `$0.066`；促销后常规 `$0.24`，差别很大。[S1]
- **STT 可能按会话里的所有发送音频计费**，不是「用户实际开口的分钟数」；切断静音上传可能省钱，但会影响 barge-in/VAD 行为，需看供应商语义和实际账单。
- **LLM 是不受本表限制的变量**：不同模型价格差很大，长历史反复注入、工具调用、重试、推理模式可能显著放大输入输出用量。
- **TTS 取消不等于已经生成的字符必定退款**；预算应按**供应商已计费字符**而不是最终用户听到的字符结算。
- **克隆声线**：录音与相似声线须具备使用授权；高级克隆/更高并发/商业权利按当前套餐与法律限制核查。
- **运营费用**：生产 WebSocket 网关、TLS 证书、日志、监控、云服务器或 GPU、跨区网络、TURN/SFU（若未来用 WebRTC）均另算。

### 7.2 配方与成本公式（可给朋友替换单价）

```text
TTS_cost  = billable_characters / 1,000 * provider_price_per_1k_chars
STT_cost  = billed_audio_hours * provider_price_per_hour  (token制需按账单token替换)
LLM_cost  = input_tokens / 1e6 * input_price_per_1m
          + output_tokens / 1e6 * output_price_per_1m
Total_API = STT_cost + TTS_cost + LLM_cost + other_provider_fees
```

如果一个产品只替换了 TTS，**不要**又额外加收一个整合式 Speech Engine 的全通话分钟费，除非确实同时使用了两种独立服务。

---

## 8. 给朋友的三条可落地路线

### 路线一：最低成本、本地优先（最接近 25）

`Browser AudioWorklet → WS → 本机流式 ASR → TurnPolicy → 文本 LLM API / 本地 LLM → 本机 Qwen3-TTS → PlaybackGate → Speaker`。

- 优点：云语音成本低、能保留音色/隐私自主权；打断与长期身份掌控在自己手里。
- 难点：Mac/GPU 性能、模型常驻、抢占调度、设备音频驱动、本地 ASR 长会话稳定性。
- **建议**：先做好可打断的单路语音，再逐步添加情绪/空间声场；不要一开始把音效识别、Avatar 和全套 Memory 混在一起。

### 路线二：最快交付、ELE 帮你把声音做好

`Browser → WS → local/Eleven Scribe Realtime → TurnPolicy → LLM → ElevenLabs Flash v2.5 (TTS WS) → PCM 转换 → PlaybackGate → Speaker`。

- 优点：不用本地跑 TTS GPU，可以快速获得商用 TTS 声线候选。
- 难点：网络 RTT、流式文本聚合策略、套餐和字符用量；需核查高级克隆声线中文表现、取消语义。
- **建议**：Cloud TTS 使用真正 WS 流式入口，不要先等完整 WAV 再拆包；保留 25 的独立 gate。

### 路线三：快速展示全托管 Agent

`Browser / Telephony → ElevenLabs Speech Engine/Agents 或其他端到端 Realtime Agent → Provider Audio Out`。

- 优点：最少自建组件，验证产品对话体验快。
- 风险：自定义插话状态机、记忆权限、角色自主权、音频事实证据和用户数据流不一定与你原本架构一致。
- **建议**：可以独立做 Demo；不要认为它是 25 Broker 的可替换 TTS 模块。

---

## 9. 验收与性能：至少测这十件事

| 测试 | 最低检查点 |
|---|---|
| 首声延迟 | 分开测：ASR final、Turn commit、LLM 首 delta、TTS 第一 PCM、客户端真实 `playback_started`；给出 p50/p95。 |
| 续句 | 「我觉得……还是这样」不把自然停顿误判为新轮。 |
| 短应答 | 「嗯嗯」「我在」不应被强制认成需要插话或结束。 |
| 真打断 | 用户插话时旧 generation 立即 revoke；服务端和客户端都 drop 迟到音频。 |
| 回声 | 扬声器播 AI、自身麦克风拾音时不会无限自打断。 |
| 网络 | 丢包、断线、重连时同一通话的 stale id 被拒绝。 |
| 后端掉线 | ASR/TTS/LLM 一路超时不使旧音频复活，也不死循环重试。 |
| 声线 | 相同 voiceId/相同参考音色跨 chunk、跨取消保持一致。 |
| 费用 | 取消半句、重试和 fallback 后账单字符、音频分钟、tokens 可对账。 |
| 隐私 | 未经授权不上传原始麦克风/参考音频；挂断清理所有缓冲和临时凭据。 |

**建议性能目标（非已实现结果）**：旁挂声音增强不阻塞主链；目标额外首声延迟 P95 ≤50 ms；打断确认与播放停止耗时需按设备测试并单独报告；零 stale audio resurrection；任何云 API 故障可退回本地/安全停机。若 50ms 无法达到，先做独立旁挂而不是牺牲实时讲话体验。

---

## 10. 实际构建顺序 / 交付结构

```text
project/
  client/
    capture-worklet.js        # 采音20ms PCM
    playback-worklet.js       # PCM ring buffer / play ack / flush
    call-ui.js                # 只投影状态，不裁决 call 权威
  server/
    transport/ws.js           # frame-id, sequence, reconnect
    asr/adapter.js            # local / eleven_scribe / soniox
    turn/turn-policy.js       # WAIT/CANDIDATE
    broker/generation.js      # ownership, cancellation
    broker/playback-gate.js   # stale/duplicate fail closed
    resident/adapter.js       # streaming text LLM
    chunker.js                # speakable phrases
    tts/adapter.js            # local / eleven_flash / v4_ttd
    metrics.js                # latency/cost/late-drops
  tests/
    interrupt.spec.js
    echo.spec.js
    reconnect.spec.js
    cost-ledger.spec.js
```

**增量里程碑**：
1. `M1`：客户端向服务端送 20 ms 帧、原样回声，不丢序列，能关麦克风；
2. `M2`：接入流式 ASR 与基础 Turn Policy，处理自然停顿；
3. `M3`：接入流式 LLM + TTS，音频首段与文字后段并行；
4. `M4`：加入 generation_id / 取消 / PlaybackGate / 客户端 flush；
5. `M5`：模拟真实设备回声、断线、重连与长会话；
6. `M6`：成本计量、API 配额与超额安全回退；
7. `M7`：可选非语言听觉、Foley、空间音效和数字人视频，逐项隔离验收。

---

## 11. 官方参考来源与核对日期

以下链接用于**价格与接口**。代码架构来自本项目冻结源码；外部文档不等于 25 的验收证据。

- **[S1] ElevenLabs API 官方定价**（2026-10-11，含 v4 Turbo 10/12 限时促销）：https://elevenlabs.io/pricing/api
- **[S2] ElevenLabs TTS 延迟优化、真实 streaming / WebSocket 的区别**：https://elevenlabs.io/docs/eleven-api/guides/how-to/best-practices/latency-optimization
- **[S3] ElevenLabs 实时 TTS WebSocket 指南（Flash）**：https://elevenlabs.io/docs/eleven-api/guides/how-to/websockets/realtime-tts
- **[S4] ElevenLabs 流式输入 TTS WS 接口**：https://elevenlabs.io/docs/api-reference/text-to-speech/v-1-text-to-speech-voice-id-stream-input
- **[S5] ElevenLabs TTS WS 与 Text-to-Dialogue WS（v3/v4）对照**：https://elevenlabs.io/docs/eleven-api/guides/how-to/websockets/tts-vs-ttd-websockets
- **[S6] Soniox API 官方定价**（含 token-based 折合费率）：https://soniox.com/pricing
- **[S7] OpenAI GPT-4.1 mini 官方价格**：https://developers.openai.com/api/docs/models/gpt-4.1-mini
- **[S8] ElevenLabs Scribe v2 实时语音识别说明**：https://elevenlabs.io/docs/eleven-api/guides/how-to/speech-to-text/realtime/server-side-streaming
- **[S9] Cartesia 官方产品/套餐，替代 TTS 时需重新确认模型与费用**：https://www.cartesia.ai/pricing
- **[S10] Soniox 当前实时 ASR 模型及 endpoint 说明**：https://soniox.com/docs/stt/models

**最后声明**：本文的价格以公开官方页面的 **2026-10-11 时点**为基准，实际付款前必须确认具体 plan、voice clone 权限、额度、数据处理与计费事件；实验性的 Qwen3 本地音色或 ELE 声线并不自动获得声音素材/真人声纹的商业使用许可。
---

## 附录｜一线补充（临 Lin @ hail-gate，2026-10-11）

> 以下为本机施工与联调第一现场记录，与主文同一冻结源码核对。**事实等级同主文**：标「已验证」=在本机 NONLIVE 环境实跑过；「建议」=工程判断。

### A. 真机三件事（iOS Safari，照文档做也会死在这）

1. **安全上下文**：`getUserMedia` 在非 HTTPS/localhost 下不可用。`http://192.168.x.x` 局域网调试页在 iPhone 上**拿不到麦克风**——我们 mini 隔离实验明确标注「普通 LAN HTTP 非安全上下文，不能冒充真机音视频可测」。真机验证要么 HTTPS、要么 localhost 反向端口。**这是「照文档做完但手机上不出声」的第一死因。**
2. **AudioContext 用户手势解锁**：iOS 要求音频上下文在用户手势后 `resume`；页面加载即播放必哑。我们的通话面把第一次媒体初始化挂在真实点击路径上，不为「自动进入通话」妥协。
3. **后台与锁屏**：iOS 后台/锁屏对音频链路、WebSocket、播放队列的行为不可当作桌面等效；设计通话面时挂起/恢复都要走显式状态，不能假设「页面还在就还在通话」。

### B. 诚实失败语义（我们和别人拉开差距的不是管道是这条）

- 拿不到摄像头/麦克风权限时，通话面**如实显示无声音无画面 + 可重试**，不伪造 active。这条在 iPhone 验收返工里被独立判官单独验证过。
- 同理：**未接后端如实 OFFLINE/HOLD，不拿 demo fixture 冒充 live 成功**。我们的 copy-scan 甚至曾因 mock provider 的诚实自标「Mock Provider：本地固定回复」误伤过——**诚实标记被禁词扫描当成违规**，教训是禁词表要分清「UI 骨架文案」与「数据内容」。
- 建议给所有「看起来像成功」的状态留一个反向检查：这个状态的证据链是什么？答不出就不该亮绿灯。

### C. EchoGuard 为什么存在（barge-in 最容易抄错的一刀）

打断最难的不是「检测到用户发声」，而是**别把自己的播放当插话**：扬声器在放 AI、麦克风收回来的音量天然超阈值。仅靠音量/能量阈值 100% 自打断。我们的做法是**证据聚合**：扬声器侧回声特征 + ASR 结果 + 客户端 speech_detected 多源交叉才升级 CONFIRMED_BARGE_IN；「疑似」阶段只做轻量 duck，不做永久撤销。抄架构图容易，抄这层证据仲裁才是本体。

### D. 沙箱验证拓扑（我们怎么离线验收这条链）

四层独立进程，每层可单独健康探活，且**层间失败如实报错不降级伪装**：

```text
Call gateway sandbox（呼叫权威，端口示例）
      ↑↓  P6 bridge（媒体桥，端口示例）
Voice runtime + Transcript sink（各自独立服务）
```

- 沙箱呼叫发起 API 在沙箱里发一通真 call_session，auto-answer ~1.8s 后可跑到 `outgoing → active → ended` 全生命周期——今天刚在本机实测，返回真实 `call_session_id`，0 错误。
- **真坑预警**：桥活着 ≠ 栈活着。我们曾遇到 bridge 与 voice server 在、`/health` 200，但伴生网关进程已死——沙箱呼叫 API 返回 `{"error":"fetch failed"}`。健康探针要探「能干活」不是「能应答」。
- 同理，seam 代理 502 的根因是上游端口指错（默认端口 A vs 实际端口 B）；还有一个挂了 3 天的僵尸进程占着端口、TCP 接受后返回空响应——排查顺序建议：**先 curl 上游本体，再查 seam 映射，最后查进程死活**。

### E. 本周真踩过的坑（按「现象→根因→防法」）

| 现象 | 根因 | 防法 |
|---|---|---|
| 示例网关会话 API 502 | seam env 默认值和实际后端端口不一致 | 起服务先 curl 上游本体，别先怪代理 |
| 端口被占但服务不响应 | 3 天前僵尸进程 accept 后返回空 | 起新实例前先 `lsof -i:PORT` + 功能探活，不是只看 LISTEN |
| 通话发起 `fetch failed` | p6 桥和 voice 在、伴生网关死了 | 健康检查加「功能探针」不只「端口探针」 |
| Reaction 按钮承诺后 toast「还没开放」 | 文案按 `adapter.demo`（mock provider 算 true）而非真能力 | 能力文案一律按 capabilities 真值，不按环境标志 |
| 测试把诚实标注当违规 | 禁词扫描不区分 UI 骨架与数据内容 | 扫描要分「界面文案」与「内容文本」两层 |
| git clone 报 non-monotonic index | exFAT 卷上的 AppleDouble `._*` 文件 | clone 后 `find . -name '._*' -delete`，写文件走 shell 不走 apply_patch |

**最后一句前线心得**：这套系统里最难写的从来不是管道，是**每一层的诚实**——音频别假播放、状态别假成功、测试别假绿。真做到这三条，剩下的都是体力活。
