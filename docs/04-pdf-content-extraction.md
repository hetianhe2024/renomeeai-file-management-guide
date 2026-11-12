# 按 PDF 内容重命名

Automatically rename PDF files 的难点不在“批量”，而在每个文件的名字来源不同。资源管理器里选中 50 个 PDF 再按 F2，得到的是 `Invoice (1).pdf`、`Invoice (2).pdf`：同一个主干加上序号。论文、合同和发票需要的是彼此不同的名字。名字可以来自 PDF 自带的信息字典、页面上的文本，或扫描图像的 OCR 结果。三种来源要分开处理。

## PDF 里实际有两套书目信息

经典位置是文档信息字典（Document Information Dictionary）[^pdf32000]，常见键为：

[^pdf32000]: <https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf>

| 键 | 含义 | 拿来当文件名时的问题 |
| --- | --- | --- |
| `/Title` | 标题 | 经常为空，或等于导出前的 Word 文件名 |
| `/Author` | 作者 | 可能是软件用户名，不是论文作者 |
| `/Subject` | 主题 | 往往空白 |
| `/Keywords` | 关键词 | 适合检索，通常太长 |
| `/Creator` | 创建该文档的程序 | 如 Microsoft Word，不能当标题 |
| `/Producer` | 生成 PDF 的程序 | 如某一款 PDF 库 |
| `/CreationDate` | 创建时间 | 格式为 `D:YYYYMMDDHHmmSS`，不是页面上的出版年 |

较新的文件还会带 XMP 元数据，标题和作者可能写在 `dc:title`、`dc:creator`。只读信息字典的工具，会和只读 XMP 的工具得到不同结果。重命名前看你的工具读的是哪一套。

信息字典为空时，标题通常仍印在第一页。那就要抽取文本，而不是再查一次元数据。

## 先看信息字典，再看首页

下面用 pypdf 读取标题和首页文本[^pypdf]。它适合检查一批文件“有没有现成标题”，不是生产环境的唯一选择。Poppler 的 `pdftotext` 对复杂排版往往更稳[^pdftotext]。

[^pypdf]: <https://pypdf.readthedocs.io/>
[^pdftotext]: <https://poppler.freedesktop.org/>

```python
from pypdf import PdfReader

reader = PdfReader("paper.pdf")
meta = reader.metadata
title = (meta.title or "").strip() if meta else ""
first_page = reader.pages[0].extract_text() or ""
print(title)
print(first_page[:500])
```

命令行只抽首页：

```text
pdftotext -f 1 -l 1 -layout paper.pdf -
```

`-layout` 尽量保留栏位。双栏论文如果不用它，作者和标题可能粘在同一行。

判断顺序：

1. `/Title` 非空，且不是 `Microsoft Word - xxx` 这类导出残留，就用它。
2. 标题为空或明显是软件名，就从首页文本里取标题行。学术 PDF 的标题多在第一页上部，作者在标题下方。
3. `extract_text()` 几乎为空，这是扫描件或纯图片 PDF。文本层不存在，继续抽取没有帮助，需要 OCR。
4. 文件加密且没有打开口令时，两种读取都会失败。这种文件应跳过并列入报告，不要生成一个看起来像标题的名字。

## 从字段到文件名

学术文件常用：

```text
年份_第一作者姓_标题关键词.pdf
2024_zhang_image-recognition-survey.pdf
```

年份优先用页面或信息字典里的出版年。`/CreationDate` 是 PDF 生成时间，预印本和正式出版年经常不同。作者字段有多人时，文件名里放第一作者，避免把整行作者都放进去。

发票和合同更适合：

```text
日期_对方名称_单据类型.pdf
2026-04-15_acme_invoice.pdf
```

金额、合同号只有在 OCR 或文本里原样出现时才写入文件名。

写入前做一次规范化：

- 删除 `\ / : * ? " < > |`。
- 连续空白改成一个连字符或下划线。
- 去掉结尾的空格和句点。
- 标题截到大约 80 个字符，保留扩展名 `.pdf`。
- 目标名已存在时追加 `_02`，不要覆盖。

同一标题的附录、补充材料和正文会撞名。撞名是数据问题，序号只是区分手段。预览里要能看出哪两个源文件撞在一起。

## 扫描件

扫描仪输出的 PDF 每一页是一张图。信息字典通常只有扫描软件的名字，`extract_text()` 为空。这时“按内容重命名”等于：对前几页做 OCR，再从识别结果里找日期、供应商、文号。

OCR 会把 `0` 和 `O`、逗号和小数点搞混。金额和证件号要回看图像或预览，不能只看生成的文件名。识别语言要包含文档实际使用的文字；中英混排的发票如果只用英文模型，中文机构名会丢。

功能页把 50MB 以内的 PDF 标成可以读取标题、作者和正文，扫描件另走 OCR[^renomeeFeatures]。官方 FAQ 写的是另一条边界：一旦属于按内容命名，要在应用内确认后把前 3 页送到 AI 服务，而不是上传整个文件[^renomeeFaq]。两处说法都在，操作时以应用里当时的确认提示为准。标题如果出现在第 4 页以后，这次请求不会看到它。这个限制应当写进你的操作预期：封面和题名页要放在前 3 页，否则应先用信息字典或手工指定。不同文件要不同名字的几种做法，官网有一篇对照[^renameMultiplePdfs]。

[^renomeeFeatures]: <https://renomeeai.com/en/product/features/>
[^renomeeFaq]: <https://renomeeai.readthedocs.io/en/latest/en/faq/>
[^renameMultiplePdfs]: <https://renomeeai.com/en/blog/rename-multiple-pdf-files-different-names/>

只替换文件名里已有的年份时，仍然用预览命令，不需要打开 PDF：

```powershell
Get-ChildItem -Filter "*.pdf" |
  Rename-Item -NewName { $_.Name -replace '^2024_', '2025_' } -WhatIf
```

## 一批论文的前后对比

```text
下载名                              信息字典 /Title              新文件名
arxiv_2403.01234.pdf               Image Recognition Survey    2024_zhang_image-recognition-survey.pdf
Microsoft Word - final.docx.pdf    （空，或等于临时文件名）      从首页文本取标题后再命名
scan_0008.pdf                      （无文本层）                 OCR 后再命名，识别失败则保持原名
```

第三条保持原名，比生成一个无法核对的标题更安全。批量任务的报告里应留下“未改名”的原因：无文本、加密、标题冲突或 OCR 为空。
