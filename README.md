# EmoManifold 二维演示

这个仓库只包含一条闭环：一张二维 Grimace 漫画脸，以及驱动它的演示。

文本进去，每个 token 带上效价（valence）和唤醒度（arousal），FKCN 把这两个数收成表情权重，Grimace 把权重画成脸。这里没有论文文稿，也没有实体机器人或 29 自由度硬件。

许可证是 MIT，版权人 Yige Liu，2026。见 [LICENSE](LICENSE)。第三方组件见 [NOTICE](NOTICE)。

## 闭环里有什么

1. **浏览器演示。** 页面把对话发给本机 API。API 流式返回 token 和 VAD。页面自己做 FKCN，并用 `grimace-main` 画出脸。
2. **不加载语言模型。** `share/grimace_from_va` 只吃 valence / arousal，画出同一张脸。它不需要 PyTorch。

默认对话模型是 `Qwen/Qwen2.5-0.5B-Instruct`（`config.py` 里的 `qwen`）。权重不在仓库里，第一次运行前要下载，并在本机拟合 scaler。

## 环境

- Python 3.9 及以上。推理可以用 CPU。有 NVIDIA GPU 时把 `EMO_DEVICE` 设为 `cuda`；苹果 GPU 设为 `mps`。
- 浏览器演示需要 Node.js 20。
- 把 VA 画成 PNG 需要 Playwright 的 Chromium。合成 MP4 还需要本机有 ffmpeg。

## 不下载模型，先把脸画出来

在 `share/grimace_from_va` 里：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m grimace_from_va --self-test
python -m grimace_from_va --valence 0.81 --arousal 0.66
```

`--self-test` 只检查 VA 到 6 维权重的映射，不开浏览器。写成 PNG：

```bash
python -m playwright install chromium
python -m grimace_from_va --valence 0.81 --arousal 0.66 -o happy.png
```

一条轨迹写成 MP4（需要 ffmpeg）：

```bash
python -m grimace_from_va --csv example/sample_va.csv -o out.mp4
```

更多参数见 `share/grimace_from_va/README.md`。Windows 上激活虚拟环境用 `.venv\Scripts\activate`。

## 浏览器里的对话闭环

模型约 0.5B，下载和第一次拟合 scaler 都会花一些时间。下面的命令在仓库根目录执行。

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-api.txt
python scripts/download_model.py
python scripts/train_scaler.py --model qwen
EMO_DEVICE=cpu python scripts/api_server.py
```

`scripts/install_dependencies.sh` 只安装 `requirements.txt`。跑 API 时用上面的 `requirements-api.txt`，里面已经包含前者。

API 监听 `http://127.0.0.1:8788`。`GET /health` 在模型加载完之后返回 `{"ok": true}`。`POST /v1/chat/stream` 以 SSE 返回 token 和 VAD。端口用环境变量 `EMO_API_PORT` 改。

另开一个终端：

```bash
cd web
npm ci
npm run dev
```

浏览器打开 Vite 打印的地址。页面默认请求 `http://127.0.0.1:8788`。要改地址，复制 `web/.env.example` 为 `web/.env`，设置 `VITE_API_URL`，再重新启动 `npm run dev`。不要把 `web/.env` 提交进仓库。

发送一句话之后，回答逐 token 出现。悬停 token 可以看到 V、A，以及模型给出的 D。右侧的脸会跟着效价和唤醒度变化。

先构建再由 API 托管页面也可以：

```bash
cd web && npm ci && npm run build && cd ..
EMO_DEVICE=cpu python scripts/api_server.py
```

然后打开 `http://127.0.0.1:8788`。构建时若设置了 `VITE_API_URL`，页面会请求那个地址；不设置时同样是 `http://127.0.0.1:8788`。

换模型时同时改下载、scaler 和启动：

```bash
EMOTION_MODEL=ernie python scripts/download_model.py
EMOTION_MODEL=ernie python scripts/train_scaler.py --model ernie
EMOTION_MODEL=ernie EMO_DEVICE=cpu python scripts/api_server.py
```

`ernie` 用 `data/vad_lexicon/vad_lexicon_chinese_ernie.json`，`qwen` 用 `vad_lexicon_chinese_qwen.json`。scaler 写到 `outputs/scalers/<模型名>_scaler.json`。这个目录被 git 忽略。API 启动时如果找不到对应 scaler，会直接报错。

## 目录

```
grimace-main/             二维 Grimace 引擎（上游 MIT）
share/grimace_from_va/    VA → 脸，不需要语言模型
web/                      浏览器演示
scripts/api_server.py     本机推理 API，127.0.0.1:8788
scripts/download_model.py 按 config.py 下载当前模型
scripts/train_scaler.py   拟合 outputs/scalers/
src/                      投影、FKCN、流式生成
data/vad_lexicon/         运行时极词表
data/training_corpus.txt  拟合 scaler 用的句子
config.py                 模型名与路径
```

## 词库

`data/vad_lexicon/` 里是运行时用的极词 JSON，不是整份 NRC VAD Lexicon。整份词表不在本仓库中，也不应再放进来。这些 JSON 里的分数来自 NRC VAD Lexicon。使用这些分数时请引用 Mohammad, ACL 2018。详见 [NOTICE](NOTICE)。

## 许可证

- 本仓库原创代码：MIT，Copyright (c) 2026 Yige Liu。
- Grimace：MIT，见 `grimace-main/LICENSE` 与 `share/grimace_from_va/grimace_from_va/vendor/LICENSE.grimace`。上游 <https://github.com/grimace-project/web>。
