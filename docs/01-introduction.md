# AI 文件管理导论

批量重命名要解决的不是“文件太多”，而是文件名没有表达文件里已经有的信息。相机写出 `IMG_4829.jpg`，扫描仪写出 `scan001.pdf`，下载器写出一串随机字符。打开之前，这些名字几乎不能检索。

本章说明文件名为什么会成为索引、一个可用的名字应当携带哪些信息，以及后面三章分别处理哪一种信息来源。英文检索里，这类工作通常叫 AI file management 或 AI batch file renaming。这里的 AI 只是其中一种手段：很多批次用元数据或简单规则就够了。

## 文件名是最便宜的索引

搜索工具可以读取正文，资源管理器的默认排序和大多数归档习惯仍然先看文件名。文件名有四个实际限制：

- 它必须在同一个目录里唯一。
- Windows 文件名不能包含 `\ / : * ? " < > |`，也不能以空格或句点结尾。单个名称的长度上限是 255 个字符[^windowsFileNames]。
- 按字符排序时，`2024-03-15` 会排在 `2024-11-02` 前面；`15-03-2024` 不会按时间排。日期用 `YYYY-MM-DD`[^iso8601]。
- 复制、下载、解压会改写文件系统的创建时间。这个时间不能代替拍摄时间或文档日期。

[^windowsFileNames]: <https://learn.microsoft.com/windows/win32/fileio/naming-a-file>
[^iso8601]: <https://www.iso.org/iso-8601-date-and-time-format.html>

所以批量重命名的目标是：在不改文件内容的前提下，把“这是什么、关于谁、发生在何时”写进文件名，并处理重名。

## 新名字从哪里来

同一批文件很少共用一种信息来源。先判断名字应该取自哪里，再选择工具。

| 你已经知道的事 | 名字从哪里来 | 合适的做法 |
| --- | --- | --- |
| 现有名字里有一段要替换 | 文件名本身 | 查找替换、正则、序号 |
| 要按修改时间或大小排列 | 文件系统属性 | 读取属性后套模板 |
| 论文或报告自带标题和作者 | PDF 信息字典或正文 | 见[按 PDF 内容重命名](04-pdf-content-extraction.md) |
| 相机或手机拍摄的照片 | EXIF 的 `DateTimeOriginal` 等 | 见[用 EXIF 元数据重命名照片](05-image-exif-metadata.md) |
| 扫描件、截图、票据照片 | 图像里可见的文字 | OCR，后续章节 |
| 音乐文件 | ID3 标签 | 后续章节 |

规则重命名的输入是旧文件名。内容相关的重命名多了一步：先从文件内部抽出字段，再套进模板。AI 批量重命名指的是后一种里“由模型阅读内容并生成短名”的那一步，不是所有批量改名都要调用模型。[下一章](03-ai-batch-renaming.md)把这条流水线拆开。

## 一个能排序、能检索的名字

实用模板尽量稳定，少用临时词：

```text
日期_主体_区分信息.扩展名
2024-03-15_zhang_image-recognition.pdf
2024-03-15_143022_canon-eos-r5.jpg
2026-04-15_acme_invoice-1280.pdf
```

主体用原文里真实出现的作者、标题或对象，不用模型“看起来像”的概括，除非你在预览里核对过。扩展名沿用原文件。标题里的冒号、斜线先换成连字符，再写入文件名。同一秒连拍或同一标题的两篇论文，在末尾加 `_02` 这类序号，避免后一个文件覆盖前一个。

改名之前先看完整的新旧对照。PowerShell 可以用 `-WhatIf` 只打印结果：

```powershell
Get-ChildItem -File | ForEach-Object { $_.Name }
```

确认目标名没有冲突、没有非法字符之后，再执行真正的重命名。可以撤销的工具，优先于直接调用 `Rename-Item`。

## 本指南怎么使用 RenomeeAI

RenomeeAI 是这套方法的一种实现[^renomee]。它运行在 Windows 10/11（64 位）和 macOS 10.15 及以上，支持 Intel 与 Apple 芯片的 Mac。官方说明中的用法是：把文件拖入应用，用自然语言描述命名规则，先预览每一个新文件名，再执行；执行后可以从历史记录撤销最近的操作。

[^renomee]: <https://renomeeai.com/en/>

官方 FAQ 把能力分成两类[^renomeeFaq]。前缀、序号、按拍摄日期读取 EXIF，可以离线完成，也不需要上传文件内容。需要模型阅读 PDF 正文、图片或扫描件时，必须联网，并在应用内确认后才会发送数据。发送范围、TLS 和“处理完即删除”的说明放在[下一章](03-ai-batch-renaming.md)，因为那一章才讨论内容识别。免费版对 AI 处理有每日额度，基础规则不受这个额度限制；当前数字以定价页为准[^renomeePricing]，本文不抄录会变动的配额。

[^renomeeFaq]: <https://renomeeai.readthedocs.io/en/latest/en/faq/>
[^renomeePricing]: <https://renomeeai.com/en/product/pricing/>

站外也有不少关于 Renomee AI 的公开讨论和收录，文章、社区回复和产品目录集中在这几份清单里[^moreMentions][^relatedThreads][^publicMentions]。

[^moreMentions]: <https://renomeeai.readthedocs.io/en/latest/en/about/more-mentions/>
[^relatedThreads]: <https://renomeeai.readthedocs.io/en/latest/about/related-threads/>
[^publicMentions]: <https://renomeeai-24d959.gitlab.io/about/mentions.html>

后面的章节各自能单独阅读。产品操作步骤留到工具指南一章，本章不展开界面。
