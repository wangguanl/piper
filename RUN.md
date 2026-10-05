# 运行命令

- 项目：piper（rhasspy/piper）
$12026-09-09
- 运行方式：直接运行（官方 Windows 二进制 + Flask HTTP 桥）
- 硬件评估：**满足** — Piper 官方面向 Raspberry Pi 4；ONNX 单音色约数十 MB，CPU 即可。本机 RTX 4080 / 16GB（评估时占用约 6.2GB），远超需求。Docker 仅用于编译 C++ 发行包，不跑服务。

## 环境准备

```powershell
$env:Path = "E:\Programs\ffmpeg-master-latest-win64-gpl\bin;$env:Path"
$env:UV_INDEX_URL = "https://pypi.tuna.tsinghua.edu.cn/simple"

# 1) 官方 Windows 二进制（本仓库 src/python_run 在 Windows 无 piper-phonemize 轮子，不可用）
New-Item -ItemType Directory -Force -Path E:\AI\local-voice\piper\runtime | Out-Null
curl.exe -L --fail -o E:\AI\local-voice\piper\runtime\piper_windows_amd64.zip `
  "https://ghproxy.net/https://github.com/rhasspy/piper/releases/download/2023.11.14-2/piper_windows_amd64.zip"
# 也可用 GitHub 直链；慢则走 ghproxy
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory(
  'E:\AI\local-voice\piper\runtime\piper_windows_amd64.zip', 'E:\AI\local-voice\piper\runtime')

# 2) 音色（HF 官方超时，用镜像直链）
New-Item -ItemType Directory -Force -Path E:\AI\local-voice\piper\models | Out-Null
curl.exe -L --fail -o E:\AI\local-voice\piper\models\zh_CN-huayan-medium.onnx `
  "https://hf-mirror.com/rhasspy/piper-voices/resolve/v1.0.0/zh/zh_CN/huayan/medium/zh_CN-huayan-medium.onnx"
curl.exe -L --fail -o E:\AI\local-voice\piper\models\zh_CN-huayan-medium.onnx.json `
  "https://hf-mirror.com/rhasspy/piper-voices/resolve/v1.0.0/zh/zh_CN/huayan/medium/zh_CN-huayan-medium.onnx.json"

# 3) HTTP 桥依赖（flask）；可选装 PyPI piper-tts 继任包作对照，中文请仍用官方 exe
cd E:\AI\local-voice\piper\src\python_run
uv venv .venv
uv pip install flask
```

## 启动

- 推荐：`pwsh -NoProfile -File .\start.ps1`（菜单；默认 [1] HTTP 桥）
- 服务 Id：`http`、`cli`（冒烟写 `models/cli_test.wav`）
- 可选：`-Model <onnx路径>`（默认 `models\zh_CN-huayan-medium.onnx`）
- 等价手动命令：

```powershell
# CLI
'你好，这是 Piper 语音合成测试。' |
  & E:\AI\local-voice\piper\runtime\piper\piper.exe `
    --model E:\AI\local-voice\piper\models\zh_CN-huayan-medium.onnx `
    --output_file E:\AI\local-voice\piper\models\cli_test.wav

# HTTP
& E:\AI\local-voice\piper\src\python_run\.venv\Scripts\python.exe `
  E:\AI\local-voice\piper\runtime\http_bridge.py `
  --host 127.0.0.1 --port 5000 `
  --piper E:\AI\local-voice\piper\runtime\piper\piper.exe `
  --model E:\AI\local-voice\piper\models\zh_CN-huayan-medium.onnx
```

## 验证

```powershell
curl.exe -G --data-urlencode "text=你好，这是测试。" -o E:\AI\local-voice\piper\models\http_test.wav "http://127.0.0.1:<端口>/"
# 预期：WAV 非空（通常 >1KB）
```

CLI 实测：`cli_test.wav` ≈ 119KB，RTF ≈ 0.07。

## 备注

- 端口起点 5000，占用则 `start.ps1` 自动顺延。
- 镜像：GitHub Release 可用 `ghproxy.net`；HF=`https://hf-mirror.com`；PyPI=清华。
- Windows 不能 `pip install -e src/python_run`（无 `piper-phonemize` win 轮子）。上游开发已迁至 https://github.com/OHF-Voice/piper1-gpl ；PyPI `piper-tts` 1.8.0 有 win 轮子，但当前版本在本机合成中文会触发 espeak Unicode 问题，故 HTTP 走官方 `piper.exe` 桥接。
- `RUN.md` / `start.ps1` / `runtime/` / `models/` 为本机材料，默认不提交。
