# LLM-RAG-Voice-System

离线语音交互系统，基于 RK3576 NPU 将 **实时语音识别 (ASR)**、**本地大模型推理 (LLM/RAG)** 与 **语音合成 (TTS)** 串联起来，通过 ZeroMQ 构建可替换、可扩展的模块化流水线。整体系统在嵌入式设备上即可完成“语音输入 → 知识检索/推理 → 语音回复”的闭环交互。

![System Architecture](docs/image.png)

## 功能亮点

- **全离线部署**：所有模型均可在端侧运行，不依赖云端服务或外部网络。
- **松耦合架构**：ASR、LLM、TTS 之间通过 ZeroMQ 消息交互，可按需更换实现或部署在不同进程/设备上。
- **低延迟优化**：ASR 采用流式识别，TTS 通过双缓冲队列与播放线程解耦，在 4 秒内完成端到端交互。
- **可观测性**：日志打印展示模块间的消息流，便于调试与性能分析。

## 目录结构

```
├── docs/                     # 架构图等文档
├── llm/                      # 大模型（RKLLM）部署与示例
│   ├── models/               # 模型说明占位符
│   ├── rknn-llm/             # RKLLM Runtime 与官方示例
│   └── test/                 # ZeroMQ LLM Demo（对接 ASR/TTS）
├── tts/                      # SummerTTS 推理引擎与服务端
│   ├── src/                  # TTS 推理核心（纯 C++/Eigen）
│   └── tts_server/           # ZeroMQ 服务端、播放线程与工具
├── voice/                    # 基于 sherpa-onnx 的流式 ASR
│   ├── models/               # 预训练 ASR 模型存放路径
│   └── sherpa-onnx/          # 第三方 ASR 引擎源码（含改动）
├── zmq-comm-kit/             # ZeroMQ 封装库（REQ/REP 客户端与服务端）
└── LICENSE.txt
```

## 模块说明

### 1. 语音识别（voice/）
- 基于 [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) 的流式 Zipformer 模型，使用 `sherpa-onnx-microphone` 进行实时录音与识别。
- `voice/sherpa-onnx/voice/sherpa-onnx-microphone.cc` 中嵌入 ZeroMQ 客户端：
  - 将识别文本通过 `ZmqClient` 发送到 LLM 服务端（默认 `tcp://localhost:6666`）。
  - 在播放完成前阻塞新的音频写入（`tcp://localhost:6677`），避免语音覆盖。
- 支持通过环境变量选择麦克风设备、采样率，详见 sherpa-onnx 自带说明。

### 2. 大模型服务（llm/）
- `rknn-llm/` 提供 RKNN 官方 Runtime、模型转化脚本及示例，可在 RK3576 上执行 DeepSeek 等大模型推理。
- `llm/test/llm_test.cpp` 给出最小化 LLM 服务：
  - 使用 `ZmqServer` 接收 ASR 文本请求并回执确认。
  - 通过 `std::wregex` 将长文本切分成可合成片段，逐段发送给 TTS 服务端 (`tcp://localhost:7777`)。
  - 真实部署时，可将 `message_worker` 替换为 RKLLM 推理与 RAG 检索逻辑，仅需保持 ZeroMQ 接口一致。

### 3. 语音合成与播放（tts/）
- 底层使用 SummerTTS（纯 C++/Eigen 实现，无外部 NN 依赖），模型需单独下载并放入 `tts/models/`，详见 `tts/README.md`。
- `tts/tts_server` 实现完整的服务端逻辑：
  - `DoubleMessageQueue` 提供文本、音频双队列，支持并发写入与逐段播放。
  - `TTSModel` 包装 SummerTTS 推理接口，完成模型加载、推理与资源释放。
  - `AudioPlayer` 基于 ALSA 播放 PCM 数据，可按需调整播放采样率。
  - `main.cpp` 负责 ZeroMQ 服务端、推理线程与播放线程的编排，实现双缓冲解耦与播放结束通知。

### 4. ZeroMQ 通信封装（zmq-comm-kit/）
- 提供 `ZmqServer`（REP）与 `ZmqClient`（REQ）两个轻量封装，简化发送/接收与超时配置。
- 编译后生成 `libzmq_component.so` 供 ASR/LLM/TTS 等模块复用。

## 环境依赖

- CMake ≥ 3.12、GCC/G++ ≥ 9 或兼容 C++17 的交叉编译工具链
- ZeroMQ 与 `libzmq` 开发包
- ALSA (`libasound2-dev`) 与 PortAudio（供麦克风采集/播放）
- pthread / OpenMP（SummerTTS 依赖）
- RKNN SDK（部署 RK3576 LLM 时需要，路径通过 `RKLLM_API_PATH` 指定）

## 模型准备

| 模块 | 资源位置 | 说明 |
| ---- | -------- | ---- |
| ASR  | `voice/models/` | 放置 sherpa-onnx 官方预训练模型，例如 `sherpa-onnx-streaming-zipformer-small-bilingual-zh-en-2023-02-16`（已提供）。
| LLM  | `llm/rknn-llm/res/` | 使用 RKNN 工具将 DeepSeek 等模型量化/切分，具体流程参考 `llm/rknn-llm/doc/`。
| TTS  | `tts/models/` | 从 SummerTTS README 中提供的网盘下载模型，例如 `single_speaker_fast.bin`、`single_speaker_english.bin` 等。

## 编译步骤

1. **构建 ZeroMQ 组件库**
   ```bash
   cd zmq-comm-kit
   mkdir -p build && cd build
   cmake ..
   make -j$(nproc)
   sudo make install   # 或者将生成的 libzmq_component.so 手动拷贝到 /usr/local/lib
   sudo ldconfig
   ```

2. **编译 TTS 服务端**
   ```bash
   cd ../../tts
   mkdir -p build && cd build
   cmake ..
   make -j$(nproc)
   ```
   生成的 `tts_server` 可执行文件位于 `tts/build/`，运行时需要指定模型路径。

3. **编译 LLM Demo 或自定义服务**
   ```bash
   cd ../llm/test
   mkdir -p build && cd build
   cmake -DRKLLM_API_PATH=/path/to/rkllm ..
   make -j$(nproc)
   ```
   如使用 RKNN 推理，请根据 `rknn-llm` 官方指导链接 RKNN runtime 与模型。

4. **编译 ASR 客户端**

   sherpa-onnx 提供多种构建方式（CMake、Python）。在嵌入式部署中，可参考官方文档或使用项目中修改过的源码编译 `sherpa-onnx-microphone`。示例：
   ```bash
   cd ../../voice/sherpa-onnx
   mkdir -p build && cd build
   cmake .. -DSHERPA_ONNX_ENABLE_PORTAUDIO=ON
   make -j$(nproc)
   ```

## 运行流程

1. **启动 TTS 服务端**（提供语音合成与播放）：
   ```bash
   ./tts/build/tts_server ../models/single_speaker_fast.bin
   ```
   - 默认监听 `tcp://*:7777`（接收文本）与 `tcp://*:6677`（播放状态）。

2. **启动 LLM 服务**：
   ```bash
   ./llm/test/build/llm_test
   ```
   - 监听 `tcp://*:6666`，接收 ASR 文本并转发给 TTS。
   - 可替换为接入 RKLLM 推理的自定义程序，只需保持 ZeroMQ 协议一致。

3. **启动 ASR 录音客户端**：
   ```bash
   ./voice/sherpa-onnx/build/bin/sherpa-onnx-microphone \
     --tokens=../models/sherpa-onnx-streaming-zipformer-small-bilingual-zh-en-2023-02-16/tokens.txt \
     --encoder=../models/.../encoder-epoch-12-avg-4.onnx \
     --decoder=../models/.../decoder-epoch-12-avg-4.onnx \
     --joiner=../models/.../joiner-epoch-12-avg-4.onnx
   ```
   - 语音识别结果会实时发送给 LLM，收到播放结束通知后继续下一段识别。

4. **联调建议**
   - 逐个启动模块，观察日志 `[voice -> llm]`、`[llm -> tts]`、`[tts -> voice]` 等前缀以确认通信正常。
   - 如需跨设备部署，可在各模块的构造函数中修改 ZeroMQ 地址（例如将 `tcp://localhost` 改成指定 IP）。

## 自定义与扩展

- **替换模型**：只需更换对应目录下的模型文件，并在启动命令中指定新的路径。
- **集成 RAG**：在 `message_worker` 内添加知识检索与上下文拼接逻辑，再将推理结果分片发送到 TTS。
- **播放策略**：`AudioPlayer::play` 支持自定义播放速率，可扩展为输出至 I2S、蓝牙等音频设备。
- **线程优先级**：`utils::set_realtime_priority` 可用于提升 TTS 推理线程优先级，在实时场景下减少卡顿。

## 常见问题

1. **ZeroMQ 超时**：如网络异常，`ZmqClient/ZmqServer` 会抛出 `ZmqCommunicationError`，可在外层捕获并实现重试。
2. **音频设备不可用**：确保 ALSA/PortAudio 已正确安装，并为运行用户授予访问权限。必要时通过 `SHERPA_ONNX_MIC_DEVICE` 指定麦克风。 
3. **模型加载失败**：确认模型文件路径正确、权限足够，且与编译时的量化参数匹配。

## 许可证

本项目整体遵循仓库根目录的 `LICENSE.txt`。各子模块保留原仓库的许可证（如 sherpa-onnx、SummerTTS、RKNN Runtime 等），在商用或二次分发前请逐一确认其许可要求。

