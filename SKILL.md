---
name: auto-remotion
description: |
  从已有录屏/产品演示视频生成官网宣传片的工作流。

  当用户提到以下场景时触发：
  - "把录屏转成宣传片"、"用录屏做产品视频"
  - "把演示视频做成官网介绍"
  - "Remotion 切片"、"视频分镜"
  - "产品宣传视频生成"、"screen recording to promo video"
  - 用户想用 Remotion 把长视频切成短片段做宣传片

  本技能覆盖从原始录屏素材到完整 Remotion 宣传片的完整流程：
  目标确认 → 素材识别 → 分镜策划 → 结构化规格 → Remotion 实现
  → 字幕轨 → 中文配音（edge-tts）→ BGM → 渲染出片

  每个阶段都有具体检查清单、常见问题和决策框架。
---

## 核心理念

这条工作流的核心不是"让 AI 替代人工"，而是：

> **把人工判断和自动化工具各自放在最合适的位置。**
> 人工判断哪些画面有价值，工具负责把价值组合成视频。

**执行顺序很重要**：
1. 先锁定素材区间
2. 再锁定段落结构
3. 再锁定总时间线
4. 最后才叠加字幕、配音、BGM、包装

违反这个顺序，会导致大量返工。

---

## 阶段一：明确目标与约束

在动手之前，先对齐：

1. **输入是什么**：录屏、直播录屏、剪辑素材包，还是多段演示视频？
2. **输出是什么**：官网宣传片、产品介绍、销售演示，还是社媒短视频？
3. **时长目标**：严格 60 秒、可浮动到 70 秒，还是优先完整表达？
4. **核心价值**：产品能力、用户体验、结果展示，还是品牌感？
5. **哪些后置**：字幕、配音、BGM 放到第二阶段？

如果约束不先说清楚，后面会在"要不要保留完整结果""能不能接受更长"这类问题上反复拉扯。

---

## 阶段二：建立结构化中间产物

不要直接写代码。先建立以下文件：

### 2.1 剪辑执行稿（`.md`）

按 Segment 分解，每段包含：
- 目标 / 画面描述 / 屏幕文案 / 旁白 / Remotion 对接建议
- 这是讨论和审阅的基础

### 2.2 分镜表（`storyboard.json`）

```json
{
  "compositionId": "MyPromoV1",
  "fps": 30,
  "durationInFrames": 1800,
  "canvas": "1920x1080",
  "segments": [
    {
      "segmentId": "SegmentIntro",
      "segmentIndex": 0,
      "startFrame": 0,
      "durationFrames": 150,
      "goal": "片头引入，30字内概括价值",
      "text": { "eyebrow": "", "title": "", "body": "" },
      "narration": "产品名，让 AI 完成真实任务。",
      "clip": null,
      "overlay": "top-bar"
    }
  ]
}
```

### 2.3 编辑规格（`edit-spec.json`）

帧级时间线，包含真实素材区间和速度：

```json
{
  "compositionId": "MyPromoV1",
  "fps": 30,
  "durationInFrames": 1800,
  "sourceVideo": { "path": "./public/source.mp4" },
  "segments": [
    {
      "segmentId": "SegmentIntro",
      "startFrame": 0,
      "durationFrames": 150,
      "clips": []
    }
  ]
}
```

---

## 阶段三：识别母视频时间点

这一步是整个链路的核心桥梁。

**不要跳过**。即使有 AI 视频理解能力，在 MVP 阶段人工识别仍然最快。

推荐做法：
1. 用户（或你引导用户）观看原视频，给出粗粒度内容分区
2. 在粗粒度分区中细化为可剪辑时间点
3. 区分"价值展示片段"和"过程等待片段"
4. 用结构化表格记录所有时间点

**关键原则**：
- 对话类片段要确保"输入→发送→回复出现"的因果链完整
- 长任务演示不要只靠整体快进，要拆成"发起/执行/结果/成果展示"多个叙事节点
- 真实切片时间点以**分**为单位记录（`4:16` 而不是 `4.267`）

---

## 阶段四：Remotion 骨架搭建

### 4.1 项目结构约定

```
remotion-app/
├── public/              ← 所有源素材放这里（视频、音频、图片）
│   ├── source.mp4      ← 源视频
│   ├── voiceover.mp3    ← 配音
│   └── bgm.mp3          ← BGM
├── src/
│   ├── Composition.tsx  ← 主 composition（含所有 Segment 组件）
│   └── Root.tsx        ← composition 注册
└── package.json
```

**所有源素材必须放 `public/`，用 `staticFile("文件名")` 引用。**

### 4.2 视频切片工具函数

```tsx
type ClipSpec = {
  trimBeforeFrames: number;
  trimAfterFrames: number;
  playbackRate: number;
};

const clip = (
  startSeconds: number,
  endSeconds: number,
  playbackRate = 1
): ClipSpec => ({
  trimBeforeFrames: Math.round(startSeconds * 30),
  trimAfterFrames: Math.round(endSeconds * 30),
  playbackRate,
});

// 高倍速片段务必 muted
<Video
  src={videoSrc}
  muted                      // ← 超过 10x 强烈建议 muted
  trimBefore={trimBeforeFrames}
  trimAfter={trimAfterFrames}
  playbackRate={playbackRate}
  style={{ width: "100%", height: "100%", objectFit: "cover" }}
/>
```

### 4.3 Sequence 嵌套与时长管理

**父子 Sequence 时长是关键原则**：

> 外层 Sequence 的 `durationInFrames` 必须足够容纳所有子片段的总时长。

如果子片段时长总和超过父级，会发生截断（子片段被砍掉尾部）。

```tsx
// 正确示例
<Sequence from={840} durationInFrames={606}>  {/* ← 必须 ≥ 所有子片段之和 */}
  <SegmentModules />
</Sequence>
```

### 4.4 TypeScript 踩坑清单

| 错误写法 | 正确写法 |
|---------|---------|
| `<AbsoluteFill pointerEvents="none">` | `<AbsoluteFill style={{ pointerEvents: "none" }}>` |
| `<Video style={{ objectFit: "cover" }}>` | `<Video objectFit="cover">` |
| `playbackRate={0}` | 不支持，改为纯色背景 |
| `import JSON from "./data.json"` | 内联为 TypeScript 常量数组 |

---

## 阶段五：字幕轨接入

### 5.1 字幕数据结构

字幕 cue 用**左闭右开**区间 `[startFrame, endFrame)`：

```tsx
type SubtitleCue = {
  cueId: string;
  segmentId: string;
  startFrame: number;  // 包含
  endFrame: number;    // 不包含
  text: string;
};

const activeCue = subtitleCues.find(
  (cue) => frame >= cue.startFrame && frame < cue.endFrame
);
```

### 5.2 渲染组件

```tsx
const SubtitleTrack: React.FC = () => {
  const frame = useCurrentFrame();
  const activeCue = subtitleCues.find(
    (cue) => frame >= cue.startFrame && frame < cue.endFrame
  );
  if (!activeCue) return null;

  return (
    <AbsoluteFill style={{ pointerEvents: "none" }}>
      <div style={{
        position: "absolute", left: 140, right: 140, bottom: 42,
        display: "flex", justifyContent: "center",
      }}>
        <div style={{
          maxWidth: 1080, padding: "16px 24px", borderRadius: 22,
          background: "rgba(5, 8, 22, 0.72)", color: "white",
          fontSize: 28, textAlign: "center",
        }}>
          {activeCue.text}
        </div>
      </div>
    </AbsoluteFill>
  );
};
```

---

## 阶段六：中文配音（edge-tts）

### 6.1 流程

```
voiceover-script.json（结构化旁白文本）
    → edge-tts 生成各段 MP3
    → ffmpeg 合并（每段后加静音填充对齐目标帧时长）
    → 放入 public/
    → <Audio src={staticFile("voiceover.mp3")} /> 接入 Composition
```

### 6.2 静音填充是关键

**不能假设配音自然时长等于目标帧时长。**

edge-tts 按自然语速生成，每句实际时长和目标帧时长必然有偏差（可能 ±0.5-2 秒）。直接合并会导致：
- 配音总时长比视频短
- 后半段 Segment 没有配音
- 字幕和配音完全错位

**正确做法**：每段配音后测量实际时长，用 `anullsrc` 生成静音填充到目标秒数：

```bash
# 生成静音
ffmpeg -f lavfi -i anullsrc=r=24000:cl=mono -t 2.5 -q:a 9 silence.mp3
# 合并
ffmpeg -f concat -safe 0 -i concat_list.txt -acodec libmp3lame output.mp3
```

### 6.3 中文语音选项

| Voice | 特点 | 适用场景 |
|-------|---|---|
| `zh-CN-XiaoxiaoNeural` | 专业女声，清晰流畅 | 产品介绍（本次选用） |
| `zh-CN-YunxiNeural` | 年轻男声，有点活泼 | 科技产品演示 |
| `zh-CN-YunjianNeural` | 阳刚男声 | 强技术感、专业工具 |

### 6.4 配音与时间线匹配策略

微调顺序：
1. 先调文字内容长度
2. 次调语速（`rate` 参数）
3. 最后才改 Segment 时长（因为改时长会影响所有子片段基准）

---

## 阶段七：BGM 接入

### 7.1 来源

Pixabay Music（商用免费，无需署名）
关键词：`"light technology"`、`"corporate tech"`

### 7.2 动态音量接入

```tsx
<Audio
  src={staticFile("bgm.mp3")}
  volume={(frame) => {
    const t = frame / 30;
    if (t < 3) return (t / 3) * 0.4;           // 淡入
    if (t > 63.2) return ((66.2 - t) / 3) * 0.4; // 淡出
    return 0.2;                                      // 配音密集段
  }}
/>
```

### 7.3 音量原则

**配音清晰度优先**：BGM 必须作为背景，不能盖过人声。

- 配音密集段：0.15-0.25
- 无配音段（片头/片尾）：0.3-0.5
- 本次实践最终值：峰值 0.4，正常段 0.2

---

## 阶段八：渲染出片

### 8.1 渲染命令

```bash
npx remotion render MyCompositionV1 --fps=30 --frames=0-{N-1} --output="output.mp4"
```

**帧范围是左闭右开 `[0, N)`**，总帧数 N 意味着最后一帧是 N-1。

### 8.2 磁盘空间要求

Remotion 渲染需要：
- webpack bundle 缓存（~400MB）
- Chromium headless（~100MB）
- 临时帧文件

渲染前确保 **C 盘有 5-10GB 可用空间**。`%TEMP%` 中的 `remotion-webpack-bundle-*` 可提前清理。

### 8.3 Studio 预览 ≠ 最终渲染

- Studio 预览受浏览器解码限制，高倍速（>10x）可能卡顿
- 最终渲染用 ffmpeg 硬解码，流畅得多
- **Studio 只做节奏判断，最终质量以渲染出片为准**

---

## 关键设计决策框架

### Option A vs Option B 决策

当某个 Segment 时长问题无法通过调速解决时：

| 选项 | 做法 | 适用条件 |
|-----|------|---------|
| Option A（压缩） | 强制缩短时长，裁剪内容 | 甲方严格卡时长 |
| Option B（接受更长） | 让叙事完整，接受更长总时长 | 优先内容完整 |

**决策原则**：官网宣传片的核心目标是"把产品价值讲清楚"，而非"严格卡 N 秒"。压缩到不完整，反而失去意义。

### 冻结帧实现

不要用 `Img src="video.mp4#t=1.0"`（依赖 HTTP 服务器）或 `playbackRate={0}`（不支持）。改用纯色背景或预渲染图片。

---

## 项目结构总览

```
project/
├── work/                          ← 工作流与资产层
│   └── projects/{project-id}/
│       ├── edit-script.md          ← 剪辑执行稿
│       ├── storyboard.json         ← 分镜表
│       ├── edit-spec.json         ← 编辑规格（帧级时间线）
│       ├── subtitle-track.json      ← 字幕时间轴
│       ├── voiceover-script.json   ← 配音脚本
│       ├── generate_voiceover.py  ← 配音生成脚本
│       └── process_bgm.py         ← BGM 混合脚本
└── remotion-app/                  ← Remotion 实现层
    ├── public/
    │   ├── source.mp4            ← 源视频
    │   ├── voiceover.mp3         ← 配音
    │   └── bgm.mp3               ← BGM
    └── src/
        ├── Composition.tsx        ← 主 composition
        └── Root.tsx              ← 注册
```

---

## 常见问题速查

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 子片段被截断 | 父级 Sequence durationInFrames 不够 | 检查并扩大父级时长 |
| Studio 预览高倍速片段卡顿 | 浏览器解码压力大 | 设置 `muted: true`，Studio 只判断节奏 |
| 配音和字幕错位 | 配音总时长 ≠ 视频时长 | 每段后加静音填充对齐 |
| 渲染报 ENOSPC | C 盘空间不足 | 清理 %TEMP%，确保 5-10GB 可用 |
| 渲染时 FreezeFrame 报错 | `#t=` 或 `playbackRate=0` 不支持 | 改用纯色背景 |
