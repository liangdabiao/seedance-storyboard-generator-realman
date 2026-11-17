# 备选通道：apiz.ai（gpt-image flare 生图 + MiniMax-H3 生视频）

> 用途：当默认通道（即梦 Seedream / Seedance）**积分不足、未授权或排队过久**时，改用 apiz.ai 跑同一套「文→资→视」流程。
> 本文只讲**怎么调通**，分镜提示词怎么写仍以 `SKILL.md` 与 `references/真人感电影写实指南.md` 为准，两边不冲突。
> 所有价格与参数均为 2026-11 实测，模型升级后以 `apiz models info <id> --json` 为准。

---

## 1. 选型对照

| 环节 | 本通道模型 | 档位 | 说明 |
|---|---|---|---|
| 生图 | `apiz/gpt-image-2.5-flare` | `quality: low` | 最多 16 张参考图；low 档已足够做资产图与首帧 |
| 生视频 | `minimax/h3` | `480p` | 4–15 秒整数；原生带音频；支持图片/视频/音频混合参考 |
| 备选高清 | `minimax/h3` | `768P` / `1080p` / `2K` | 只在交付级成片时升级，价格约为 480p 的 1.5 / 2.3 / 2.5 倍 |

**实测单价（2026-11）**

- 生图 low 档：**12–19 积分/张**，随画面复杂度与 token 量浮动（竖版 9:16 出现过 19 分，横版 16:9 与另一次竖版都是 12 分）。**不要按 2 分/张估算**——那是 1:1 小图的历史数据。
- 视频 480p：**12 积分/秒**，另加每次 **Context 7 积分**；15 秒一条 = 180 + 7 = **187 分**。
- 参考图：**前 5 张免费**，超出后 7 积分/张；参考视频按输出单价计。
- 单集 15 秒全成本（2 张图 + 1 条视频）≈ **211–225 分**。

---

## 2. 前置：CLI 与账号

```bash
A=/c/Users/49707/.local/bin/apiz.exe    # Windows Git Bash 写法，不在 PATH
$A user info --json                      # 确认登录身份
$A account balance --json                # 账户积分
```

**两个额度不是一回事（必看）**：`account balance` 是账户总积分，**每个 API Key 另有独立配额上限**。配额用尽时会报：

```
HTTP 429 — 计费失败: API Key 额度不足，剩余额度: 166.00，本次需要: 187
```

此时账户里可能还有几千分也照样跑不动，CLI 无命令可改，只能去 apiz.ai 控制台**重置/提高该 Key 的额度**或**新建 Key**。`account checkin` 签到接口目前返回「签到活动已结束」，别指望靠签到补分。

---

## 3. 生图：flare 2.5

### 3.1 纯文生图

```bash
$A generate "$(cat assets/C01.txt)" \
  --model apiz/gpt-image-2.5-flare \
  --aspect-ratio "9:16" \
  --params '{"quality":"low"}' \
  --wait --json
```

### 3.2 图生图（保持同一个人/同一场景）

先把参考图变成公网 URL。**本地图必须走 `upload`，外链走 `transfer`（免费）**：

```bash
$A upload assets/C01_ref.png --folder myproj --remark ref --json
#  → {"public_url":"https://cdn-hk.51sux.com/storage/myproj/.../xxx.png"}

$A transfer "https://某外链.com/a.png" --type image      # 外链转存，免费
```

再把 URL 塞进 `reference_image_urls`：

```bash
$A generate "$(cat assets/C01.txt)" \
  --model apiz/gpt-image-2.5-flare \
  --aspect-ratio "9:16" \
  --params '{"quality":"low","reference_image_urls":["https://cdn-hk.51sux.com/storage/.../ref.png"]}' \
  --wait --json
```

**注意**

- 比例必须显式写 `--aspect-ratio`，不写会按 1:1 出图。
- **不要传** `image_size` / `resolution` / `num_images`，会直接 HTTP 400。
- 返回结果里 `result.images[0].url` 是 `v3-tasks/` 临时链接，**会过期**；下载落盘后再 `upload` 一次换成 `storage/` 永久地址，供后续视频引用。
- **跨年龄 i2i 要换真实年龄参考图**。只写"同一人成熟二十岁"模型会明显弱化年龄：二十出头的人像喂进去，出来还是二十多岁。正确做法是找该年龄段的真实参考图，并在提示词里写死发际线、鬓角、皱纹深度、面部凹陷这些可判定的特征。

### 3.3 落盘

```bash
curl -sL -o "assets/C01.png" "https://cdn-hk.51sux.com/v3-tasks/....png"
```

---

## 4. 生视频：MiniMax-H3

### 4.1 接口形状（最容易踩的坑）

H3 的入参是一个 **`content` 数组**，正文和素材都在里面，`role` 区分用途：

```json
{
  "content": [
    {"type": "text", "text": "<整段提示词正文>"},
    {"type": "image_url", "image_url": {"url": "https://.../C01.png"}, "role": "reference_image"},
    {"type": "image_url", "image_url": {"url": "https://.../S01.png"}, "role": "reference_image"}
  ],
  "resolution": "480p",
  "duration": 15,
  "ratio": "16:9",
  "aigc_watermark": false
}
```

**`params.content` 会整体覆盖命令行里的 prompt 参数。** 只传图片项、把正文交给命令行位置，会得到：

```
HTTP 400 — content 必须包含至少一个非空 text 项
```

这个报错**不扣费**，可以放心先用它探接口形状。正文必须写成 `content[0] = {"type":"text","text":...}`，命令行第一个位置参数传个占位串（如 `"E01"`）即可。

其余约束：`content` / `resolution` / `duration` 三项必填；`duration` 是 4–15 的整数；`ratio` 可选 `adaptive`、`21:9`、`16:9`、`4:3`、`1:1`、`3:4`、`9:16`。

### 4.2 用 Python 拼参数，避免 shell 转义

中文提示词直接塞命令行容易炸，统一走 JSON 文件：

```bash
python -c "
import json,io
prompt=io.open('E01_prompt.txt',encoding='utf-8').read().strip()
imgs=['https://cdn-hk.51sux.com/storage/.../C01.png',
      'https://cdn-hk.51sux.com/storage/.../S01.png']
content=[{'type':'text','text':prompt}]+[{'type':'image_url','image_url':{'url':u},'role':'reference_image'} for u in imgs]
p={'content':content,'resolution':'480p','duration':15,'ratio':'16:9','aigc_watermark':False}
io.open('E01_params.json','w',encoding='utf-8').write(json.dumps(p,ensure_ascii=False))
"
```

### 4.3 提交与取件

```bash
$A generate "E01" --model minimax/h3 --params "$(cat E01_params.json)" --wait --json > .e01.json
```

- `--wait` 会阻塞到终态，15 秒片子约 30–60 秒返回。
- 成功时 JSON 里有 `result.video_url`、`result.resolution`、`result.ratio`、`result.usage` 和 `price`（实际扣费）。
- 失败/额度不足会打印 `Error:` 到 stderr，注意用 `> file 2>&1` 一起接住。

```bash
curl -sL -o "视频/E01.mp4" "<video_url>"
```

---

## 5. 成片自检（零成本，先做再交）

```bash
# 规格与音轨
ffprobe -v error -show_entries format=duration -show_entries stream=codec_type,codec_name,width,height -of default=nw=1 视频/E01.mp4

# 音量：确认不是静音、说话段能量略高于全片
ffmpeg -hide_banner -i 视频/E01.mp4 -af volumedetect -f null /dev/null 2>&1 | grep -E "mean_volume|max_volume"
ffmpeg -hide_banner -ss 8 -t 3 -i 视频/E01.mp4 -af volumedetect -f null /dev/null 2>&1 | grep -E "mean_volume|max_volume"

# 抽帧拼图，一次看全片（2 列 3 行）
ffmpeg -y -i 视频/E01.mp4 -vf "select='eq(n\,15)+eq(n\,120)+eq(n\,200)+eq(n\,270)+eq(n\,360)+eq(n\,430)',scale=416:-1,tile=2x3" -frames:v 1 视频/contact.png
```

判断口型：**抽说话那 1–2 秒的连续帧看嘴是否张开**，比事后跑语音识别便宜得多，也别为此安装任何模型。

---

## 6. 已知的成片偏差（跑之前先防）

- **H3 会把动作演快**：时间轴写"11.5 秒起身"，实际 9.3 秒就站起来了，尾段容易拖空。留白别压在最后一段，关键动作往前放，收尾给 2–3 秒缓冲。
- **开场不要给纯空镜**。写「特写桌面：台灯、稿纸、铅笔」会真的出 4 秒无人画面，观众第一反应是"人不见了"。15 秒短片的第一镜必须有人在场（中景双人同框最稳），空镜只能当插入镜。
- **参考图 2–3 张最有效**：人物全身 + 场景正打 + （可选）配角。超过 5 张开始收费，且并非越多越准。

---

## 7. 成本预算模板

```bash
# 跑之前先把这张表报给用户，得到确认再执行
图片：N 张 × 12~19 分 = ___
视频：集数 × 187 分（480p/15s，含 7 分 Context）= ___
合计：___ 分（1 积分 = 0.01 元）
```

按用户既定要求：**任何消耗积分的调用（含重做、返拍）执行前先查余额、报预估、等确认**；用户没点名的环节不跑。
