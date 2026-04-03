# Python OCR 程序示例*
好的，这里给你一个 **Python OCR 程序示例**，使用常见的开源库 `pytesseract`（Tesseract OCR 的 Python 封装）和 `Pillow` 来处理图片。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import pytesseract
from PIL import Image
import sys
import os

def ocr_image(image_path):
    """
    对指定图片进行 OCR 识别，返回识别出的文本
    """
    if not os.path.exists(image_path):
        raise FileNotFoundError(f"文件不存在: {image_path}")

    # 打开图片
    img = Image.open(image_path)

    # 使用 Tesseract OCR 识别
    text = pytesseract.image_to_string(img, lang="chi_sim+eng")  # 支持中文和英文
    return text

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("用法: python ocr.py <图片路径>")
        sys.exit(1)

    image_path = sys.argv[1]
    try:
        result = ocr_image(image_path)
        print("识别结果:")
        print(result)
    except Exception as e:
        print(f"OCR 失败: {e}")
```
## 📌 使用说明
1. 安装依赖：
   ```bash
   sudo apt-get install tesseract-ocr libtesseract-dev
   pip install pytesseract pillow
   ```
   如果需要中文支持，还要安装中文语言包：
   ```bash
   sudo apt-get install tesseract-ocr-chi-sim
   ```
2. 保存脚本为 `ocr.py`。  
3. 执行：
   ```bash
   python ocr.py test.png
   ```
4. 输出结果就是识别出的文本。  
## ✅ 总结
- 这个程序可以识别 **中文 + 英文**，你可以根据需要调整 `lang` 参数。  
- 如果要批量识别，可以在 `main` 里循环多个图片路径。  
- 如果要识别 PDF，可以结合 `pdf2image` 把 PDF 转成图片再 OCR。  

# DeepSeekOCR
**DeepSeekOCR 是 DeepSeek 团队推出的深度学习 OCR 框架，主打高精度文档解析，支持 PDF、图片转 Markdown，具备多种识别模式（快速模式、表格解析模式、逐字识别模式），对标 PaddleOCR、EasyOCR 等开源库，同时也试图替代商业云服务如 Amazon Textract。**  [PyPI](https://pypi.org/project/deepseek-ocr/)  [deepwiki.com](https://deepwiki.com/deepseek-ai/DeepSeek-OCR/2.1-installation)  [skywork.ai](https://skywork.ai/blog/integrate-deepseek-ocr-python-step-by-step-tutorial/)  
## 📌 DeepSeekOCR 的核心特点
- **三种识别模式**  
  - **FREE_OCR**：快速模式，适合 80% 的常见场景。  
  - **GROUNDING**：复杂表格和版面解析模式。  
  - **OCR_IMAGE**：逐字识别，适合高精度需求。  
- **智能回退**：自动切换模式以保证最佳质量。  
- **批量处理**：支持多文档并行识别，带进度跟踪。  
- **异步/同步 API**：适合不同应用场景。  
- **输出格式**：直接生成 Markdown，方便后续结构化处理。  
## 📊 DeepSeekOCR 与其他 OCR 库对比
| 库名称         | 技术基础             | 优势 | 劣势 | 适用场景 |
|----------------|----------------------|------|------|----------|
| **DeepSeekOCR** | 深度学习 + Markdown 输出 | 高精度，支持复杂表格，输出结构化 | 需 GPU，依赖较多 | 合同、发票、PDF 文档 |
| **PaddleOCR**   | 飞桨深度学习框架     | 功能最全，支持检测+识别+版面分析 | 部署复杂 | 发票、合同、表格 |
| **EasyOCR**     | PyTorch              | 安装简单，支持 80+ 语言 | 精度略低，表格解析弱 | 多语言自然场景 |
| **DocTR**       | TensorFlow/Keras     | 文档版面分析，学术背景强 | 社区较小 | PDF、扫描文档 |
| **Tesseract**   | 传统 OCR 引擎        | 稳定、轻量、跨平台 | 精度低，复杂版面差 | 简单票据、扫描件 |
## 📌 安装与使用
### 安装
```bash
pip install deepseek-ocr
```
需要 **CUDA 11.8+** 和 **Python 3.12**，推荐 GPU 显存 ≥40GB（如 A100-40G）。  [deepwiki.com](https://deepwiki.com/deepseek-ai/DeepSeek-OCR/2.1-installation)  

### 使用示例
```python
from deepseek_ocr import DeepSeekOCR

ocr = DeepSeekOCR(mode="GROUNDING")  # 表格解析模式
result = ocr.process("invoice.pdf")

print("识别结果 Markdown:")
print(result)
```
## ⚠️ 风险与取舍
- **优势**：高精度、支持复杂文档结构化输出、Markdown 格式方便二次处理。  
- **劣势**：依赖 GPU，资源消耗大；社区相对新，生态尚在发展。  
- **适合场景**：企业级合同、发票、表格解析；科研文档结构化处理。  

✅ **总结**：DeepSeekOCR 对标 PaddleOCR、EasyOCR、DocTR 等开源库，同时也瞄准 Amazon Textract 等商业服务的替代方案。它的定位是 **高精度 + 文档结构化 + Markdown 输出**，适合需要复杂文档解析的场景。  

# MonkeyOCR
**MonkeyOCR 是一个新兴的深度学习 OCR 框架，由 Yuliang Liu 开发，主打轻量化、多模态文档解析，支持复杂版面分析和多语言识别。它比传统的 Tesseract/pytesseract 更先进，适合处理发票、合同、表格、自然场景文字等复杂任务。**  [Github](https://github.com/Yuliang-Liu/MonkeyOCR)  [deepwiki.com](https://deepwiki.com/Yuliang-Liu/MonkeyOCR/2.1-installation-and-setup)  [FileFormat](https://products.fileformat.com/ocr/python/monkeyocr/)  
## 📌 MonkeyOCR 的特点
- **基于深度学习**：采用 LMM（Large Multimodal Models）与文档解析模型（如 LayoutLMv3、PP-DocLayoutV2）。  
- **多语言支持**：可识别自然场景、表格、票据、身份证等多种文档。  
- **轻量化设计**：相比 PaddleOCR 更轻量，适合快速部署。  
- **GPU 加速**：支持 CUDA 11.8–12.8，推荐使用 NVIDIA RTX 30/40 系列或 V100。  
- **应用场景**：发票扫描、合同解析、身份证识别、PDF 转结构化数据。  
## 📊 与其他 OCR 库对比
| 库名称         | 技术基础         | 优势 | 劣势 | 适用场景 |
|----------------|------------------|------|------|----------|
| **pytesseract** | 传统 OCR 引擎    | 稳定、轻量、跨平台 | 准确率低，复杂版面差 | 简单文档、票据 |
| **EasyOCR**     | PyTorch 深度学习 | 多语言支持，识别率高 | 模型较大，速度一般 | 多语言场景 |
| **PaddleOCR**   | 飞桨深度学习框架 | 高精度，版面分析强 | 部署复杂，依赖多 | 发票、合同、表格 |
| **MonkeyOCR**   | LMM + 文档解析   | 轻量化，支持复杂文档结构 | 社区较新，生态尚在发展 | 合同、票据、自然场景 |
## 📌 安装与使用示例
### 安装
```bash
git clone https://github.com/Yuliang-Liu/MonkeyOCR.git
cd MonkeyOCR
pip install -r requirements.txt
```
### 使用示例
```python
from monkeyocr import OCR

ocr = OCR(model="pp-doclayoutv2")  # 选择模型
result = ocr.recognize("invoice.png")

print("识别结果:", result["text"])
```
## ⚠️ 风险与取舍
- **优势**：比 Tesseract 更智能，支持复杂文档结构化解析。  
- **劣势**：社区较新，生态和文档不如 PaddleOCR 完善。  
- **建议**：如果你要处理 **合同、发票、表格** 等复杂文档，MonkeyOCR 是一个值得尝试的新选择；如果只是简单 OCR，pytesseract 更轻量。  
