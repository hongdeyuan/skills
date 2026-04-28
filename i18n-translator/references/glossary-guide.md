# 飞书术语表接入指南

## 概述

从飞书多维表格（Base）中获取团队统一的术语翻译对照表，确保 i18n 翻译中专有名词一致。

## 前提条件

1. 已安装 `lark-cli`（`npm i -g @larksuite/cli`）
2. 已配置应用（`lark-cli config init`）
3. 已获得 Base 读取权限（`lark-cli auth login --domain base --domain wiki`）

## 完整接入流程

### Step 1：解析链接

```bash
# Wiki 链接 → 解析真实 base-token
lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}'
# 从返回结果中取 node.obj_token 和 node.obj_type

# Base 直接链接 → 从 URL 提取
# https://xxx.feishu.cn/base/{base-token}?table={table-id}
```

### Step 2：获取表结构

```bash
lark-cli base +field-list \
  --base-token <obj_token> \
  --table-id <table_id> \
  --as user
```

记下中文字段名（如 "术语(中文)"）和英文字段名（如 "术语(英文)"）。

### Step 3：拉取记录

```bash
# 首批（最大 200 条/次）
lark-cli base +record-list \
  --base-token <obj_token> \
  --table-id <table_id> \
  --as user --limit 200

# 如需分页
lark-cli base +record-list ... --limit 200 --offset 200
```

### Step 4：提取术语映射

```python
import json

# records = 从 +record-list 获取的记录列表
glossary = {}
for rec in records:
    fields = rec.get('fields', {})
    zh = extract_text(fields.get('<中文字段名>'))
    en = extract_text(fields.get('<英文字段名>'))
    if zh and en:
        glossary[zh.strip()] = en.strip()

def extract_text(value):
    """处理飞书 Base 的多种字段格式"""
    if isinstance(value, str):
        return value
    if isinstance(value, list):
        texts = []
        for item in value:
            if isinstance(item, dict) and 'text' in item:
                texts.append(item['text'])
            elif isinstance(item, str):
                texts.append(item)
        return ''.join(texts)
    return ''

# 保存
with open('/tmp/glossary.json', 'w') as f:
    json.dump(glossary, f, ensure_ascii=False, indent=2)
print(f"提取 {len(glossary)} 条术语")
```

### Step 5：持久化（可选）

将术语表保存到项目中供后续自动加载：

```bash
cp /tmp/glossary.json <project_root>/src/i18n/glossary.json
```

## 注意事项

- 飞书 Base 的 fields 返回格式因字段类型而异（纯文本 vs 富文本 vs 数组）
- 使用 `--as user` 访问用户有权限的 Base（bot 通常无权限）
- `+record-list --limit` 最大 200，超过需分页
- 术语表自动加载路径：`src/i18n/glossary.json` > `/tmp/glossary.json`
