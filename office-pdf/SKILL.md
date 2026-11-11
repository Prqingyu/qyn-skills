---
name: office-pdf
description: 读取 PDF —— 提取文字与页数；扫描件则把页面渲染成图片再按图片读。当 PDF 是输入、需要里面的文字/表格/日期/提交方式时使用。加载本技能后再动手。
---

# PDF 读取

`read` 工具**读不了 PDF**（二进制）。按下面顺序选提取器，用第一个能用的。

## 1. 先探一下哪个能用

```bash
python -c "import pymupdf"    # 或  python -c "import pypdf"
where pdftotext               # poppler
```

优先用 `load_workspace_dependencies` 返回的**捆绑 Python**。若它没有 PDF 库，退到
`pdftotext`；仍没有，最后才用**系统 Python 的 PyMuPDF** —— 后者是**明确例外**
（捆绑运行时不含 PDF 库时，它常是本机唯一能读 PDF 的东西）；用了就在回复里说明。

## 2. 抽文字

```python
import pymupdf                      # 若只有 pypdf，用下面的替代
doc = pymupdf.open("in.pdf")
print("页数", doc.page_count)
for i, page in enumerate(doc, 1):
    print(f"--- p{i} ---")
    print(page.get_text())
```

```python
# 替代：pypdf（纯 Python，不能渲染）
from pypdf import PdfReader
for i, page in enumerate(PdfReader("in.pdf").pages, 1):
    print(f"--- p{i} ---")
    print(page.extract_text())
```

## 3. 扫面件（抽不到文字）

`get_text()` 返回空 ⇒ 那是图片。渲染成 PNG 再用 `read_image` 当图片读：

```python
page.get_pixmap(dpi=150).save(f"p{i}.png")
```

## 边界

- **一个提取器都没有** ⇒ 如实说"读不了 PDF"，**不要**凭文件名或上下文编内容。
- 表格 / 多栏容易串行：**关键数字（日期、金额、学号、比例）要回原文核对**再引用。
- 抽到的文字若乱码，多半是编码或子集字体问题 —— 换 `pdftotext -layout` 或渲染成图再读。
- **只读**：不要把 PDF 转存成其他文件写进工作区，除非用户要。
