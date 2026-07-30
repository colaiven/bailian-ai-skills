# 百炼 AI Skills

给 Claude Code 增加 **AI 图片生成**、**AI 视频生成** 和 **视频解析** 三个技能：前两者后端调用阿里云百炼 token-plan（通义万相 2.7 + HappyHorse 1.1），后者纯本地把视频拆成可重制的分镜清单（`shots.json`）与人类留档报告（`report.md`）。三者配合即可"解析一段视频 → 让 AI 重制"，全程无需额外装 CLI（视频解析缺 ffmpeg 时会自动 pip 装 imageio-ffmpeg）。

| Skill | 能力 | 模型 |
|-------|------|------|
| `ai-image-generation` | 文生图、图片编辑、多图参考、组图，最高 4K | `wan2.7-image` / `wan2.7-image-pro` |
| `ai-video-generation` | 文生视频、图生视频、参考图生视频，720P/1080P，3-15 秒 | `happyhorse-1.1-t2v/i2v/r2v` |
| `video-to-shots` | 解析视频 → 可重制分镜清单 `shots.json` + 人类留档 `report.md`（纯本地，不调 API） | 本地视觉解析（输出对接 `happyhorse-1.1-*`） |

## 🚀 安装（一句话）

把下面这句话发给你电脑上的 Claude（需要已安装 Claude Code）：

> 请安装这个仓库里的 skills：https://github.com/colaiven/bailian-ai-skills
> 按仓库 README 的"AI 安装指令"一节执行，API Key 我会提供。

Claude 会自动克隆仓库、复制 skills、配置 Key，你只需要粘贴一次百炼 API Key。

## 🤖 AI 安装指令

（本节写给执行安装的 Claude。人类手动安装见下一节。）

1. **克隆仓库**到临时目录：
   ```bash
   git clone https://github.com/colaiven/bailian-ai-skills.git /tmp/bailian-skills
   ```
   无 git 环境则下载 zip 解压。

2. **复制 skills** 到用户目录（覆盖即可）：
   ```bash
   mkdir -p ~/.claude/skills
   cp -r /tmp/bailian-skills/skills/ai-video-generation ~/.claude/skills/
   cp -r /tmp/bailian-skills/skills/ai-image-generation ~/.claude/skills/
   cp -r /tmp/bailian-skills/skills/video-to-shots ~/.claude/skills/
   ```

3. **配置 API Key**：询问用户索取阿里云百炼 token-plan 的 API Key（`sk-` 开头）。
   - 若 `~/.claude/settings.json` 的 `env` 中已有指向 `token-plan.cn-beijing.maas.aliyuncs.com` 的 `ANTHROPIC_AUTH_TOKEN`，说明本机 Claude 已由百炼托管，skills 会自动回退使用该变量，**可跳过本步**。
   - 否则将 Key 以 `BAILIAN_API_KEY` 合并写入（勿覆盖其他配置，勿改用 ANTHROPIC_AUTH_TOKEN，以免破坏用户的官方 Claude 认证）：
   ```bash
   node -e '
   const fs = require("fs"), os = require("os"), path = require("path");
   const p = path.join(os.homedir(), ".claude", "settings.json");
   let s = {}; try { s = JSON.parse(fs.readFileSync(p, "utf8")); } catch {}
   s.env = s.env || {};
   s.env.BAILIAN_API_KEY = process.argv[1];
   fs.writeFileSync(p, JSON.stringify(s, null, 2));
   console.log("已写入", p);
   ' "sk-用户的key"
   ```

4. **零成本验证** Key 有效性（只读请求，不花钱）：
   ```bash
   curl -s -H "Authorization: Bearer sk-用户的key" \
     https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1/models
   ```
   返回模型列表（HTTP 200）即配置成功；401 说明 Key 无效或与端点不配套。

5. 提醒用户**重启 Claude Code**，然后即可使用：
   - "画一张夕阳下的橘猫，水彩风格"
   - "生成一条视频：无人机航拍森林"
   - "解析这个视频，生成可重制的分镜清单和提示词"

## 📦 手动安装（人类版）

1. 复制 `skills/` 下全部子目录（`ai-image-generation` / `ai-video-generation` / `video-to-shots`）到 `~/.claude/skills/`
2. 在 `~/.claude/settings.json` 的 `env` 中加入：
   ```json
   { "env": { "BAILIAN_API_KEY": "sk-你的key" } }
   ```
3. 重启 Claude Code

⚠️ 不要把百炼 Key 写进 `ANTHROPIC_AUTH_TOKEN` 而不同时设置 `ANTHROPIC_BASE_URL`，那会让官方 Claude 认证失败。如果你想连 Claude Code 本体一起切换到百炼模型，需同时设置 `ANTHROPIC_BASE_URL=https://token-plan.cn-beijing.maas.aliyuncs.com/apps/anthropic` 及模型映射（如 `ANTHROPIC_MODEL=qwen3.8-max-preview`）。

## 用法示例

- 画一张赛博朋克城市夜景，霓虹灯，雨后的街道
- 生成一张 4K 的水墨山水（自动切换 pro 版）
- 把这张图里人物的衣服换成红色：[图片]
- 生成一条视频：一只猫在草地上奔跑，阳光明媚
- 把这张图片变成视频，镜头缓缓推进：[图片]
- 用这张人物照片作参考，生成他在海边散步的视频（r2v，保持人物一致）
- 解析当前目录这个视频，拆成可重制的分镜清单和提示词（video-to-shots，产出 `shots.json` + `report.md`）
- 用 video-to-shots 解析出的 `shots.json`，逐镜头调 ai-video-generation 重制这条视频

生成结果默认存到 `~/ai-out/image/` 和 `~/ai-out/video/` 下按日期分好的文件夹；想改位置直接告诉 Claude。

## 实测兼容性（token-plan 套餐）

| 功能 | 状态 |
|------|------|
| 文生图 / 图片编辑 / 多图参考 | ✅ |
| 文生视频 / 图生视频 / 参考图生视频 | ✅ |
| 视频编辑（视频生视频） | ❌ 套餐不含，需普通 DashScope Key 按量付费 |
| 视频解析为分镜清单（video-to-shots） | ✅ 纯本地，不消耗额度；缺 ffmpeg 时自动 pip 装 imageio-ffmpeg |

## 常见问题

| 现象 | 处理 |
|------|------|
| `401 invalid_api_key` | token-plan 的 key 只能配 `token-plan.cn-beijing.maas.aliyuncs.com` 域名，两者必须配套 |
| `Model not exist` | 模型名拼错或套餐不含该模型（见上表） |
| `url error`（图片接口） | 请求体须用 messages 结构（skills 已内置正确格式） |
| `does not support synchronous calls` | 视频接口须带 `X-DashScope-Async: enable` 请求头（已内置） |
| 找不到 skill | 未重启 Claude Code，或 skills 未放到 `~/.claude/skills/` |

## 费用与安全

- 图片、视频从 token-plan 额度中扣除，视频按秒计费；失败任务不计费
- API Key 等同密码：勿提交 git、勿公开发布；怀疑泄露立即去百炼控制台作废重建
