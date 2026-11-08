---
name: high-res-redraw
description: 高清重绘预处理工作流——对输入原图依次执行「PS 等效高斯模糊 8」「ControlNet Lineart 线稿提取(1024)」「从生成逻辑反推中文提示词」，最终交付一张模糊图、一张线稿图、一段反推提示词。适用于想对某张插画/角色图做 AI 高清重绘(img2img / ControlNet)前的参考与线稿准备。
description_zh: "高清重绘预处理：高斯模糊8 + Lineart线稿(1024) + 反推中文提示词"
description_en: "Hi-res redraw prep: Gaussian blur 8 + ControlNet Lineart (1024) + reverse-prompt"
---

# 高清重绘 (High-Res Redraw)

把一张原图预处理成「高清重绘」所需的三件套：**模糊参考图 + 干净线稿 + 反推中文提示词**。

## 何时使用
- 用户想对某张插画 / 角色图做 AI 高清重绘（img2img 或 ControlNet），需要先准备模糊参考图、线稿与提示词。
- 用户明确点名：高斯模糊 + Lineart 线稿 + 反推提示词。
- 用户提到「高清重绘」「重绘参考」「线稿 + 模糊 + 提示词」等关键词。

## 输入
- 一张原图（任意路径，记为 `$SRC`；输出记为 `$OUT` 前缀）。

## 环境与依赖
首次使用需安装（较重，需联网，约数分钟）：
```bash
pip install pillow numpy
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install controlnet_aux opencv-python-headless
```
线稿权重首次会从 HuggingFace 下载（`lllyasviel/Annotators`）。国内用镜像加速：
```bash
export HF_ENDPOINT=https://hf-mirror.com
```

## 步骤

### 第 1 步：高斯模糊 8（PS 等效）
Photoshop「高斯模糊 半径 8」的 radius 即高斯核标准差 σ；Pillow 的 `GaussianBlur(radius)` 的 radius 同样取 σ，二者完全等价。
```python
from PIL import Image, ImageFilter
img = Image.open($SRC).convert("RGB")
img.filter(ImageFilter.GaussianBlur(8)).save(f"{$OUT}_blur8.jpg", quality=95)
```

### 第 2 步：ControlNet Lineart 线稿（1024×1024）
使用 `LineartDetector`（标准 / 精细版）。`detect_resolution=1024` 是关键——默认 512 会丢失小结构（头饰纹样、角脊、细发丝）。
ControlNet 输出为黑底白线，反相得到白底黑线线稿。
```python
import numpy as np
from PIL import Image
from controlnet_aux import LineartDetector
img = Image.open($SRC).convert("RGB")
det = LineartDetector.from_pretrained("lllyasviel/Annotators")
out = det(img, detect_resolution=1024, image_resolution=1024)
a = np.array(out.convert("L"))
Image.fromarray((255 - a).astype("uint8"), "L").save(f"{$OUT}_lineart_1024.jpg", quality=95)
```
**避坑（已验证）：**
- 不要用 `AnimeLineartDetector`：对平滑肤色人脸区域响应极弱，人脸会空白，且背景易出瓷砖状伪影。
- 不要用 `HEDdetector` / `PidiNetDetector` 当干净线稿：它们输出软边缘概率图，线粗、糊、缺小结构。
- 务必 `detect_resolution=1024`，否则小结构被下采样吃掉。

### 第 3 步：从生成逻辑反推中文提示词
用视觉理解能力分析原图，从「若要用 AI 重新生成这张图，背后的生成逻辑是什么」出发，反推出一段结构化中文正向提示词（逗号分隔）。

**输出规范（严格）：**
- 只输出「可用于生图的提示词本身」。
- **不得**附带任何维度拆解、分析、负向提示词、解释或落地方向说明。
- 必须带以下前缀（这是本工作流的生图指令）：

> 请你根据以下提示词，还原图1线稿，并且根据图片2物体固有色进行上色，最后生图：<正向提示词>

其中「图1」为本工作流第 2 步产出的线稿图，「图片2」为用户输入的原图（取其物体固有色用于上色）。

### 第 4 步：交付
向用户交付三样：
1. 高斯模糊图（`${OUT}_blur8.jpg`）
2. Lineart 线稿图（`${OUT}_lineart_1024.jpg`）
3. 反推的中文提示词（保存为 `.md` 并展示，或直接对话给出；内容仅为「前缀 + 正向提示词」，不含其他）

## 备注
- 高斯模糊与线稿均为本地处理；线稿权重下载一次后缓存，后续秒级。
- 反推提示词依赖模型视觉理解，非代码步骤。
- 若用户后续要接 Stable Diffusion 出图，第 1 步模糊图可作 img2img 底图，第 2 步线稿可作 ControlNet Lineart 控制图，第 3 步提示词即为生成提示词。
