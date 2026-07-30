---
name: video-to-shots
description: "解析本地视频（抽帧 + 逐帧视觉识别 + 字幕 OCR + 音频探测），产出两份文件：shots.json（机器契约，schema 对齐 ai-video-generation / HappyHorse 的 API body，供下游零转换消费）与 report.md（人类留档的解析报告 + 后期清单）。触发词：解析视频、视频拆解、视频转提示词、把视频变成提示词、视频转分镜、生成 shots.json、为视频生成可重制提示词、video to shots、video parse、deconstruct video"
allowed-tools: Read, Write, Glob, Bash(*)
---

# 视频解析 → shots.json + report.md

把一支本地视频拆成「可被 `ai-video-generation` 直接重制」的结构化清单。**一次运行产出两个文件**：

- `shots.json` —— **机器契约**，下游 `ai-video-generation` **唯一**消费的文件。
- `report.md` —— **人类留档**，解析报告 + 适配/后期说明，下游**不读**。

> 本 skill 只负责“解析 + 写清单”，**不**调用视频生成接口、**不**上传图床、**不**做后期拼接/叠字幕。这些是下游/后期的事，本 skill 只把它们**结构化地声明**进 json，供后续环节读取。

## 遵循的运行约束

- **不派生子代理**（不用 Agent 工具）。逐帧识别在主会话用 `Read` 循环完成，必要时在同一回复内并行多个 `Read`。
- **`Read` 图片一律用绝对路径**。工作目录未必等于文件所在目录，用相对路径会“文件不存在”。
- 全程中文输出注释与文档。

---

## 与 ai-video-generation 的契约（最重要的一节）

`ai-video-generation`（HappyHorse 1.1）**不读任何文件**——是“运行它的那个 Claude”读文件、拼 curl；其接口是**单任务**接口（一次 POST = 一个 clip）。所以本 skill 的 `shots.json` 设计目标 = **让消费端零改写、可 for-loop 地遍历出一次次 API 调用**。

**契约规则（两个 skill 的耦合面，钉死不改）：**

1. `shots[].body` 的字段名**必须与 HappyHorse API body 完全一致**：`model` / `input.prompt` / `input.media[].type` / `input.media[].url` / `parameters.resolution` / `parameters.ratio` / `parameters.duration`。**不要**自创 `text/seconds/quality` 等别名——这是“零转换”的命门，消费端可 `jq -c '.shots[].body'` 逐条 `curl -d @-`。
2. 图床 URL 是运行时外部依赖，**本 skill 不上传**。`shots[].url` 先写 `null`、`url_status:"pending"`，并把本地首帧路径放在 `first_frame_local` 供下游/用户去传。消费端按 `url_status` 选 i2v 或退回 t2v。
3. `post_process` 与 `captions` **不参与生成**，留给后期（叠字幕脚本/剪辑脚本可直接遍历）。
4. 实际写盘的 `shots.json` **必须是合法 JSON（无注释）**。下文示例用 jsonc 仅为可读。

---

## 输出文件 1：shots.json（schema v1）

```jsonc
{
  "version": 1,
  "meta": {
    "source_file": "原始视频文件名或路径",
    "duration_sec": 27.8,
    "resolution": "1038x720",
    "fps": 30,
    "has_audio": true,
    "frame_interval_sec": 2,                 // 抽帧间隔，默认2
    "parsed_by": "ffmpeg 抽帧 + 逐帧视觉识别 + 字幕OCR + volumedetect",
    "style_prefix": "整片统一风格前缀，会被拼到每个 shot 的 prompt 前",
    "aspect_decision": "源画幅说明 + 生成建议：16:9后加黑边 或 9:16填满",
    "reference_note": "疑似参考作品/氛围说明（可选，可空串）",
    "audio_note": "音轨性质 + 与HappyHorse自动音的关系 + 是否需后期合原声"
  },
  "shots": [
    {
      "id": "shot_01",
      "timecode": "00:00-00:02",
      "first_frame_local": "video_frames/f_001.jpg",   // 本地，i2v 上传前参考
      "url": null,                                     // 传图床后回填公网URL
      "url_status": "pending",                         // pending|uploaded|skipped
      "body": {                                        // 字段名=API body，可直接POST
        "model": "happyhorse-1.1-i2v",                 // url_status=skipped时改t2v并删media、补ratio
        "input": {
          "prompt": "<style_prefix> + 本镜单一画面与运动描述",
          "media": [ { "type": "first_frame", "url": "<=回填.url>" } ]
        },
        "parameters": { "resolution": "720P", "duration": 3 }   // i2v无ratio；t2v需补ratio
      },
      "captions": [ { "start": 0.0, "end": 2.0, "text": "本镜出现的字幕" } ]
    }
  ],
  "post_process": {
    "letterbox": "上下加黑边，目标宽银幕2.35:1（不需要则空串）",
    "concat_order": ["shot_01", "shot_02"],
    "original_audio": "原视频文件名（用于后期合原声；不需要则空串）",
    "caption_style": "右下，粉色，繁体，宽字距，描边发光（按实际）",
    "captions_full": [ { "start": 0.0, "end": 2.0, "text": "全量字幕带时间码" } ]
  }
}
```

**字段说明：**

| 字段 | 必填 | 说明 |
|---|---|---|
| `version` | 是 | schema 版本，当前 `1` |
| `meta.style_prefix` | 是 | 全片风格词，拼到每镜 prompt 前，避免重复 |
| `shots[].id` | 是 | `shot_01` 递增，与 `concat_order` 对应 |
| `shots[].timecode` | 是 | `MM:SS-MM:SS`，由帧索引×间隔换算 |
| `shots[].first_frame_local` | 是 | 该镜首帧本地相对/绝对路径 |
| `shots[].url` / `url_status` | 是 | 默认 `null` / `pending` |
| `shots[].body` | 是 | **可直接 POST 的 HappyHorse body**，字段名见契约 |
| `shots[].body.parameters.duration` | 是 | 整数，`clamp(镜头秒数, 3, 15)` |
| `shots[].body.parameters.ratio` | t2v 必填 | i2v 不写（由首帧比例决定） |
| `shots[].captions` | 是 | 该镜时间窗内的字幕子集，可空数组 |
| `post_process.captions_full` | 是 | 全量字幕 + `start/end`，供后期叠字幕 |

**model 默认策略**：有首帧 → 默认 `happyhorse-1.1-i2v`（最还原）；若用户/下游决定不传图，则把该镜 `model` 改 `happyhorse-1.1-t2v`、删除 `input.media`、在 `parameters` 补 `ratio`（默认 `16:9`，纯竖屏意图则 `9:16`）。本 skill 默认产出 **i2v + pending** 形态。

---

## 输出文件 2：report.md（模板骨架）

运行时按此骨架填充，保证留档质量稳定。**首行必须注明**“机器消费请看 shots.json，本文件仅留档”。

```markdown
# 视频解析报告：<source_file>

> 机器消费请看同目录 `shots.json`；本文件仅供人类留档，不被下游读取。
> 时长 Xs ｜ 分辨率 ｜ fps ｜ 是否含音轨 ｜ 抽帧间隔 Xs（共 N 帧）

## 一、整体调性
<题材/情绪/类型，1-3 句>

## 二、视觉风格 / 调色 / 画幅
<高key/过曝/暖调/胶片颗粒/画幅黑边 等>

## 三、人物设定（角色圣经）
- 角色A：<发色/服装造型1/造型2/表演风格>
- 角色B：<...>

## 四、分镜表
| 时间 | 场景 | 画面内容 | 机位/运动 | 字幕 |
|---|---|---|---|---|
| 00-02s | ... | ... | ... | ... |

## 五、字幕全量（带时间码）
<逐句，与 shots.json 的 captions_full 一致>

## 六、BGM / 音乐（推断，非听辨）
<风格/情绪；说明本模型不能听音频，歌词来自字幕OCR>

## 七、给 ai-video-generation 的适配说明
- i2v 首帧 = 各 `first_frame_local`，**需先传图床**成公网 URL 回填 `shots[].url`；不传则该镜退回 t2v。
- 字幕/黑边/拼接/原声：**模型做不了**，按 `post_process` 后期处理。
- 时长：单次 3-15s，本清单已按镜头拆好，逐条生成再按 `concat_order` 拼接。

## 八、生成产物索引
- shots.json：<绝对路径>
- 关键帧目录：<绝对路径>
```

---

## 执行流程

### 1. 定位视频
当前目录或用户指定路径的 `.mp4/.mov/.mkv/.webm`。用 `Glob`/`ls` 确认；多个则问用户。记下**绝对路径**。

### 2. 确保 ffmpeg 可用（自举）
```bash
command -v ffmpeg >/dev/null 2>&1 && FFMPEG=ffmpeg || {
  pip install -q imageio-ffmpeg >/dev/null 2>&1
  FFMPEG=$(python -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())")
}
echo "$FFMPEG"
```
后续用 `"$FFMPEG"` 调用（路径可能含空格，务必加引号）。

### 3. 读元数据
`"$FFMPEG" -i "<video>"` 抓 `Duration` / `Stream ... Video: ... WxH ... fps` / `Stream ... Audio`（有无音轨）。

### 4. 抽帧
默认 `fps=0.5`（每 2 秒 1 帧）。输出到 `<cwd>/video_frames/`：
```bash
mkdir -p video_frames
"$FFMPEG" -y -i "<video>" -vf fps=0.5 -q:v 2 video_frames/f_%03d.jpg
```
帧 i（从 1）对应时间 ≈ `(i-1)*interval` 秒。

### 5. 逐帧视觉识别（主会话 Read，**不派子代理**）
用**绝对路径** `Read` 每张 jpg，可同回复并行多张。每帧记录：场景/背景、主体、动作、服装、机位、**画面内字幕文字**。

### 6. 合并镜头（切镜检测）
相邻帧若**背景或服装突变** → 视为切镜，归入新 shot。为每个 shot 定：
- `timecode` = 首帧时间 ~ 末帧时间（见换算规则）。
- `first_frame_local` = 该 shot 首帧路径。
- `prompt` = `meta.style_prefix` + 该镜“单一画面 + 镜头运动”描述（**一镜一画面，不塞多场景**）。

### 7. 字幕归位
把每帧 OCR 字幕按帧时间去重合并（连续相同字幕合并为一条），形成 `captions_full`（带 `start/end`），并按 shot 时间窗切片填 `shots[].captions`。无字幕则空数组。

### 8. 音频探测（不做 ASR）
```bash
"$FFMPEG" -i "<video>" -af volumedetect -f null - 2>&1 | grep -E "mean_volume|max_volume"
```
mean/max 非静音 → `has_audio:true`，`audio_note` 写“原曲人声+伴奏；HappyHorse 自动音对不上，需后期合原声”。**歌词以字幕 OCR 为准，不调语音识别**（本模型不能听音频）。

### 9. 写 shots.json
严格按 schema v1，**合法 JSON 无注释**。`duration = clamp(round(镜头秒数),3,15)`；若某镜头跨度 >15s，**拆成多个 shot**。`url=null`、`url_status="pending"`。

### 10. 写 report.md
按模板骨架填充，首行加“机器消费请看 shots.json”。

### 11. 自检
```bash
python -c "
import json,sys
d=json.load(open('shots.json',encoding='utf-8'))
assert d.get('version')==1 and d.get('shots'), '缺 version 或 shots'
for s in d['shots']:
    b=s['body']; assert b.get('model') and b['input'].get('prompt') and 3<=b['parameters'].get('duration',0)<=15
    if b['model'].endswith('t2v'): assert b['parameters'].get('ratio'), 't2v 缺 ratio'
print('shots:',len(d['shots']),'| pending url:',sum(s['url_status']=='pending' for s in d['shots']),'| captions:',len(d['post_process']['captions_full']))
"
```
通过再继续；失败则修 json。

### 12. 收尾
告诉用户两个文件的**绝对路径** + 关键帧目录 + 自检结果，并补一句：
> 下游 `ai-video-generation` 只读 `shots.json`：取 `shots[].body` 直接 POST，按 `url_status` 选 i2v/t2v；字幕/黑边/拼接/原声按 `post_process` 后期处理。

---

## 镜头 / 字幕 / 时长 换算规则

- `interval` = 1 / 抽帧 fps（默认 2s）。帧 i 时间 = `(i-1)*interval`。
- shot 的 `timecode` = `首帧时间` ~ `下一 shot 首帧时间`（末个 shot 取到 `duration_sec`）。
- shot 的 `body.parameters.duration` = `clamp(round(末-首+interval 的覆盖秒数), 3, 15)`。
- caption 的 `start/end` = 该字幕首次出现帧时间 ~ 下一次变化帧时间（末条取到 `duration_sec`）。
- 时间统一保留 1 位小数；`timecode` 用 `MM:SS` 取整展示即可。

---

## 常见坑

- **Read 用相对/错误路径** → “文件不存在”。一律绝对路径。
- **把整段多场景散文塞进一个 prompt** → 模型在 3-15s 里糊成一团。必须一镜一画面。
- **本地 jpg 直接填进 i2v 的 url** → `url error`。本 skill 只放 `first_frame_local`，url 留 null。
- **以为模型能生成同步中文字幕 / 黑边 / 原曲** → 都不能，全归 `post_process` 后期。
- **写 json 带注释** → 非法。示例是 jsonc，落盘必须纯 JSON。

---

## 权限说明

`allowed-tools` 用 `Bash(*)`，因为本流程涉及多种 shell 操作（pip 自举、python 取 ffmpeg 路径、以**绝对路径**调用 ffmpeg、mkdir、自检），窄模式（如 `Bash(ffmpeg *)`）匹配不到 `"$FFMPEG" ...` 这种以变量/绝对路径开头的命令，会频繁触发权限或匹配失败。**正文已限定 Bash 仅用于**：检测/安装 ffmpeg、抽帧、音频探测、目录创建、json 自检。若你的权限策略偏严，可改为显式 allow 这些命令的具体前缀，但需自行处理绝对路径匹配。
