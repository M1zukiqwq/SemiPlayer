# seek 命令的编排设计

> 属于 ApiLayer 模块。描述关键帧 seek、精准 seek 的公开语义和内部编排。
> 总体世代号原则见 `docs/architecture.md`。

## 当前能力

调用方必须明确选择模式：

| 模式 | 语义 |
|------|------|
| `PreviousKeyframe` | 定位到目标 PTS 之前或恰好位于目标 PTS 的最近视频关键帧 |
| `NextKeyframe` | 定位到目标 PTS 之后或恰好位于目标 PTS 的最近视频关键帧 |
| `Accurate` | 定位到目标左侧最近关键帧，解码后仅暴露目标 PTS 及之后的音视频 |

前两种快速模式不解码并丢弃关键帧与目标 PTS 之间的数据，因此开销主要是容器定位和
下游 generation 切换。`Accurate` 会从前一关键帧解码到目标，长 GOP 下延迟更高，但落点准确。

`NextKeyframe` 在目标之后不存在关键帧时可以返回定位失败；实现不会静默回退到前一个
关键帧，因为那会违反调用方选择的方向。

## C ABI

```c
typedef uint32_t semi_seek_mode_t;

enum {
    SEMI_SEEK_MODE_PREVIOUS_KEYFRAME = 1,
    SEMI_SEEK_MODE_NEXT_KEYFRAME = 2,
    SEMI_SEEK_MODE_ACCURATE = 3
};

semi_handle_t semi_player_seek(int64_t position_us, semi_seek_mode_t mode);
```

负位置或未知模式通过有效 handle 完成为 `SEMI_ERR_INVALID_ARGUMENT`。定位失败映射为
`SEMI_ERR_INVALID_RESOURCE`。

## 编排

```text
ApiLayer::SeekCommand { position_us, mode }
    ↓
Demuxer::seek(position_us, mode)
    ↓
FfmpegDemuxerBackend::seek(position_us, mode)
    ↓
定位成功后 generation + 1
    ↓
各下游 worker 观察新 generation，丢弃旧数据并 reset backend
```

FFmpeg backend 优先使用主视频流作为 seek 参考流，把微秒目标转换到视频流 time base，
再调用 `av_seek_frame`：

- `PreviousKeyframe` 使用 `AVSEEK_FLAG_BACKWARD`；
- `NextKeyframe` 不使用 `AVSEEK_FLAG_BACKWARD`；
- `Accurate` 与 `PreviousKeyframe` 一样定位到左侧关键帧；
- 三种模式都不使用 `AVSEEK_FLAG_ANY`，因此 demux 定位目标保持为关键帧。

没有视频流时回退到主音频流进行方向性时间定位。

## generation 顺序

generation 只能在 backend 定位成功后推进：

```text
backend seek 成功 → generation.bump(optional target PTS) → 清除 demux pending 状态 → 读取新数据
```

如果定位失败，generation 和当前会话保持不变。旧队列数据由 generation 检查自然失效；
精准模式的 target PTS 与新 generation 原子关联，普通模式不携带 target。这样下游即使错过
`GenerationChanged` 通知，也能通过 generation 快照恢复正确门限。

## 精准模式的数据门限

- `VideoDecoder` 丢弃无 PTS 或 `PTS < target` 的帧，首个 `PTS >= target` 的帧打开门限；
- `AudioDecoder` 丢弃完全位于目标前的 PCM，对跨过目标的首帧按采样点向上取整并裁掉前缀；
- `AudioResampler` 从首个输入 PTS 建立输出游标，后续 PTS 按实际输出 sample 数推进，包含 drain 输出；
- generation 变化会清空门限状态并 reset decoder/resampler backend。

## SDL example 策略

- 向左跳转使用 `PreviousKeyframe`；
- 向右跳转使用 `NextKeyframe`。

这样向右跳转不会因为 FFmpeg 回退到当前 GOP 起点而产生视觉上的反向跳转。
