# 📖 MOSS-Music-8B AutoDL 预配置镜像

本项目提供了一个基于 MOSS-Music-8B-Instruct 模型的 AutoDL 社区镜像说明文档。该镜像已为您封装好 PyTorch、CUDA、SGLang 及相关依赖，实现开箱即用，免去繁琐的本地环境搭建过程。

---

## ✨ 镜像简介

本镜像专为高精度的音乐理解与分析任务构建，适用于需要处理长音频、提取旋律、识别歌词及进行音乐结构分析的场景。

* **基础环境**：Python 3.12 / PyTorch 2.9.1+cu128 / CUDA 12.8
* **内置模型**：MOSS-Music-8B (Instruct 版本，已随镜像打包在系统盘)
* **推理框架**：SGLang + FlashInfer
* **硬件建议**：强烈推荐 RTX 4090 / A5000 / A100。8B 模型在 FP16 精度下加载需约 17-18GB 显存，12GB 显卡（如 3080Ti）会直接 OOM（显存溢出）。

---

## 🚀 快速开始与使用指南

在 AutoDL 平台基于此镜像创建实例并开机后，请按照以下步骤操作。

### 1️⃣ 确认环境与路径

镜像已默认配置好环境，且模型已内置在系统盘中。请先进入项目根目录，确认模型文件存在：

```bash
cd /root/MOSS-Music
ls -lh ./weights/MOSS-Music-8B-Instruct/
# 预期应看到 4 个 .safetensors 文件及 config.json 等
```

---

### 2️⃣ 启动 SGLang 推理服务

在终端中执行以下命令启动后台服务（默认使用 30000 端口）：

```bash
python -m sglang.launch_server \
  --model-path ./weights/MOSS-Music-8B-Instruct \
  --port 30000 \
  --trust-remote-code \
  --mem-fraction-static 0.85
```

> **注**：
> 1. 必须携带 `--trust-remote-code` 参数，因为 MOSS-Music 使用了自定义的模型架构代码。
> 2. 如果处理超长音频导致 OOM，请适当降低 `--mem-fraction-static`（如降至 0.7）。
> 3. 首次启动需要加载权重，请耐心等待直到终端出现 `Uvicorn running on http://0.0.0.0:30000`。

---

### 3️⃣ 两种使用方式（任选其一）

#### 👉 方式一：Gradio 网页版 UI（推荐，最直观）

如果您希望通过可视化界面上传音频并进行对话，请按照以下步骤操作：

##### 第一步：启动 WebUI 服务
在 AutoDL 的 JupyterLab 中，**新开一个终端**（不要在跑 SGLang 的那个终端里操作），执行：

```bash
cd /root/MOSS-Music
python app.py
```

##### 第二步：等待启动
终端会输出类似 `Running on local URL: http://127.0.0.1:7860` 的提示，说明网页版 UI 已在后台启动完成。**请保持这个终端不要关闭**。

##### 第三步：配置 AutoDL 端口映射
1. 回到 AutoDL 控制台，找到你的实例。
2. 点击 **“自定义服务”** 按钮（通常在“快捷工具”区域）。
3. 在弹出的窗口里，将端口号填入 `7860`。
4. 点击“确定”或“开启”，AutoDL 会为你生成一个可访问的外网链接。

##### 第四步：在浏览器中打开
点击生成的链接，即可打开 MOSS-Music 的网页界面。在网页里拖拽上传音频，输入你的 Prompt，就能看到分析结果了。

> **💡 常见报错：如果网页打不开？**
> * 检查终端是否报错（如缺少 Gradio 库，可执行 `pip install gradio` 补装）。
> * 检查端口映射是否填写正确（必须是对应 `app.py` 启动时打印的真实端口，通常是 7860）。
> * 检查是否在 AutoDL 控制台点击了“开启”映射，有时候映射需要几秒钟生效。

---

#### 👉 方式二：HTTP 请求调用（适合代码集成与快速测试）

如果你需要把功能集成到自己的软件或脚本里，可以直接向 SGLang 接口发送请求。SGLang 的多模态接口需要将音频转为 Base64，并使用带前缀的 `data:audio/wav;base64,...` 格式。

你可以使用以下 Python 脚本发送请求：

```python
import requests
import base64

# 1. 将你的音频转为 Base64 并拼接前缀
with open("/root/test_audio.wav", "rb") as f:
    audio_b64 = base64.b64encode(f.read()).decode("utf-8")
    audio_data_uri = f"data:audio/wav;base64,{audio_b64}"

# 2. 构造请求发送给 SGLang 接口
url = "http://127.0.0.1:30000/generate"
payload = {
    "text": "请详细分析这段音频的曲式结构和情感走向。",
    "audio_data": audio_data_uri,
    "sampling_params": {"temperature": 0.3, "max_new_tokens": 1024}
}

response = requests.post(url, json=payload)
print(response.json().get("text"))
```

---

## 💡 常见使用流程

1. 将测试音频文件上传到系统盘（如 `/root/` 目录下，或者直接使用你上传的路径）。
2. 启动 SGLang 服务端（使用方式二必须启动；使用方式一的 WebUI 通常也需要它提供底层推理支持）。
3. 启动 WebUI（方式一）或使用 Python 脚本发送请求（方式二）。
4. 在终端按 `Ctrl+C` 即可停止相应的服务或网页。

---

## ⚠️ 开源协议与声明

MOSS-Music 模型本身遵循 Apache 2.0 开源协议发布。本镜像仅为方便开发者快速部署而制作的环境封装镜像，未对原模型权重进行任何恶意篡改。关于模型的更多技术细节与原始代码，请参考 MOSS-Music 官方 GitHub 仓库：[https://github.com/OpenMOSS/MOSS-Music](https://github.com/OpenMOSS/MOSS-Music)

---

## 📌 常见问题与注意事项

1. **系统盘空间**：由于模型打包在系统盘（`/root`），系统盘初始占用较大，请勿随意重置系统。
2. **端口映射**：使用 `app.py` (WebUI) 时，请确保在 AutoDL 控制台开启了对应的自定义服务端口（默认 7860）。
3. **显存溢出 (OOM)**：若处理超长音频时遇到显存溢出，请尝试调低 `--mem-fraction-static` 参数。