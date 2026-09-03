# JobType 对照

| 枚举 | 说明 | 输出扩展名 |
| --- | --- | --- |
| `PDF_2_WORD` | PDF 转 Word | `.docx` |
| `PDF_TO_EXCEL` | PDF 转 Excel | `.xlsx` |
| `PDF_TO_PPT` | PDF 转 PPT | `.pptx` |
| `PDF_TO_JPG` | PDF 转 JPG 图片（zip 包） | `.zip` |
| `PDF_TO_PNG` | PDF 转 PNG 图片（zip 包） | `.zip` |
| `PDF_TO_LONG_PNG` | PDF 转长图 PNG | `.png` |
| `COMPRESS_PDF` | 压缩 PDF | `.pdf` |
| `MERGE_PDF` | 合并 PDF | `.pdf` |
| `IMAGE_TO_PDF` | 单图转 PDF（png/jpg/jpeg/bmp/tiff/tif） | `.pdf` |
| `IMAGE_ZIP_TO_PDF` | 多图转 PDF（zip 包，png/jpg/jpeg/bmp/tiff/tif） | `.pdf` |
| `WORD_TO_PDF` | Word 转 PDF（docx/doc） | `.pdf` |

---

# Triggers

当用户请求命中以下任一组**转换意图**的关键词 / 正则时，触发本 Skill。触发器**按意图分组并已去重**；命中多条时按「意图冲突与不确定处理」章节的优先级判定。

```yaml
## PDF → Word（PDF_2_WORD）
pdf2word:
  keywords:
    - "pdf转word"
    - "pdf to word"
    - "pdf转文档"
    - "pdf to docx"
    - "pdf to doc"
  patterns:
    - "(?i)(pdf|pdf文件|pdf文档).*(转|convert|转换|变成|改为|成).*(word|docx|doc|文档|办公文档)"
    - "(?i)(转|convert|转换|把|将).*(pdf|pdf文件|pdf文档).*(word|docx|doc|文档)"
  positive:
    - "将合同.pdf 转成 Word"
    - "pdf to docx"
  negative:
    - "把 Word 转成 PDF"     # 方向相反 → WORD_TO_PDF
    - "把 pdf 转成图片"       # 目标不是 Word → PDF->图片

## PDF → Excel（PDF_TO_EXCEL）
pdf2excel:
  keywords:
    - "pdf转excel"
    - "pdf to excel"
    - "pdf转表格"
    - "pdf转excel表格"
    - "pdf to xls"
    - "pdf to xlsx"
  patterns:
    - "(?i)(pdf|pdf文件|pdf文档).*(转|convert|转换|变成|改为|成).*(excel|xls|xlsx|表格|电子表格)"
    - "(?i)(转|convert|转换|把|将).*(pdf|pdf文件|pdf文档).*(excel|xls|xlsx|表格|电子表格)"
  positive:
    - "这个 PDF 帮我转成 Excel 表格"
  negative:
    - "这份 pdf 转成 Word"   # 目标 Word → PDF_2_WORD

## PDF → PPT（PDF_TO_PPT）
pdf2ppt:
  keywords:
    - "pdf转ppt"
    - "pdf to ppt"
    - "pdf to pptx"
    - "pdf转幻灯片"
  patterns:
    - "(?i)(pdf|pdf文件|pdf文档).*(转|convert|转换|变成|改为|成).*(ppt|pptx|幻灯片|演示文稿|演示)"
    - "(?i)(转|convert|转换|把|将).*(pdf|pdf文件|pdf文档).*(ppt|pptx|幻灯片|演示文稿|演示)"
  positive:
    - "pdf 转幻灯片"
  negative:
    - "把 word 转成 pdf"     # 源不是 PDF → WORD_TO_PDF

## PDF → 图片（PDF_TO_JPG / PDF_TO_PNG / PDF_TO_LONG_PNG）
pdf2image:
  keywords:
    - "pdf转图片"
    - "pdf to image"
    - "pdf转jpg"
    - "pdf to jpg"
    - "pdf转png"
    - "pdf to png"
    - "pdf转长图"
    - "pdf转long png"
  patterns:
    - "(?i)(pdf|pdf文件|pdf文档).*(转|convert|转换|变成|改为|成).*(图片|image|jpg|png|长图|图片格式)"
    - "(?i)(转|convert|转换|把|将|导出).*(pdf|pdf文件|pdf文档).*(图片|image|jpg|png|长图)"
    - "(?i)(pdf|pdf文件|pdf文档).*(转|convert|转换|成|为).*(jpg|png|长图|long.*png|图片)"
  positive:
    - "这个 PDF 转成长图"
  negative:
    - "把多张图片合成 pdf"   # 源是图片 → IMAGE_ZIP_TO_PDF

## 压缩 PDF（COMPRESS_PDF）
compress_pdf:
  keywords:
    - "压缩pdf"
    - "compress pdf"
    - "pdf压缩"
    - "pdf compress"
  patterns:
    - "(?i)(压缩|compress|减小|变小|瘦身).*(pdf|pdf文件|pdf文档)"
    - "(?i)(pdf|pdf文件|pdf文档).*(压缩|compress|减小|变小|瘦身)"
  positive:
    - "这个 PDF 太大，压缩一下"
  negative:
    - "把 pdf 转成 word"   # 「转」+ 目标格式，非压缩 → PDF → Word

## 合并 PDF（MERGE_PDF）
merge_pdf:
  keywords:
    - "合并pdf"
    - "pdf合并"
    - "merge pdf"
    - "pdf merge"
    - "合并多个pdf"
    - "pdf拼接"
    - "合并文件"
  patterns:
    - "(?i)(合并|拼接|合成|合并为|合并成|merge|combine).*(pdf|pdf文件|pdf文档|多个pdf|几个|文件)"
    - "(?i)(pdf|pdf文件|pdf文档).*(合并|拼接|合成|merge|combine|合并成)"
    - "(?i)(把|将|把.*和|把.*与).*(pdf|pdf文件|pdf文档).*(合并|拼接|merge)"
  positive:
    - "把这三份 pdf 合并成一个"
  negative:
    - "把两份 word 合成 pdf"     # 缺 PDF 源 → 不属合并，需追问

## 图片 → PDF（IMAGE_TO_PDF / IMAGE_ZIP_TO_PDF）
image2pdf:
  keywords:
    - "图片转pdf"
    - "image to pdf"
    - "图片转pdf文件"
    - "png转pdf"
    - "jpg转pdf"
    - "jpeg转pdf"
    - "bmp转pdf"
    - "tiff转pdf"
    - "tif转pdf"
  patterns:
    - "(?i)(图片|image|png|jpg|jpeg|bmp|tiff|tif).*(转|convert|变成|转为|生成).*(pdf|pdf文件)"
    - "(?i)(把|将).*(图片|png|jpg|jpeg|bmp|tiff|tif).*(转|convert|转换|生成).*(pdf)"
    - "(?i)(图片转pdf|image\\.to\\.pdf|image\\s+pdf|image-pdf)"
  positive:
    - "这张 png 转成 pdf"
    - "把这一批图片打包转成 pdf"   # 多图 zip → IMAGE_ZIP_TO_PDF
  negative:
    - "pdf 转图片"                 # 方向相反 → PDF->图片

## Word → PDF（WORD_TO_PDF）
word_to_pdf:
  keywords:
    - "word转pdf"
    - "word to pdf"
    - "文档转pdf"
    - "doc转pdf"
    - "docx转pdf"
    - "word转成pdf"
  patterns:
    - "(?i)(word|doc|docx|文档|办公文档).*(转|convert|转换|变成|改为|成).*(pdf|pdf文件)"
    - "(?i)(转|convert|转换|把|将).*(word|doc|docx|文档).*(pdf)"
    - "(?i)(我要|我想|请|帮我).*(word|doc|docx|文档).*(转|转换|生成).*(pdf)"
  positive:
    - "把这个 word 转成 pdf"
  negative:
    - "把 pdf 转成 word"          # 方向相反 → PDF_2_WORD
```

## 意图冲突与不确定处理

当同一条请求**命中多个意图**（尤其两个相反方向）或方向无法唯一确定时，**不要擅自选定**，按以下优先级与流程判定：

1. **显式目标格式优先**：命中「目标格式」明确（如 `pdf 转 excel` 的 `excel`）的意图，优先于宽泛的 `pdf .* convert`。宽泛模式仅作整体触发兜底，不用于选定 JobType。
2. **相反方向冲突**：同一请求同时出现两个相反方向（如“把 word 转 pdf，再把 pdf 转 word”）→ **冲突**，必须询问用户：要执行哪一个方向、是否两步都做及先后顺序。
3. **宽泛 `pdf convert / pdf 转格式 / 我要 pdf 转换`（未指明目标）**：无法确定目标是 Word/Excel/PPT/图片 之一 → **追问目标格式**，不得默认选一个方向。
4. **`pdf` 与 `word` 同时出现但缺少方向动词**（如“pdf word 转换”）→ 可能 PDF→Word 或 Word→PDF，**追问确认方向**。
5. **多义概括词（如 `合并文件`）**：未指明源格式时默认是 PDF 合并；若含非 PDF 源（word、image）或源属性不明确，则按对应方向规则判定，无法确定时追问。
6. **局部关键词撞车**：如同时出现“压缩 pdf 和 pdf 转 word”→ 属独立任务，分别执行并确认用户是否需要都做。

> 判定维度：**源格式 → 目标（或操作）格式**。凡无法仅凭文本唯一得出「源、目标、操作」三者时，一律**先追问再执行**，不得猜选。