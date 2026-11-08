# high-res-redraw（高清重绘）

WorkBuddy skill：把一张原图预处理成「高清重绘」所需的三件套。

## 流程

1. **高斯模糊 8** —— PS 高斯模糊「半径 8」等效（标准差 σ = 8）。
2. **ControlNet Lineart 线稿** —— 1024×1024 提取，输出白底黑线干净线稿。
3. **反推中文提示词** —— 从生成逻辑反推，输出固定前缀 + 正向提示词（纯净，无分析/负向/解释）。

## 交付物

- 一张高斯模糊图（`*_blur8.jpg`）
- 一张 Lineart 线稿图（`*_lineart_1024.jpg`）
- 一段可用于生图的中文提示词（`*_reverse_prompt.md`，格式：`请你根据以下提示词，还原图1线稿，并且根据图片2物体固有色进行上色，最后生图：<正向提示词>`）

## 环境与依赖

```bash
pip install pillow numpy
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install controlnet_aux opencv-python-headless
```

首次运行会从 HuggingFace 下载线稿权重（`lllyasviel/Annotators`）。国内可用镜像加速：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

## 用途

三件套可直接用于 Stable Diffusion 高清重绘：模糊图作 img2img 底图，线稿作 ControlNet `lineart` 控制图，提示词作生成词。

## License

[MIT](LICENSE)
