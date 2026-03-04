---
name: feishu-doc
description: |
  Feishu document read/write operations. Activate when user mentions Feishu docs, cloud docs, or docx links.
---

# Feishu Document Tool

Single tool `feishu_doc` with action parameter for all document operations, including table creation for Docx.

## Token Extraction

From URL `https://xxx.feishu.cn/docx/ABC123def` → `doc_token` = `ABC123def`

## Actions

### Read Document

```json
{ "action": "read", "doc_token": "ABC123def" }
```

Returns: title, plain text content, block statistics. Check `hint` field - if present, structured content (tables, images) exists that requires `list_blocks`.

### Write Document (Replace All)

```json
{ "action": "write", "doc_token": "ABC123def", "content": "# Title\n\nMarkdown content..." }
```

Replaces entire document with markdown content. Supports: headings, lists, code blocks, quotes, links, images (`![](url)` auto-uploaded), bold/italic/strikethrough.

**Limitation:** Markdown tables are NOT supported.

### Append Content

```json
{ "action": "append", "doc_token": "ABC123def", "content": "Additional content" }
```

Appends markdown to end of document.

### Create Document

```json
{ "action": "create", "title": "New Document", "owner_open_id": "ou_xxx" }
```

With folder:

```json
{
  "action": "create",
  "title": "New Document",
  "folder_token": "fldcnXXX",
  "owner_open_id": "ou_xxx"
}
```

**Important:** Always pass `owner_open_id` with the requesting user's `open_id` (from inbound metadata `sender_id`) so the user automatically gets `full_access` permission on the created document. Without this, only the bot app has access.

### List Blocks

```json
{ "action": "list_blocks", "doc_token": "ABC123def" }
```

Returns full block data including tables, images. Use this to read structured content.

### Get Single Block

```json
{ "action": "get_block", "doc_token": "ABC123def", "block_id": "doxcnXXX" }
```

### Update Block Text

```json
{
  "action": "update_block",
  "doc_token": "ABC123def",
  "block_id": "doxcnXXX",
  "content": "New text"
}
```

### Delete Block

```json
{ "action": "delete_block", "doc_token": "ABC123def", "block_id": "doxcnXXX" }
```

### Create Table (Docx Table Block)

```json
{
  "action": "create_table",
  "doc_token": "ABC123def",
  "row_size": 2,
  "column_size": 2,
  "column_width": [200, 200]
}
```

Optional: `parent_block_id` to insert under a specific block.

### Write Table Cells

```json
{
  "action": "write_table_cells",
  "doc_token": "ABC123def",
  "table_block_id": "doxcnTABLE",
  "values": [
    ["A1", "B1"],
    ["A2", "B2"]
  ]
}
```

### Create Table With Values (One-step)

```json
{
  "action": "create_table_with_values",
  "doc_token": "ABC123def",
  "row_size": 2,
  "column_size": 2,
  "column_width": [200, 200],
  "values": [
    ["A1", "B1"],
    ["A2", "B2"]
  ]
}
```

Optional: `parent_block_id` to insert under a specific block.

### Upload Image to Docx (from URL or local file)

```json
{
  "action": "upload_image",
  "doc_token": "ABC123def",
  "url": "https://example.com/image.png"
}
```

Or local path with position control:

```json
{
  "action": "upload_image",
  "doc_token": "ABC123def",
  "file_path": "/tmp/image.png",
  "parent_block_id": "doxcnParent",
  "index": 5
}
```

Optional `index` (0-based) inserts the image at a specific position among sibling blocks. Omit to append at end.

**Note:** Image display size is determined by the uploaded image's pixel dimensions. For small images (e.g. 480x270 GIFs), scale to 800px+ width before uploading to ensure proper display.

### Upload File Attachment to Docx (from URL or local file)

```json
{
  "action": "upload_file",
  "doc_token": "ABC123def",
  "url": "https://example.com/report.pdf"
}
```

Or local path:

```json
{
  "action": "upload_file",
  "doc_token": "ABC123def",
  "file_path": "/tmp/report.pdf",
  "filename": "Q1-report.pdf"
}
```

Rules:

- exactly one of `url` / `file_path`
- optional `filename` override
- optional `parent_block_id`

## Reading Workflow

1. Start with `action: "read"` - get plain text + statistics
2. Check `block_types` in response for Table, Image, Code, etc.
3. If structured content exists, use `action: "list_blocks"` for full data

---

## ⚠️ 重要：创建文档的正确工作流（必读）

### 已知问题

1. **`write` 和 `append` API 经常返回 400 错误** - 不要依赖这两个动作来写入内容
2. **`create_table_with_values` 返回 400 错误但会创建空表格** - 需要手动填充单元格
3. **`update_block` 更新 Page 根块会把内容当作标题** - 必须更新具体的 Text 块

### ✅ 正确的文档创建流程

```
步骤 1: 创建文档（必须带 owner_open_id）
   feishu_doc action: "create", title: "标题", owner_open_id: "ou_xxx"

步骤 2: 创建表格结构
   feishu_doc action: "create_table", doc_token: "xxx", row_size: N, column_size: M

步骤 3: 获取所有块的 block_id
   feishu_doc action: "list_blocks", doc_token: "xxx"
   → 找到每个 Text 块的 block_id（每个 TableCell 下有一个 Text 子块）

步骤 4: 逐个更新 Text 块内容
   feishu_doc action: "update_block", doc_token: "xxx", block_id: "doxcnTextBlockId", content: "单元格内容"
   → 注意：更新的是 Text 块（block_type: 2），不是 TableCell 块（block_type: 32）

步骤 5: 验证文档内容
   feishu_doc action: "read", doc_token: "xxx"
```

### ❌ 错误做法（不要使用）

```json
// 错误：write/append 会返回 400 错误
{ "action": "write", "content": "markdown内容..." }

// 错误：update_block 更新 Page 根块，内容变成标题
{ "action": "update_block", "block_id": "文档根ID（与doc_token相同）", "content": "内容" }

// 错误：create_table_with_values 会失败
{ "action": "create_table_with_values", "values": [[...]] }
```

### 块类型参考

| block_type | 名称      | 说明                           |
| ---------- | --------- | ------------------------------ |
| 1          | Page      | 文档根块，block_id = doc_token |
| 2          | Text      | 文本块，可以更新内容           |
| 31         | Table     | 表格块                         |
| 32         | TableCell | 表格单元格，包含 Text 子块     |

### 文档权限

创建文档时**必须**传入 `owner_open_id`（从 inbound metadata 的 `sender_id` 获取），否则用户看不到文档内容。

## Configuration

```yaml
channels:
  feishu:
    tools:
      doc: true # default: true
```

**Note:** `feishu_wiki` depends on this tool - wiki page content is read/written via `feishu_doc`.

## Permissions

Required: `docx:document`, `docx:document:readonly`, `docx:document.block:convert`, `drive:drive`
