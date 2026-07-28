---
name: ai-video-generation
description: "使用阿里云百炼 token-plan 生成 AI 视频：HappyHorse 1.1 文生视频（t2v）与图生视频（i2v），支持 720P/1080P、3-15 秒、带音频。用途：短视频、营销素材、产品演示、创意动画。触发词：视频生成、AI 视频、文生视频、图生视频、百炼、HappyHorse、通义万相、生成视频、video generation, text to video, image to video, t2v, i2v"
allowed-tools: Bash(curl *), Bash(node *)
---

# 百炼 AI 视频生成（HappyHorse）

通过阿里云百炼 token-plan 端点调用 HappyHorse 1.1 视频模型。**异步任务**：创建任务 → 轮询状态 → 下载视频。

## 认证与端点

- API Key：读取环境变量 `$BAILIAN_API_KEY`（推荐，独立于 Claude 认证）；若未设置则回退到 `$ANTHROPIC_AUTH_TOKEN`（本机为百炼托管 Claude 时的配置）
- 命令中统一写法：`${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}`
- Base URL：`https://token-plan.cn-beijing.maas.aliyuncs.com`
- 若两个变量都为空，停止并提示用户配置，**不要**让用户把 key 明文发在对话里

## 模型

| 模型 | 用途 | 输入 |
|------|------|------|
| `happyhorse-1.1-t2v` | 文生视频 | 仅提示词 |
| `happyhorse-1.1-i2v` | 图生视频（首帧驱动） | 提示词 + 首帧图片 URL |
| `happyhorse-1.1-r2v` | 参考图生视频（定人物/风格） | 提示词 + 1-9 张参考图 |
| ~~`happyhorse-1.0-video-edit`~~ | 视频编辑（v2v） | ❌ token-plan 不支持，勿调用 |

## 第一步：创建任务

### 文生视频（t2v）

```bash
curl -s -X POST "https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/video-generation/video-synthesis" \
  -H "X-DashScope-Async: enable" \
  -H "Authorization: Bearer ${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "happyhorse-1.1-t2v",
    "input": { "prompt": "一只猫在草地上奔跑，阳光明媚，电影感镜头" },
    "parameters": { "resolution": "720P", "ratio": "16:9", "duration": 5 }
  }'
```

### 图生视频（i2v）

```bash
curl -s -X POST "https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/video-generation/video-synthesis" \
  -H "X-DashScope-Async: enable" \
  -H "Authorization: Bearer ${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "happyhorse-1.1-i2v",
    "input": {
      "prompt": "镜头缓缓推进，云层流动",
      "media": [ { "type": "first_frame", "url": "https://示例图片.png" } ]
    },
    "parameters": { "resolution": "720P", "duration": 5 }
  }'
```

### 参考图生视频（r2v）

用 1-9 张参考图锁定人物外貌、物体或画面风格，再按提示词生成全新视频：

```bash
curl -s -X POST "https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/video-generation/video-synthesis" \
  -H "X-DashScope-Async: enable" \
  -H "Authorization: Bearer ${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "happyhorse-1.1-r2v",
    "input": {
      "prompt": "图中的人物缓缓转头微笑，电影感光影",
      "media": [
        { "type": "reference_image", "url": "https://参考图1.jpg" }
      ]
    },
    "parameters": { "resolution": "720P", "duration": 5 }
  }'
```

成功返回：`{"output":{"task_id":"...","task_status":"PENDING"},...}`

## 第二步：轮询任务状态

每 20 秒查询一次，直到 `SUCCEEDED` 或 `FAILED`。生成通常耗时 **1-5 分钟**。

```bash
curl -s "https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/tasks/{task_id}" \
  -H "Authorization: Bearer $ANTHROPIC_AUTH_TOKEN"
```

推荐用 node 一次性轮询到底（避免反复调用）：

```bash
node -e "
const sleep = ms => new Promise(r => setTimeout(r, ms));
(async () => {
  for (let i = 0; i < 30; i++) {
    const r = await fetch('https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/tasks/{task_id}', {
      headers: { Authorization: 'Bearer ' + (process.env.BAILIAN_API_KEY || process.env.ANTHROPIC_AUTH_TOKEN) }
    });
    const j = await r.json();
    const st = j.output?.task_status;
    console.log('状态:', st);
    if (['SUCCEEDED','FAILED','CANCELED','UNKNOWN'].includes(st)) { console.log(JSON.stringify(j, null, 2)); return; }
    await sleep(20000);
  }
  console.log('超时');
})();
"
```

## 第三步：下载视频

成功后视频地址在 `output.video_url`（有效期有限，立即下载）。

**默认保存目录**：`~/ai-out/video/{当天日期}/`（Windows 即 `C:\Users\用户名\ai-out\video\...`；用户指定过其他目录则从用户指定）
**文件名格式**：`happyhorse-{t2v|i2v}-{时分秒}.mp4`，按日期分目录、时间戳命名，避免覆盖

```bash
OUT=~/ai-out/video/$(date +%Y-%m-%d)
mkdir -p "$OUT"
curl -L -o "$OUT/happyhorse-i2v-$(date +%H%M%S).mp4" "<output.video_url 的值>"
```

下载完成后，把完整路径（Windows 格式）告诉用户。若用户指定了保存位置，以用户指定为准。

## 参数说明

| 参数 | 可选值 | 说明 |
|------|--------|------|
| `resolution` | `720P` / `1080P` | 分辨率，1080P 更贵更慢 |
| `duration` | `3`-`15` | 视频秒数，按秒计费 |
| `ratio` | `16:9` / `9:16` / `1:1` / `4:3` / `3:4` | 仅 t2v 支持；i2v 由图片比例决定 |

## 重要约束

1. **异步接口**：请求头 `X-DashScope-Async: enable` 必须带，否则报错
2. **不要重复提交**：任务创建后只轮询，不重复创建
3. `task_id` 有效期 24 小时
4. 按秒计费，生成前跟用户确认时长和分辨率
5. 图片 URL 必须公网可访问；本地图片需先上传图床
6. **视频编辑（v2v）不支持**：`happyhorse-1.0-video-edit`、`wan2.7-videoedit` 在 token-plan 端点返回 `Model not exist`，勿尝试调用；用户要求"视频生视频/改视频"时，说明需改用普通 DashScope API Key（按量付费）
7. t2v 不接受图片输入（传了也会被忽略）；r2v 的 media 类型必须为 `reference_image`

## 常见错误

| 错误 | 原因与处理 |
|------|-----------|
| `401 invalid_api_key` | key 与端点不配套（token-plan key 只能配 token-plan 域名） |
| `Model not exist` | 模型名拼写错误，用本文件列出的模型名 |
| `url error` | i2v 的图片 URL 不可访问 |
| `does not support synchronous calls` | 漏了 `X-DashScope-Async: enable` 请求头 |
