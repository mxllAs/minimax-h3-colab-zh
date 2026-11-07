# H3 Toybox

一个接在 ComfyUI 后面的 MiniMax H3 网页前端（H3 Studio 改名重做，2026-10-30）。

启动：`start_h3toybox.bat`（需要 ComfyUI 已在 8188 运行），开 http://127.0.0.1:8190

## 三种模式

| 模式 | H3 任务 | 条件节点 | 改写用的指南 |
|---|---|---|---|
| 🧸 参考生成 | Ref2VA | `MiniMaxH3ReferenceToVideo`（图 9／视频 3／声音 3） | `guides/..._ref_en.md` |
| 🎞 首尾帧 | I2VA／FL2VA／L2VA（看放了几张） | `MiniMaxH3ImageToVideo` | `guides/..._base_en.md` |
| ✏️ 纯文本 | T2VA | `MiniMaxH3ImageToVideo`（不接图） | `guides/..._base_en.md` |

- 改写模型是 Ref2VA 训练的，base 任务它常回六段式；`normalize_base()` 会把它收回三段式，并重建第一行的对齐句（那一行是固定格式）。
- 指南来源：<https://huggingface.co/MiniMaxAI/MiniMax-H3/tree/main/docs>

## 🧵 延伸视频（参考模式）

素材栏里的视频可以勾「延伸这支视频」，这时它不是参考视频，而是要被接下去的原片：

1. ffmpeg 把原片最后 3 秒转成 24fps、生成尺寸（`h3t_tail_*.mp4`）
2. `MiniMaxH3MotionContext`（NikoDemon80/ComfyUI-H3-Motion-Context）把尾巴 22 格钉进新片段开头（气锁），接在语意桥之后、采样器之前
3. 原片最后一帧同时当成一张参考图，告诉改写模型「从这一格接下去」
4. 解码后 `MiniMaxH3MotionContextTrim` 剪掉那 22 格，再用 ffmpeg 把原片接上新片段 → `outputs/<job>_full.mp4`

SelfLift 采样器会把 `minimax_keyframes` 缩到低分辨率阶段，所以首尾帧与气锁都能直接接两段采样。

## 工作流改动（相对 WF.json）

- 拿掉只为预览存在的 `MiniMaxChunkFeedForward`(401) 与 `ModelPreviewOverrideKJ`(329)，SigmaShift(333) 直接接 scheduler／sampler
- `VHS_VideoCombine` 换成内置 `CreateVideo` ＋ `SaveVideo`（存到 `output/H3Toybox/`）
- 需要的插件：T8 audio（Semantic Bridge）、SelfLift、H3 Motion Context

## 进度与剩余时间

服务器用标准库自己连 ComfyUI 的 websocket（不需要 websocket-client），送件时带同一个 client_id，就收得到那个 prompt 的 executing／progress 事件。

- 阶段：排队 → 准备 → 低分辨率采样 → 高分辨率采样 → 解码存盘 →（延伸）接回原片
- 预估：采样时用「已跑完的步」的中位数秒数外推；高分辨率步还没开始时，用历史的高／低步数比换算
- 每次跑完把各阶段时间（依像素×帧数范式）以移动平均写进 `stats.json`，越用越准

## 提示词改写的三种来源（设置页）

| 来源 | 说明 |
|---|---|
| 🧸 本地小模型（缺省） | H3 官方 9B 改写模型的 W4A8 量化版，在 ComfyUI 的 `TextGenerate` 里跑 |
| 💎 Gemini | 填 API 密钥；模型缺省 `gemini-3.8-flash`，清单另有 `gemini-3.5-flash-lite`，也可以自己输入；按「验证」会列出这把密钥可用的模型 |
| 🔌 OpenAI 兼容端点 | 端点网址、密钥（本机服务器可留空）、模型；「验证」送一个极短的 POST 确认端点活着，顺便读 `/models` |

- 三种来源吃的是同一份消息（`rewrite_message()`：system＝官方指南，user＝文本与图片交错）；base 任务的回复一律经过 `normalize_base()`。
- 密钥只存在服务器的 `config.json`，**不会传回浏览器**（只回 `has_key`）；表单留空＝沿用、按「清除」才会删。

## ⚡ 快速模式（缺省开）

SelfLift 两段采样从 12 步（低 6＋高 6）降到 8 步（低 4＋高 4）。步数设置在 `config.json` 的 `steps`。

## 📁 素材库文件夹

素材可以拖进文件夹整理；拖到「🧸 全部素材」就移出来。文件夹只有一层，存在 `library/folders.json`，素材的 `folder` 字段指向它。删文件夹不会删素材，素材会回到最外层。在文件夹里上传，或从生成结果「加入素材库」，都会直接放进目前的文件夹。

## 💾 数据放在哪里

用户做出来的东西都放在 `H3TOYBOX_DATA`（没设就是程序文件夹本身）：`library/`（含 `folders.json`）、`outputs/`、`jobs.json`、`stats.json`、`config.json`。
API 密钥另外存在 `H3TOYBOX_SECRETS`（缺省 `DATA/secrets.json`），不写进 `config.json`。
启动时 `restore_inputs()` 会把素材库里、但 ComfyUI input 文件夹缺的文件补回去（新的运行阶段 input 是空的）。

Colab 版 ③ 缺省 `SAVE_TO_DRIVE`：DATA＝「我的云端硬盘/H3Toybox」，ComfyUI 的 `output/H3Toybox` 也链接到那里的 `videos/`；
`SAVE_API_KEYS` 缺省不勾，密钥只放在运行阶段的暂存空间。

## 已知

- 延伸接点后约第 2 格有一帧几乎重复（帧差 1.5，正常约 19），播放时可能有 1/24 秒的顿点，尚未查明是哪一侧。

## Colab

repo 根目录的 `H3_Toybox.ipynb`：① 安装 → ② 下载约 50 GB → ③ 启动并用 Colab 内置代理开网页。
Colab 版的 `config.json` 由 ③ 写入（`free_after_rewrite: true`：改写完就卸载 9B 模型，避免和 H3 模型同时占内存）。
