---
name: ai-image-generation
description: "使用阿里云百炼 token-plan 生成 AI 图片：通义万相 2.7（wan2.7-image / wan2.7-image-pro）。支持文生图、图片编辑（改图/换装/涂鸦移植）、多图参考生成、组图输出，最高 4K 分辨率。同步接口，秒级返回。触发词：生成图片、文生图、画一张、画个、做张图、图片编辑、改图、万相、AI 绘画、image generation, text to image, generate image, draw"
allowed-tools: Bash(curl *), Bash(node *)
---

# 百炼 AI 图片生成（通义万相 2.7）

通过阿里云百炼 token-plan 端点调用万相 2.7 图像模型。**同步接口**，直接返回图片 URL，无需轮询。

## 认证与端点

- API Key：读取环境变量 `$BAILIAN_API_KEY`（推荐，独立于 Claude 认证）；若未设置则回退到 `$ANTHROPIC_AUTH_TOKEN`（本机为百炼托管 Claude 时的配置）
- 命令中统一写法：`${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}`
- 端点：`https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation`
- 若两个变量都为空，停止并提示用户配置，**不要**让用户把 key 明文发在对话里

## 模型选择

| 模型 | 规格 | 适用 |
|------|------|------|
| `wan2.7-image` | 1K / 2K | 日常出图（默认，更快更省） |
| `wan2.7-image-pro` | 1K / 2K / **4K** | 高质量、大幅面、支持 `thinking_mode` 深度思考构图 |

用户要求"高清/4K/高质量"时用 pro，其余默认 `wan2.7-image`。

## 文生图

```bash
curl -s -X POST "https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation" \
  -H "Authorization: Bearer ${BAILIAN_API_KEY:-$ANTHROPIC_AUTH_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "wan2.7-image",
    "input": {
      "messages": [
        { "role": "user", "content": [ { "text": "一只橘猫坐在窗台上，窗外是夕阳，水彩画风格" } ] }
      ]
    },
    "parameters": { "size": "2K", "n": 1, "watermark": false }
  }'
```

## 图片编辑 / 多图参考

在 content 中先放图片（公网 URL），再放文字指令。模型能理解"图 1、图 2"的指代：

```bash
-d '{
    "model": "wan2.7-image-pro",
    "input": {
      "messages": [
        { "role": "user", "content": [
            { "image": "https://图片1.jpg" },
            { "image": "https://图片2.jpg" },
            { "text": "把图 2 的涂鸦喷绘在图 1 的汽车上" }
        ] }
      ]
    },
    "parameters": { "size": "2K", "n": 1, "watermark": false }
  }'
```

有图片输入时，输出宽高比跟随最后一张输入图，size 只支持 1K/2K。

## 参数说明

| 参数 | 可选值 | 说明 |
|------|--------|------|
| `size` | `1K` / `2K`（默认）/ `4K`（仅 pro、仅纯文生图） | 也可指定像素如 `"1280*720"`（注意是星号），总像素 768*768 ~ 4096*4096 |
| `n` | 1-4 | 生成张数 |
| `watermark` | `false` 关闭水印 | 建议默认 false |
| `thinking_mode` | `true`/`false` | 仅 pro，深度思考构图，更慢更贵 |
| `enable_sequential` | `true` | 组图模式（连环画/多视角），配合文字描述多张内容 |

## 响应解析与下载

成功响应（同步返回）：

```json
{ "output": { "choices": [ { "message": { "content": [ { "type": "image", "image": "<URL>" } ] } } ] } }
```

`n > 1` 时 content 数组里有多个 image 条目。URL 有效期有限，**立即下载**。

**默认保存目录**：`~/ai-out/image/{当天日期}/`（Windows 即 `C:\Users\用户名\ai-out\image\...`；用户指定过其他目录则从用户指定）
**文件名格式**：`wan-image-{时分秒}-{序号}.png`

推荐一步到位脚本（调用 + 解析 + 全部下载）：

```bash
OUT=~/ai-out/image/$(date +%Y-%m-%d) && mkdir -p "$OUT" && node -e "
const body = {
  model: 'wan2.7-image',
  input: { messages: [ { role: 'user', content: [ { text: process.argv[1] } ] } ] },
  parameters: { size: '2K', n: 1, watermark: false }
};
(async () => {
  const r = await fetch('https://token-plan.cn-beijing.maas.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation', {
    method: 'POST',
    headers: { Authorization: 'Bearer ' + (process.env.BAILIAN_API_KEY || process.env.ANTHROPIC_AUTH_TOKEN), 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });
  const j = await r.json();
  const images = j.output.choices[0].message.content.filter(c => c.image);
  const fs = require('fs');
  const ts = new Date().toTimeString().slice(0,8).replace(/:/g,'');
  for (let i = 0; i < images.length; i++) {
    const buf = Buffer.from(await (await fetch(images[i].image)).arrayBuffer());
    const p = process.argv[2] + '/wan-image-' + ts + '-' + (i+1) + '.png';
    fs.writeFileSync(p, buf);
    console.log('已保存:', p);
  }
})();
" "<提示词>" "$OUT"
```

下载完成后，把完整路径（Windows 格式）告诉用户。若用户指定了保存位置，以用户指定为准。

## 常见错误

| 错误 | 原因与处理 |
|------|-----------|
| `url error` | 请求体格式错误（本接口用 messages 结构，不是 OpenAI 的 prompt 字段）；或图片编辑时图片 URL 不可访问 |
| `401 invalid_api_key` | key 与端点不配套（token-plan key 只能配 token-plan 域名） |
| `Model not exist` | 模型名错误，用 `wan2.7-image` 或 `wan2.7-image-pro` |
| 4K 报错 | 4K 仅 pro 且仅纯文生图（无图片输入、非组图）支持 |
