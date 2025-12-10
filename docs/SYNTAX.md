# Syntactic Constructs for .chat.md Files

This document provides a complete reference for all syntactic constructs used in `.chat.md` files.

## Table of Contents

1. [Role Markers (Block Delimiters)](#1-role-markers-block-delimiters)
2. [File-Specific Configuration Preamble](#2-file-specific-configuration-preamble)
3. [File Attachment Syntax](#3-file-attachment-syntax)
4. [Tool Call Syntax](#4-tool-call-syntax)
5. [Tool Execution Blocks](#5-tool-execution-blocks)
6. [Settings Block Syntax](#6-settings-block-syntax)
7. [Streaming Triggers](#7-streaming-triggers)
8. [Image Handling](#8-image-handling)
9. [Comments](#9-comments)
10. [Special Characters & Escape Sequences](#10-special-characters--escape-sequences)
11. [Block-Specific Rules](#11-block-specific-rules)
12. [Error Conditions](#12-error-conditions)

---

## 1. Role Markers (Block Delimiters)

Role markers define the structure of a conversation by separating different types of content blocks.

### Syntax

```
# %% <role>
```

### Supported Roles

| Role | Description |
|------|-------------|
| `system` | System prompt (aggregated across file) |
| `user` | User message block |
| `assistant` | Assistant response block |
| `tool_execute` | Tool execution results block |
| `settings` | TOML-like settings configuration |

### Pattern

```regex
^# %% (user|assistant|system|tool_execute|settings)\s*$
```

Role matching is **case-insensitive**.

### Example

```markdown
# %% system
You are a helpful assistant.

# %% user
Hello, how are you?

# %% assistant
I'm doing well, thank you for asking!
```

---

## 2. File-Specific Configuration Preamble

Optional key-value configuration that appears **before** the first block marker.

### Syntax

```
key=value
key="quoted value"
key=123
```

### Allowed Keys

| Key | Type | Description |
|-----|------|-------------|
| `selectedConfig` | string | References a named API configuration |
| `reasoningEffort` | string | Reasoning depth: `"minimal"`, `"low"`, `"medium"`, `"high"` |
| `maxTokens` | integer | Maximum response tokens |
| `maxThinkingTokens` | integer | Maximum thinking tokens |

### Example

```markdown
selectedConfig="claude-opus"
reasoningEffort="high"
maxTokens=4000

# %% system
Respond thoughtfully.
```

### Forbidden Keys

These must be set in global settings, not per-file:
- `type`
- `apiKey`
- `base_url`
- `model_name`
- `apiConfigs`

---

## 3. File Attachment Syntax

Files can be attached to user messages using several methods.

### Method 1: Markdown-Style Links with #file Tag

```markdown
[#file](path/to/file.js)
[#file](/absolute/path/file.txt)
[#file](~/relative/to/home.py)
```

### Method 2: "Attached file at" Syntax

```markdown
Attached file at /path/to/file.js

```javascript
// optional code fence showing content
```

Attached file at ~/Documents/image.png
```

### Method 3: Tool Result Links (in tool_execute blocks)

```markdown
<tool_result>
[Tool Result](cmdassets/tool-result-TIMESTAMP-HASH.txt)
</tool_result>
```

### Method 4: MCP Prompt Links

```markdown
[MCP Prompt: prompt-name](path/to/prompt.txt)
```

### Supported Path Formats

| Format | Example |
|--------|---------|
| Relative | `./file.txt`, `src/utils.js` |
| Absolute | `/absolute/path/file.txt` |
| Home directory | `~/Documents/file.txt` |

---

## 4. Tool Call Syntax

Tool calls use XML structure within assistant blocks.

### Basic Structure

```xml
<tool_call>
<tool_name>ToolName</tool_name>
<param name="paramName">paramValue</param>
<param name="paramName2">paramValue2</param>
</tool_call>
```

### Code Fencing Variants

Tool calls can be optionally wrapped in code fences:

````markdown
```xml
<tool_call>
<tool_name>ReadFile</tool_name>
<param name="path">/path/to/file.txt</param>
</tool_call>
```
````

Recognized fence types: `xml`, `tool_call`, `tool_code`

### Parameter Value Types

```xml
<!-- Simple string -->
<param name="key">value</param>

<!-- JSON object -->
<param name="data">{"key": "value"}</param>

<!-- JSON array -->
<param name="list">[1, 2, 3]</param>

<!-- Multi-line with quotes -->
<param name="content">"line1\nline2"</param>
```

### CDATA Sections

Use CDATA for parameter values containing XML-like content:

```xml
<param name="content"><![CDATA[
  Content with <xml> tags inside
  Multiple lines allowed
]]></param>
```

---

## 5. Tool Execution Blocks

### Empty Block (Triggers Execution)

```markdown
# %% tool_execute
```

An empty `tool_execute` block triggers execution of pending tool calls.

### Block with Results

```markdown
# %% tool_execute
<tool_result>
Result content here
</tool_result>
```

### Tool Result with File Reference

```markdown
<tool_result>
[Tool Result](cmdassets/tool-result-1234567890-abc123.txt)
</tool_result>
```

---

## 6. Settings Block Syntax

The settings block uses TOML-like syntax for configuration.

### Format

```markdown
# %% settings
[section_name]
key = "value"
key2 = 123
key3 = true

array_key = [
  "item1",
  "item2",
]
```

### Supported Value Types

| Type | Example |
|------|---------|
| String | `key = "value"` or `key = 'value'` |
| Number | `key = 123` or `key = 3.14` |
| Boolean | `key = true` or `key = false` |
| Array | `key = [value1, value2]` |

---

## 7. Streaming Triggers

### Empty Assistant Block

An empty assistant block at the end of the file triggers LLM streaming:

```markdown
# %% user
What is the capital of France?

# %% assistant

```

The assistant block must contain only whitespace (or nothing) after the marker.

### Idempotent Streaming Behavior

- Tokens are appended to the last non-empty assistant block
- If the target text is deleted, the streamer aborts
- A file lock prevents concurrent streaming/listening operations

---

## 8. Image Handling

### Supported Formats

`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.bmp`

### Markdown Image Syntax

```markdown
![alt text](path/to/image.png)
![Tool generated image](cmdassets/tool-image.png)
```

### Attachment in User Blocks

```markdown
# %% user
Here's the image I want you to analyze:
[#file](screenshot.png)
```

### Tool-Generated Images

MCP tools can embed images in responses. These are automatically saved to a `cmdassets/` directory adjacent to the chat file.

### Restriction

Images are **NOT allowed** in `# %% system` blocks.

---

## 9. Comments

### In Configuration Preamble

```markdown
# This is a comment
selectedConfig="provider"  # Inline comment
maxTokens=4000  # Another inline comment
```

### In Settings Block

```markdown
# %% settings
# Section comment
[section]
key = "value"  # Inline comment
```

---

## 10. Special Characters & Escape Sequences

### Whitespace Handling

- Leading/trailing whitespace in parameter values is trimmed
- Newlines within parameter values are preserved
- Inline comments after unquoted values are stripped

### Quote Handling

- Quoted strings preserve internal content exactly
- Both single (`'`) and double (`"`) quotes supported
- Outer quotes are removed during parsing

### CDATA Sections

Used when parameter values contain XML-like content:

```xml
<param name="html"><![CDATA[
<div class="container">
  <h1>Title</h1>
</div>
]]></param>
```

---

## 11. Block-Specific Rules

### System Block

- Multiple `# %% system` blocks are **aggregated** with newline separation
- Content is used as a prepended system prompt
- **Cannot contain image references** (will cause an error)

### User Block

- Can contain file attachments and images
- Text and file content are combined in the message
- Empty user blocks are skipped

### Assistant Block

- Can contain tool calls with XML structure
- Can contain plain text responses
- Empty blocks trigger streaming

### Tool Execute Block

- Contains `<tool_result>` tags with execution results
- Can reference external files via markdown links
- Empty blocks trigger tool execution

---

## 12. Error Conditions

| Condition | Description |
|-----------|-------------|
| Invalid Start Content | Non-key-value, non-comment content before first block marker |
| Forbidden Inline Config Keys | Using `type`, `apiKey`, `base_url`, `model_name`, or `apiConfigs` in preamble |
| Images in System Block | Image references in `# %% system` blocks |
| Unbalanced CDATA Tags | Missing closing `]]>` in parameter CDATA sections |

---

## File Conventions

### Chat Assets Directory

Tool-generated files are stored in `cmdassets/` adjacent to the chat file:

- **Images**: `tool-image-DATETIME.png`
- **Tool results**: `tool-result-TIMESTAMP-HASH.txt`

---

## Quick Reference

```markdown
selectedConfig="my-config"
maxTokens=4000

# %% system
You are a helpful assistant.

# %% user
[#file](code.py)
Explain this code.

# %% assistant
This code does...

<tool_call>
<tool_name>ReadFile</tool_name>
<param name="path">./example.txt</param>
</tool_call>

# %% tool_execute
<tool_result>
File contents here...
</tool_result>

# %% assistant

```
