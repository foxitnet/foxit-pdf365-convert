# 更新日志（Changelog）

本文档记录 `foxit-pdf365-convert` 技能的版本变更信息。

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号与 `SKILL.md` 中 `metadata.version` 保持一致。

## [1.03.01.4575] - 2026-08-26

### 新增（Added）

- 新增 **图片转 PDF** 功能：支持将图片转换为 PDF（`.pdf`），支持多图片。
- 新增 **Word 转 PDF** 功能：支持将 Word 文档（`.docx` / `.doc`）转换为 PDF（`.pdf`）。
- 新增 **合并 PDF** 功能：支持将多个PDF文件合并成一个。
- 新增 **PDF 转 图片** 功能：支持将PDF转为图片。
- 新增 **PDF 转 Excel** 功能：支持将PDF转为Excel。

### 变更（Changed）

- 对 `SKILL.md` 进行内容拆分，将不同主题独立维护到单独文件，`SKILL.md` 保留主流程、接口与上传规则并引用这些文件：
  - **触发规则与 JobType 对照** → 独立维护到 [JobType_and_Triggers.md](./JobType_and_Triggers.md)；
  - **API Key 相关流程**（检查流程、切换账号/替换 Key、Key 无效处理、运维安全约束）→ 独立维护到 [API_KEY.md](./API_KEY.md)；
  - **MCP 配置说明**（配置、调用、运维安全约束）→ 独立维护到 [MCP.md](./MCP.md)。

## [1.02.01.a875] - 2026-08-12

### 新增（Added）

- 新增 **PDF 转 PPT** 功能：支持将 PDF 文件转换为 PowerPoint（`.pptx`）演示文稿。
- 新增 **压缩 PDF** 功能：支持对 PDF 文件进行压缩以减小体积。
- 新增 用户增加10次试用机会。

### 变更（Changed）

- API Key 存储位置改为**本技能目录**下的 `skill_key` 文件（与 `SKILL.md` 同级），不再使用全局 `~/.pdf365/skill_key`。

### 修复（Fixed）

- 读取 Key 时去除 UTF-8 BOM（用 `utf-8-sig` 解码或 `lstrip('\ufeff')`）与首尾空白，避免带 `\ufeff` 前缀的 Key 触发 HTTP 编码异常导致上传崩溃。

## [1.01.01.b39d] - 2026-07-31

### 新增（Added）

- 新增付费相关功能模块。


## [1.00.01] - 2026-07-06

### 新增（Added）

- 新增 **PDF 转 word** 功能，支持将 PDF 文件转换为 Word（`.docx`）。


## 版本约定

- 版本号在 `SKILL.md` 的 `metadata.version` 中维护，本次变更即在上方最新条目中体现。
- 新增功能、修改逻辑、修复问题均需在上方追加对应条目，并标注「新增 （Added） / 变更 （Changed）/ 即将移除 （Deprecated）/修复 （Fixed）/ 移除 （Removed）/ 安全（Security）」类别。