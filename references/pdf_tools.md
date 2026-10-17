# PDF 解析工具（PyMuPDF / fitz）——本 skill 与配套 paper-ppt skill 共用

> **用途**：从论文 PDF 提取正文文本、参考文献、图注，以及把矢量图渲染为 PNG。
> **环境**：`pip install pymupdf`（导入名 `fitz`）。
> **原则**：全文 PDF 的文本/图表提取一律用本文件的现成函数，不要每次即兴重写。

## 1. 提取正文文本

```python
import fitz

def extract_text(pdf_path):
    """返回逐页文本列表，每项为该页全部文字。"""
    doc = fitz.open(pdf_path)
    pages = [page.get_text() for page in doc]
    doc.close()
    return pages

# 用法
pages = extract_text("paper.pdf")
for i, t in enumerate(pages):
    print(f"--- 第 {i+1} 页 ---")
    print(t)
```

## 2. 提取文末参考文献列表

```python
import re

def extract_references(pdf_path, start_marker="References"):
    """粗略提取参考文献编号。返回 (编号, 起始位置) 列表。"""
    doc = fitz.open(pdf_path)
    refs = []
    collecting = False
    for page in doc:
        txt = page.get_text()
        if not collecting:
            if re.search(rf'^\s*{start_marker}\b', txt, re.MULTILINE):
                collecting = True
        if collecting:
            for m in re.finditer(r'\[(\d+)\]\s+', txt):
                refs.append((int(m.group(1)), m.start()))
    doc.close()
    return refs

# 更稳妥：直接打印含 References 的页面全文，供核对 [n] ↔ 作者/标题
def print_reference_pages(pdf_path):
    doc = fitz.open(pdf_path)
    for i, page in enumerate(doc):
        t = page.get_text()
        if "References" in t or "REFERENCES" in t:
            print(f"=== 第 {i+1} 页（含参考文献）===")
            print(t)
    doc.close()
```

> **核对方法**：正文出现 `[32]` 时，用 `print_reference_pages()` 找到文末列表，核对 `[32]` 对应的作者/标题是否与正文描述一致；找不到该编号则标注「原文未列明该参考文献」。

## 3. 定位图注（find_caption）

```python
import re

def find_caption(page, fig_no):
    """在 page 上定位 Fig.X. 开头的图注文本块，返回 (x0,y0,x1,y1,text) 或 None。

    只匹配"独立文本块"且以 Fig.X. 开头，不用 search_for（会误命中正文里的 "Fig. X shows…"）。
    """
    for b in page.get_text('blocks'):
        x0, y0, x1, y1, txt, bno, btype = b
        if btype == 0 and re.match(rf'^Fig\.\s*{fig_no}\.', txt.strip().replace(chr(10), ' ')):
            return (x0, y0, x1, y1, txt.strip())
    return None
```

## 4. 渲染矢量图为 PNG（render_figure）

```python
def render_figure(page, caption, out_path, dpi=200):
    """图注一定在图下方：以图注顶边为下界，向上裁出图区渲染为 PNG。"""
    r = page.rect
    top = 40
    bottom = min(caption[1] + 15, r.height - 20)
    clip = fitz.Rect(40, top, r.width - 40, bottom)
    pix = page.get_pixmap(matrix=fitz.Matrix(dpi/72, dpi/72), clip=clip)
    pix.save(out_path)
    return out_path
```

## 5. 提取嵌入位图（extract_images）

```python
import os

def extract_images(pdf_path, out_dir):
    """提取 PDF 内嵌位图（get_images 只能拿到位图，矢量图需走 find_caption + render_figure）。"""
    os.makedirs(out_dir, exist_ok=True)
    doc = fitz.open(pdf_path)
    out = []
    for pno in range(len(doc)):
        for idx, img in enumerate(doc[pno].get_images(full=True)):
            xref = img[0]
            base = doc.extract_image(xref)
            fn = f"image_{pno+1}_{idx+1}.{base['ext']}"
            path = os.path.join(out_dir, fn)
            with open(path, "wb") as f:
                f.write(base["image"])
            out.append({"path": path, "page": pno+1})
    doc.close()
    return out
```

## 6. 按百分比裁剪（crop_save）

```python
def crop_save(doc, page_idx, x0_pct, y0_pct, x1_pct, y1_pct, name, out_dir, dpi=280):
    page = doc[page_idx]
    r = page.rect
    clip = fitz.Rect(r.x0 + r.width*x0_pct, r.y0 + r.height*y0_pct,
                     r.x0 + r.width*x1_pct, r.y0 + r.height*y1_pct)
    pix = page.get_pixmap(matrix=fitz.Matrix(dpi/72, dpi/72), clip=clip)
    path = f"{out_dir}/{name}.png"
    pix.save(path)
    return path
```

## 7. 无视觉兜底（模型读不了图时）

当读取 PNG/JPEG 返回 `[Unsupported Image]`（当前会话无视觉能力）时：
1. 图表信息改用 **图注文字（`find_caption`）+ 正文对图的描述 + 正文/表格中的数字**；
2. 结论处标注「图为自动定位，未目视核对」；
3. 既不要因为看不到图就跳过图表维度，也不要凭空描述图内细节。
