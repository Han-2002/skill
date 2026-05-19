---
name: docx-comment-reply
description: Read all comments from a Word (.docx) file and write reply comments back into the document as threaded replies using OOXML manipulation (comments.xml + commentsExtended.xml). Use when the user wants to batch-reply to Word document comments.
description_zh: Word文档批注回复写入
description_en: Reply to Word document comments
disable: false
agent_created: true
---

# docx-comment-reply

## When to use
When the user wants to read all comments in a `.docx` file and write reply content back into the document as threaded replies (visible as "回复" under each comment in Word).

## Steps

### 1. Read existing comments from comments.xml
```python
import zipfile, xml.etree.ElementTree as ET

ns = {'w': 'http://schemas.openxmlformats.org/wordprocessingml/2006/main'}
with zipfile.ZipFile(docx_path, 'r') as z:
    with z.open('word/comments.xml') as f:
        tree = ET.parse(f)
root = tree.getroot()
comments = root.findall('w:comment', ns)
```
Each `w:comment` has `w:id`, `w:author`, `w:date`. The **last paragraph's** `w14:paraId` in each comment is the parent anchor used by commentsExtended.

### 2. Read commentsExtended.xml to get existing paraId anchors
```python
with zipfile.ZipFile(docx_path, 'r') as z:
    with z.open('word/commentsExtended.xml') as f:
        ext_content = f.read().decode('utf-8')
```
The `w15:paraIdParent` attribute links reply comments back to their parent.

### 3. Build reply comment XML
```python
import uuid

def new_para_id():
    return uuid.uuid4().hex[:8].upper()

def make_reply_comment(cid, author, reply_text, para_id):
    text = reply_text.replace('&','&amp;').replace('<','&lt;').replace('>','&gt;').replace('"','&quot;')
    return (
        f'<w:comment w:id="{cid}" w:author="{author}" w:date="2026-05-14T15:37:00Z" w:initials="">'
        f'<w:p w14:paraId="{para_id}">'
        f'<w:pPr><w:pStyle w:val="5"/><w:rPr>'
        f'<w:rFonts w:hint="eastAsia"/>'
        f'<w:lang w:val="en-US" w:eastAsia="zh-CN"/>'
        f'</w:rPr></w:pPr>'
        f'<w:r><w:rPr><w:rFonts w:hint="eastAsia"/>'
        f'<w:lang w:val="en-US" w:eastAsia="zh-CN"/></w:rPr>'
        f'<w:t xml:space="preserve">{text}</w:t></w:r>'
        f'</w:p>'
        f'</w:comment>'
    )

def make_comment_ex(para_id, parent_para_id):
    return (
        f'<w15:commentEx w15:paraId="{para_id}" '
        f'w15:paraIdParent="{parent_para_id}" w15:done="0"/>'
    )
```

### 4. Inject replies into both XML files and repack the zip
```python
import shutil, stat, os

# Copy source to destination (ensure dest is writable)
shutil.copy2(src_path, dst_path)
os.chmod(dst_path, stat.S_IREAD | stat.S_IWRITE)

with zipfile.ZipFile(dst_path, 'r') as zin:
    names = zin.namelist()
    files = {n: zin.read(n) for n in names}

# Append reply comments to comments.xml
comments_xml = files['word/comments.xml'].decode('utf-8')
new_comments_xml = INSERT_BEFORE_CLOSE_TAG(comments_xml, '</w:comments>', reply_comment_xml_fragments)
files['word/comments.xml'] = new_comments_xml.encode('utf-8')

# Append commentsEx entries to commentsExtended.xml
ext_xml = files['word/commentsExtended.xml'].decode('utf-8')
new_ext_xml = INSERT_BEFORE_CLOSE_TAG(ext_xml, '</w15:commentsEx>', reply_ex_fragments)
files['word/commentsExtended.xml'] = new_ext_xml.encode('utf-8')

with zipfile.ZipFile(dst_path, 'w', zipfile.ZIP_DEFLATED) as zout:
    for name in names:
        zout.writestr(name, files[name])
```

## Key data to extract from original comments

For each original comment, record:
- `w:id` — comment ID (integer)
- `w:author` — author name
- The **last** `<w:p>` element's `w14:paraId` — this is the `paraIdParent` for the reply

**CRITICAL**: The `paraIdParent` must point to an **empty paragraph** (no text content) within the original comment. This empty paragraph is the thread anchor. If the original comment only has **one paragraph** (with text), you must **inject a new empty paragraph** into that comment in `comments.xml` first, and use that new empty paragraph's `paraId` as the `paraIdParent`. Also add a `w15:commentEx` entry for the new empty paragraph (without `paraIdParent`) in `commentsExtended.xml`.

Check paragraph count per comment:
```python
for c in root.findall(f'{{{ns_w}}}comment'):
    pids = [p.get(f'{{{ns_w14}}}paraId') for p in c.findall(f'{{{ns_w}}}p')]
    # If len(pids) == 1, this comment needs an empty paragraph injected
```

The reply comments should have:
- New sequential `w:id` starting from `max_existing_id + 1`
- New unique `w14:paraId` (8-char uppercase hex via uuid4)
- `w15:commentEx` entry with `w15:paraIdParent` pointing to the original comment's empty-paragraph paraId

## Pitfalls

- **Source files may be read-only** (e.g. files synced from WeChat). Always write output to a writable directory (workspace). Use `os.chmod(dst, stat.S_IREAD | stat.S_IWRITE)` after copy.
- **Do not delete the dst file before chmod**: if it exists and is read-only from a prior partial run, chmod+remove it first.
- **Single-paragraph comments need an empty anchor paragraph**: Word/WPS threaded reply requires `paraIdParent` to point to an empty (no text) paragraph inside the original comment. If an original comment has only 1 paragraph (with text), inject `<w:p w14:paraId="NEWID"><w:pPr><w:pStyle w:val="5"/></w:pPr></w:p>` before `</w:comment>`, and add a `<w15:commentEx w15:paraId="NEWID" w15:done="0"/>` entry in commentsExtended.xml.
- **String escaping**: Reply text must have `&`, `<`, `>`, `"` escaped for XML. Curly-quote characters do NOT need escaping.
- **paraId must be unique** across the whole document — use uuid4 hex.
- **commentsExtended.xml is required** for threaded replies to display correctly in modern Word/WPS. Without it, replies appear as separate top-level comments.
- **Do not use heredoc with non-ASCII Python code in bash** — encoding issues cause SyntaxError. Write the script to a `.py` file with the Write tool and run it separately.

## Verification

After writing, verify:
```python
with zipfile.ZipFile(dst_path, 'r') as z:
    with z.open('word/comments.xml') as f:
        tree = ET.parse(f)
comments = tree.getroot().findall('w:comment', ns)
print(f'Total comments: {len(comments)}')
# Should be original_count + reply_count
```
