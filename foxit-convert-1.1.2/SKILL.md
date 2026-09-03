---
name: foxit-pdf365-convert
description: PDF 文档格式转换（PDF转Word、PDF转Excel、PDF转PPT、PDF转图片、图片转PDF、Word转PDF、压缩PDF、合并PDF）
metadata:
  version: 1.03.01.4575
  author: Foxit
---
## JobType 对照

> 📌 JobType 对照与触发规则已独立维护在 **[JobType_and_Triggers.md](./JobType_and_Triggers.md)**。

---

## Triggers

> 📌 触发关键词与正则模式已独立维护在 **[JobType_and_Triggers.md](./JobType_and_Triggers.md)**（`Triggers` 章节）。

---

## API Key 管理（必须首先执行）

> 📌 API Key 相关流程（检查流程、切换账号/替换 Key、Key 无效处理）已独立维护在 **[API_KEY.md](./API_KEY.md)**。调用本 Skill 时请完整遵循该文件内容，本节不再赘述。

调用本 Skill 的**第一步**必须检查本地是否已存在 API Key。完整流程见 **[API_KEY.md](./API_KEY.md)**：

- **检查流程**：读取当前 Skill 目录内 `skill_key`，不存在则引导用户创建并保存。
- **Key 完整性**：Key 格式为 `PDF365-MCP-xxxxxxxx`，`PDF365-MCP-` 为不可分割前缀，严禁截断。
- **切换账号 / 替换 Key**：须经用户二次确认，并同步更新本地 Key 与 MCP 配置中的 `X-API-KEY`。
- **Key 无效处理**：上传或任一 MCP 工具返回鉴权失败时，立即停止并引导用户重新获取、更新两者并重载 MCP。
- **运维安全约束**：Key 需最小暴露、最小权限、可轮换、可撤销，且不得出现在日志、错误堆栈、命令历史或安装包中（见 [API_KEY.md](./API_KEY.md)「运维安全约束」与 [MCP.md](./MCP.md)「运维安全约束」）。

---

## 工作流

```text
用户触发 -> ensure_api_key -> ensure_mcp_ready -> ask_upload -> (合并PDF: ask_files -> 重命名排序 -> 打包zip -> upload -> converting / 转图片: ask_image_format -> converting / 图片转PDF: ask_image_format -> converting / 其它: converting) -> download -> (有额度: return_file / 无额度: prompt_pay -> download) / error_handler
```
| 状态 | 动作 | 成功去向 | 失败去向 |
|---|---|---|---|
| ensure_api_key | 检查本地 Key：不存在则引导用户创建；存在则读取（见 [API_KEY.md](./API_KEY.md)）。Key 有效性由后续 API 调用校验，无效时转 invalid_key | ensure_mcp_ready | end |
| ensure_mcp_ready | 检查/自动配置 MCP（见「MCP 调用模式」） | ask_upload | end |
| ask_upload | 等待用户提供文件并校验格式与大小：PDF 转换（转Word/Excel/PPT/图片/压缩/合并）仅支持 `.pdf`；图片转 PDF 场景仅支持 `.png`/`.jpg`/`.jpeg`/`.bmp`/`.tiff`/`.tif`；Word 转 PDF 仅支持 `.docx`/`.doc`；所有上传内容须 < 100MB（≥100MB 则提醒用户并停止，见「上传接口 → 文件大小校验」）。不合规则告知并请用户重新提供 | ask_files（合并）/ ask_image_format（转图片/图片转PDF）/ converting（其它） |  error_handler |
| ask_files | （合并专用）确认并收集用户提供的多个 PDF 文件及其合并顺序 | 重命名排序（步骤见「转换执行约定」→「执行流程」）/ converting | ask_files |
| ask_image_format | 两种情况之一：① 转图片：用户选择输出图片格式：jpg / png / 长图（仅限这三项）；② 图片转 PDF：校验用户提供的图片格式，仅支持 png / jpg / jpeg / bmp / tiff / tif | converting | ask_image_format |
| converting | 上传 → 创建任务 → 轮询（步骤 1-3），其中上传/创建任务若返回 Key 无效转 invalid_key | download | invalid_key / error_handler |
| invalid_key | Key 无效（仅当 API 返回鉴权失败时）→ 引导用户重新获取并更新（见 [API_KEY.md](./API_KEY.md) →「Key 无效处理」） | ask_upload | end |
| download | 调用 `downloadConvertResult(taskId)`（步骤 5）| return_file（有额度） / prompt_pay（无额度） | error_handler |
| prompt_pay | 提示用户打开支付页链接完成支付，支付完成后告知 agent（步骤 6） | download | error_handler |
| return_file | 返回下载地址 + 将文件下载到原文件所在目录（步骤 7） | end | - |
| error_handler | 询问是否重试；上传、转换阶段失败回到 `ask_upload`，下载/支付阶段失败回到 `download` |  ask_upload / download | end |

---

## 转换执行约定

### 上传接口

| 项 | 值 |
|---|---|
| URL | `POST https://open.pdf365.cn/documentConvert/upload` |
| 请求头 | `Content-Type: multipart/form-data`、`X-API-KEY: <本地存储的Key>` |
| 参数 | 文件（form-data field: `file`） |
| 成功响应 | `{ "data": { "fileUrl": "..." }, "code": 0, "msg": "success" }` |

> ⚠️ **文件大小校验（上传前必须执行，一条硬性门槛 + 两种预检口径）**：全流程涉及三种文件大小，必须区分清楚，避免「单个」「打包总合」「上传包」语义混杂。三种文件大小的判定关系如下：
>
> **三种限制（统一口径）**：
> - **① 单个源文件上限（预检）**：用户提供的**每个**源文件（单个 PDF / 单张图片 / 单个 Word）大小**必须 < 100MB**。用于普通单文件转换；多文件合并 / 多图打包时作为第一步预检。
> - **② 内部 ZIP 解压后总大小上限（预检）**：多文件合并 / 多图打包时，**参与打包的全部源文件合计大小建议 < 100MB**，用于提前预警；是否真正达标以 ③ 实测为准。
> - **③ 最终上传包硬性门槛（决定性）**：真正交给 `upload` 接口的内容（单个源文件，或 Agent 打包生成的 zip）的**实测大小必须严格 < 100MB**。**任何分支、任何情形，最终是否允许上传一律以 ③ 为准。**
>
> **三条界限的关系**：①② 是流程中的**预检**，只用来识别过大的文件并提前告知用户；**最终能否上传，一律只看 ③**。即便每个源文件都 < 100MB（① 通过），只要 Agent 打包 / 合计后 zip ≥ 100MB（③ 不通过），同样**拒绝上传**。
>
> **校验时机**：在**调用上传接口之前**校验最终待上传内容（即 ③ 的实测上传对象）的实际文件大小，不得跳过。打包 zip 场景在打包完成后、上传前校验 zip 实际大小。
> **超限处理**：凡最终上传内容 **≥ 100MB**，**立即停止上传并终止后续所有逻辑**（不调用 `upload`、不创建转换任务、不轮询），同时**明确提醒用户**超限并给出处理建议：
>   - 普通单文件 → 请用户压缩或更换更小的文件后重试；
>   - 合并 PDF / 多图打包 → 拆分合并、分批处理，或先压缩后再合并 / 再转换；
>   - 若为 zip 打包，校验的是**打包后 zip 的实测大小（即 ③）**，不是单个文件。
> - 只有 ③ 通过（< 100MB）才允许继续上传。
>
> **边界示例（各分支一致）**：
> - 单个源文件**恰好 100MB** → ① 与 ③ 均不通过，**拒绝上传**，需压缩或更换更小文件；
> - 多个源文件各自 < 100MB，但合计/打包后 zip **≥ 100MB** → **拒绝上传**，需拆分合并、分批处理或先压缩；
> - 多个源文件各自 < 100MB 且打包后 zip 实测 **< 100MB** → **允许上传**，正常继续。

> ⚠️ **文件格式校验（上传前必须执行）**：根据转换类型校验文件扩展名：
> - **普通转换（转 Word / Excel / PPT / 图片 / 压缩）**：**接受单个 PDF 文件**（扩展名为 `.pdf`）。用户只提供一个 PDF，直接上传该 PDF。
> - **合并 PDF（`MERGE_PDF`）**：**接受用户提供的多个 PDF 文件**（扩展名为 `.pdf`，单源文件 < 100MB，预检①；能否上传以打包 zip 实测为准③），但**上传的是由本 Skill 内部重命名、排序后打包生成的唯一一个 `.zip`**。ZIP 由 Agent 内部生成，用户无需提供 ZIP，也不接受用户直接提交的 ZIP。
> - **图片转 PDF（`IMAGE_TO_PDF` / `IMAGE_ZIP_TO_PDF`）**：仅支持 `.png` / `.jpg` / `.jpeg` / `.bmp` / `.tiff` / `.tif` 图片；单图直接上传；多图由 Agent 内部排序、重命名后打包成唯一一个 zip 上传。
> - **Word 转 PDF（`WORD_TO_PDF`）**：仅支持 Word 文档 `.docx` / `.doc`。
> 上传前须校验文件扩展名是否属于上述支持范围，若非支持格式，**不得上传**,应告知用户对应支持格式并停止本次流程。



> ⚠️ **ZIP 使用边界（防混淆）**：
> - **ZIP 仅用于两类场景**：`MERGE_PDF`（合并多个 PDF）与多图 `IMAGE_ZIP_TO_PDF`（多张图片转 PDF），且均由 Agent 在**内部**打包生成，上传给转换接口。
> - **普通转换（转 Word / Excel / PPT / 图片 / 压缩）绝不接受 ZIP**：若用户在此类场景提供 ZIP，拒绝并请其改为提供单个 PDF。若用户提供的 ZIP 是本地已打包的多个 PDF 而用户未明确要求「合并」，应向用户说明并引导其改用「合并 PDF」流程。
> - **用户侧一律只提供源文件**：合并场景提供多个 PDF、图片转 PDF 场景提供图片。**ZIP 只是 Agent 内部生成的传输层打包格式，不视为用户提供的源格式**，用户也可忽略 ZIP 的存在。

### MCP 工具调用

所有 MCP 工具调用均通过 MCP 端点（`https://open.pdf365.cn/mcp`）发起。

| 工具 | 参数 | 说明 |
|---|---|---|
| `createConvertTask` | `(fileUrl, jobType)` | 创建任务，返回 `data.taskId`。`jobType` 根据用户需求传入：`"PDF_2_WORD"`（转Word）、`"PDF_TO_EXCEL"`（转Excel）、`"PDF_TO_PPT"`（转PPT）、`"PDF_TO_JPG"`（转JPG）、`"PDF_TO_PNG"`（转PNG）、`"PDF_TO_LONG_PNG"`（转长图）、`"IMAGE_TO_PDF"`（单图转PDF）、`"IMAGE_ZIP_TO_PDF"`（多图zip转PDF）、`"COMPRESS_PDF"`（压缩PDF）、`"MERGE_PDF"`（合并PDF）、`"WORD_TO_PDF"`（Word转PDF） |
| `getConvertTaskStatus` | `(taskId)` | 轮询：`WAITING`/`RUNNING` 继续；`FINISH && success=true` 进入下一步；其他失败 |
| `downloadConvertResult` | `(taskId)` | 仅需 `taskId`。两种结果：① 用户有额度 → 返回 `downloadUrl`，下载成功；② 用户无额度 → 返回报文中携带支付页链接，提示用户打开链接支付，支付完成后再次调用本工具 |

### 执行流程

0. **API Key 检查** 读取本技能目录下 `skill_key`；不存在则引导用户前往 https://www.pdf365.cn/skill_key 创建 Key 并保存到本技能目录
1. **上传** `POST .../upload`（请求头携带 `X-API-KEY`）→ 提取 `data.fileUrl`。**合并 PDF 例外**：上传的是打包后的 zip，见下方「合并 PDF（MERGE_PDF）」分支。若上传返回 Key 无效/鉴权失败，转 [API_KEY.md](./API_KEY.md) →「Key 无效处理」流程
   > ⚠️ **上传前必须执行大小校验（见「上传接口 → 文件大小校验」）**：单文件转换校验源文件实际大小 < 100MB（预检①）；打包 zip 场景在上传前校验 **zip 实测大小 < 100MB（决定性 ③）**。超限则提醒用户并**停止上传与后续一切逻辑**。
2. **选择转换类型** 判断用户需求：
   - 若用户要**转图片**，在开始 converting 前必须询问用户选择图片格式（**仅允许**以下三类）：
     1. JPG 格式 → `jobType = "PDF_TO_JPG"`
     2. PNG 格式 → `jobType = "PDF_TO_PNG"`
     3. 长图（Long PNG）→ `jobType = "PDF_TO_LONG_PNG"`
     > ⚠️ 不支持其他图片格式，用户未明确选择时须引导其在上述三项中选择
   - 若用户要**图片转 PDF**，执行下方「图片转 PDF」分支
   - 若用户要**Word 转 PDF**，执行下方「Word 转 PDF」分支
   - 若用户要**合并 PDF**，执行下方「合并 PDF（MERGE_PDF）」分支
   - 其它需求直接按步骤 3 以下映射选择 `jobType`

**图片转 PDF**：将**一张或多张**图片合并转换为一个 PDF 文件。

① **确认源图片与格式** 收集用户提供的**一个或多个**图片文件
   > ⚠️ **仅支持以下图片格式**：**png / jpg / jpeg / bmp / tiff / tif**。其他格式（如 gif、webp、heic、svg 等）**不支持**，须告知用户并提供支持格式范围，请其重新提供
② **区分单图 / 多图，打包 zip**：
   - **单张图片** → `jobType = "IMAGE_TO_PDF"`，直接上传该图片
   - **多张图片** → `jobType = "IMAGE_ZIP_TO_PDF"`，需在本地完成以下排序、重命名与打包：
     1. **确认顺序** 向用户展示图片列表并请其按希望最终出现在 PDF 中的顺序依次排序；用户可按序号（1、2、3……）指定顺序，或直接按文件名自然顺序
        > 用户未指定顺序时，默认按文件名自然顺序排序
     2. **重命名** 根据确认的顺序将每张图片重命名为 `index_<序号>.<原图片后缀>`，序号从 `00` 开始、两位补零递增：排序第 1 张 → `index_00.png`、第 2 张 → `index_01.jpg`、第 3 张 → `index_02.bmp`……依次类推。后缀保留该图片的**原始扩展名**（如 `.png`/`.jpg`/`.jpeg`/`.bmp`/`.tiff`/`.tif`）
     3. **打包 zip** 将重命名后的所有图片放入同一目录，打包为**单一个** `.zip`，zip 内保持 `index_00`、`index_01`……的命名与顺序
   > ⚠️ 重命名仅改变文件名，不影响图片内容与转换后的 PDF 顺序
   > ⚠️ **上传前必须校验大小（见「上传接口 → 文件大小校验」）**：单张图片须 < 100MB（预检①）；多图打包 zip 场景校验 **zip 实测大小 < 100MB（决定性 ③）**。≥100MB 则提醒用户并**停止上传与后续逻辑**
③ **上传** `POST .../upload`（携带 `X-API-KEY`）上传单张图片或 zip → 提取 `data.fileUrl`，继续执行步骤 3（创建任务，`jobType = "IMAGE_TO_PDF"` 或 `"IMAGE_ZIP_TO_PDF"`）

**Word 转 PDF（`jobType = "WORD_TO_PDF"`）**：将 Word 文档转换为 PDF 文件。

① **确认源文件与格式** 收集用户提供的 Word 文档
   > ⚠️ **仅支持以下格式**：**.docx / .doc**。其他格式（如 txt、wps、rtf 等）**不支持**，须告知用户并提供支持格式范围，请其重新提供
② **上传** `POST .../upload`（携带 `X-API-KEY`）上传该 Word 文档。**上传前校验大小 < 100MB**，≥100MB 则提醒用户压缩后再试并停止本次上传（见「上传接口 → 文件大小校验」）→ 提取 `data.fileUrl`，继续执行步骤 3（创建任务，`jobType = "WORD_TO_PDF"`）

**合并 PDF（`jobType = "MERGE_PDF"`）**：合并是**多文件**操作，与其它单文件转换流程不同。**用户只需提供多个 PDF 源文件**，本 Skill 在内部完成文件的**重命名、排序与打包成 zip**，再上传：

① **确认源文件与顺序** 收集用户提供的**两个及以上** PDF 文件（`.pdf`，单源文件 < 100MB，预①；多源合计与最终打包能否通过以 zip 实测为准，见「文件大小校验」），向用户展示文件列表并请其按希望合并的顺序依次排序；用户可按序号（1、2、3……）指定顺序，或直接按文件名自然顺序
   > ⚠️ 用户未指定顺序时，默认按文件名自然顺序排序。若用户只提供一个 PDF，则无需合并，直接提示用户无需合并
② **重命名** 根据确认的顺序将每个 PDF 重命名为 `index_<序号>.pdf`，序号从 `00` 开始、两位补零递增：第 1 个 → `index_00.pdf`、第 2 个 → `index_01.pdf`、第 3 个 → `index_02.pdf`……依次类推
   > ⚠️ 重命名仅改变文件名，不影响文件内容与合并后的 PDF 顺序
③ **打包 zip**（Agent 内部完成）将重命名后的所有 PDF 放入同一目录，打包为**单一个** `.zip`，zip 内保持 `index_00`、`index_01`……的命名与顺序
   > ⚠️ 该 zip 由 Agent 内部生成，**仅供 `MERGE_PDF` 上传使用**；用户无需也不应提供 ZIP。上传的必须是这一个 zip 包，一次性调用 `createConvertTask`，不要对每个 PDF 分别创建任务
   > ⚠️ **上传前必须校验大小（见「上传接口 → 文件大小校验」）**：多源合计为预检②，最终校验 **zip 实测大小 < 100MB（决定性 ③）**。≥100MB 则提醒用户拆分合并、分批处理或先压缩后再合并，并**停止上传与后续逻辑**
④ **上传 zip** `POST .../upload`（携带 `X-API-KEY`）上传 Agent 打包生成的该 zip → 提取 `data.fileUrl`，继续执行步骤 3（创建任务，`jobType = "MERGE_PDF"`）

3. **创建任务** 根据所选 `jobType`，调用 `createConvertTask(fileUrl, jobType)` → 提取 `data.taskId`
   - PDF 转 Word → `jobType = "PDF_2_WORD"`
   - PDF 转 Excel → `jobType = "PDF_TO_EXCEL"`
   - PDF 转 PPT → `jobType = "PDF_TO_PPT"`
   - PDF 转 JPG → `jobType = "PDF_TO_JPG"`
   - PDF 转 PNG → `jobType = "PDF_TO_PNG"`
   - PDF 转长图 → `jobType = "PDF_TO_LONG_PNG"`
   - 压缩 PDF → `jobType = "COMPRESS_PDF"`
   - 合并 PDF → `jobType = "MERGE_PDF"`（走上方「合并 PDF（MERGE_PDF）」分支）
   - 图片转 PDF → `jobType = "IMAGE_TO_PDF"`（单图）或 `"IMAGE_ZIP_TO_PDF"`（多图，走上方「图片转 PDF」分支）
   - Word 转 PDF → `jobType = "WORD_TO_PDF"`（走上方「Word 转 PDF」分支）
4. **轮询** `getConvertTaskStatus(taskId)` 直到 `state=FINISH && success=true`（间隔 3000ms，最多重试 60 次，60次后视为失败）
5. **下载结果** `downloadConvertResult(taskId)`（仅传 `taskId`），根据返回结果分两种情况：
   - **情况一：用户有额度** → 接口返回 `downloadUrl`，进入步骤 7
   - **情况二：用户无额度** → 接口返回报文中携带支付页链接，进入步骤 6
6. **引导支付** 从返回报文中提取支付页链接，提示用户打开该链接完成支付；**等待用户告知支付完成**后，再次调用 `downloadConvertResult(taskId)` 进行下载：
   - 下载成功（返回 `downloadUrl`）→ 进入步骤 7
   - 仍提示无额度 → 重复本步骤，直到下载成功
7. **返回文件** 获取到 `downloadUrl` 后，执行以下两个动作，并在最终回复中**同时输出两项内容，缺一不可**：
   - **① 下载链接**：将 `downloadUrl` 完整展示给用户
   - **② 本地保存路径**：将文件下载到原 PDF 文件所在目录，根据转换类型命名：
     - PDF 转 Word：原文件名去掉 `.pdf` + `.docx`（如 `报告.docx`）
     - PDF 转 Excel：原文件名去掉 `.pdf` + `.xlsx`（如 `报告.xlsx`）
     - PDF 转 PPT：原文件名去掉 `.pdf` + `.pptx`（如 `报告.pptx`）
     - PDF 转 JPG：原文件名去掉 `.pdf` + `_jpg.zip`（如 `报告_jpg.zip`）
     - PDF 转 PNG：原文件名去掉 `.pdf` + `_png.zip`（如 `报告_png.zip`）
     - PDF 转长图：原文件名去掉 `.pdf` + `_long.png`（如 `报告_long.png`）
     - 压缩 PDF：原文件名去掉 `.pdf` + `_compressed.pdf`（如 `报告_compressed.pdf`）
     - 合并 PDF：下载结果为一个合并后的 PDF（内含多个 PDF），保存为 `合并_<时间戳>.pdf`（如 `合并_20260807.pdf`），或用户指定命名
     - 图片转 PDF：下载结果为合并后的 PDF，保存到图片所在目录，命名为 `图片转PDF_<时间戳>.pdf`（如 `图片转PDF_20260807.pdf`），多图时可用 `转换结果_<时间戳>.pdf`，或用户指定命名
     - Word 转 PDF：原文件名去掉 `.docx`/`.doc` + `.pdf`（如 `报告.docx` → `报告.pdf`），下载到原 Word 文件所在目录
     并在回复中展示本地保存的**绝对路径**（如 `C:\Users\xxx\Desktop\报告.pdf`）

> ⚠️ 最终回复必须同时包含「下载链接」和「本地保存绝对路径」两项，不得只输出其中一项。
>
> ⚠️ 无额度场景下，必须在用户确认支付完成后再次调用 `downloadConvertResult(taskId)` 获取最终下载地址；`downloadConvertResult` 始终只传 `taskId`，不需要订单号等额外参数。
>
> ⚠️ 步骤 7 中「下载到本地文件目录」为必执行步骤：例如原文件为 `~/Desktop/报告.pdf`，PDF转Word保存为 `~/Desktop/报告.docx`，PDF转PPT保存为 `~/Desktop/报告.pptx`，压缩PDF保存为 `~/Desktop/报告_compressed.pdf`；图片转PDF则下载到原图片所在目录，Word转PDF则下载到原Word文件所在目录，保存为同名 `.pdf`。若无法确定原文件路径（如用户通过 URL 提供），则下载到本地桌面保存。

---
## 无额度支付引导

`downloadConvertResult(taskId)` 在用户无额度时，返回报文中会携带支付页链接，关键字段：

| 字段 | 说明 |
|---|---|
| 支付页链接 | 提示用户在浏览器中打开该链接完成支付（具体字段名以接口实际返回为准） |

### 异常处理

| 场景 | 处理 |
|---|---|
| 用户提供非支持格式文件（非 PDF，图片转 PDF 时非 png/jpg/jpeg/bmp/tiff/tif，或 Word 转 PDF 时非 docx/doc） | 告知对应支持格式范围，不发起上传，请用户重新提供支持格式的文件 |
| 转换失败（`success=false` 或失败状态） | 告知用户失败原因，询问是否重试；同意则重新上传转换，拒绝则结束 |
| 用户支付后仍返回无额度 | 提示用户确认支付是否成功，确认后再次调用 `downloadConvertResult(taskId)` |
| 用户拒绝支付 | `error_handler`，询问重试 |
| `MCP_CONVERT_RESPONSE_MISSING_DATA` | 服务端异常，联系维护者 |

---
## MCP 配置

> 📌 完整 MCP 配置说明在文件 **[MCP.md](./MCP.md)**中，请阅读该文件获取全部细节。

本 Skill 依赖必需 MCP Server **`foxit-pdf365-mcp-server`**。MCP 配置与调用包含三部分，均见 `MCP.md`：

1. **客户端配置示例**：VS Code / qclaw / WorkBuddy / Claude Code / Codex / OpenCode / Cursor 的 `json` 配置片段。
2. **调用模式**：优先调用已配置的 MCP 工具；未配置时**先自动配置**，自动配置失败才启用 **curl 兜底**。
3. **curl 兜底流程**：初始化会话获取 `Mcp-Session-Id` → 初始化通知 → `tools/list` 查看 schema → `tools/call` 调用工具。

> ⚠️ 自动配置 MCP 时，应将 **当前 Skill 目录内**的 `skill_key` 中的**完整** Key（含 `PDF365-MCP-` 前缀）原样填入 `headers["X-API-KEY"]`，不得截断或去除前缀。
