---
name: i18n-translator
version: 2.0.0
description: "i18n 国际化词条翻译：一键扫描→翻译→同步→校验。扫描 React/TypeScript 项目中的中文词条，批量翻译为英文，支持飞书术语表对照、领域专有名词管理、模式规则自动翻译。当用户需要扫描项目中文词条、翻译 i18n JSON 文件、同步中英文 locale 文件、查找未翻译词条、批量翻译、管理术语表时使用。"
---

# i18n-translator

> 面向中文→英文的前端项目 i18n 翻译技能。适用于使用 i18next 的 React/TypeScript 项目。
> 核心理念：**一键自动化** — 扫描、翻译、同步、校验在一条流水线中完成。

## 核心流水线

```
┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ 1. 检测  │──▶│ 2. 扫描  │──▶│ 3. 翻译  │──▶│ 4. 同步  │──▶│ 5. 校验  │
│ 环境探测 │   │ 提取词条  │   │ 自动+手动│   │ en⇄zh   │   │ 质量报告 │
└─────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

**Agent 执行入口**：用户说"扫描/翻译/i18n"时，自动运行完整流水线。每个阶段可独立执行。

---

## 一、环境检测（自动）

执行任何 i18n 操作前，**必须先自动探测**项目配置，结果缓存到变量中供后续使用。

```bash
# 自动探测脚本 — 在项目根目录执行
PROJECT_ROOT="<project_root>"
cd "$PROJECT_ROOT"

# 1. 定位 locale 文件
EN_JSON=$(find src -name "en.json" -path "*/i18n/*" -o -name "en.json" -path "*/locale*/*" | head -1)
ZH_JSON=$(echo "$EN_JSON" | sed 's/en\.json/zh.json/')
LOCALE_DIR=$(dirname "$EN_JSON")

# 2. 定位 scanner 配置
SCANNER_CONFIG=$(find src -name "config.js" -path "*/i18n/*" | head -1)

# 3. 检测 package.json 中的 scanner 脚本名
SCANNER_SCRIPT=$(cat package.json | python3 -c "
import sys,json
scripts=json.load(sys.stdin).get('scripts',{})
for k,v in scripts.items():
    if 'i18next-scanner' in v or 'scanner' in k:
        print(k); break
" 2>/dev/null)

# 4. 检测 scanner 运行目录（从 config.js 的 input 路径推断）
# 如果 input 以 'src/' 开头 → 从项目根目录运行
# 如果 input 不含 'src/' → 从 src/ 目录运行
SCANNER_CWD=$(python3 -c "
import re
with open('$SCANNER_CONFIG') as f:
    content = f.read()
has_src_prefix = bool(re.search(r\"input.*?\\[.*?'src/\", content, re.DOTALL))
print('$PROJECT_ROOT' if has_src_prefix else '$PROJECT_ROOT/src')
" 2>/dev/null)

echo "EN_JSON=$EN_JSON"
echo "ZH_JSON=$ZH_JSON"
echo "LOCALE_DIR=$LOCALE_DIR"
echo "SCANNER_CONFIG=$SCANNER_CONFIG"
echo "SCANNER_SCRIPT=$SCANNER_SCRIPT"
echo "SCANNER_CWD=$SCANNER_CWD"
```

**探测结果存入变量后，后续步骤引用这些变量，不再硬编码路径。**

### 必须确认的信息

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `LOCALE_DIR` | locale 文件目录 | `src/i18n/locale` |
| `SCANNER_CONFIG` | scanner 配置路径 | `src/i18n/config.js` |
| `SCANNER_CWD` | scanner 运行目录 | 项目根目录 或 `src/` |
| 插值语法 | 运行时 vs scanner | 运行时 `{var}`，scanner `{{var}}` |
| `keySeparator` | 是否用 `.` 分割嵌套 key | `false`（平铺结构） |
| `contextSeparator` | 上下文分隔符 | `_::_` |

---

## 二、扫描词条

### 2.1 执行扫描

**优先用 package.json 脚本（已封装好路径和参数）：**

```bash
cd "$PROJECT_ROOT"

# 方式 1：通过 pnpm/npm 脚本（推荐）
pnpm i18n:scanner   # 或 npm run i18n:scanner

# 方式 2：直接调用（当脚本不存在时）
cd "$SCANNER_CWD" && npx i18next-scanner --config "$SCANNER_CONFIG"
```

**Scanner 行为：**
- 自动提取所有 `t('...')`、`<Trans i18nKey="...">` 中的中文 key
- 新 key 以 `defaultValue: key`（即中文原文）追加到 en.json 和 zh.json
- **已有翻译不会被覆盖**
- 若配置了 `removeUnusedKeys: true`，代码中不再使用的 key 会被移除

### 2.2 扫描后立即统计

扫描完成后自动运行统计（不需要用户介入）：

```python
import json

with open(f'{LOCALE_DIR}/en.json') as f:
    en = json.load(f)
with open(f'{LOCALE_DIR}/zh.json') as f:
    zh = json.load(f)

total = len(en)
untranslated = [k for k, v in en.items()
                if v == k and any('\u4e00' <= c <= '\u9fff' for c in k)]
en_only = set(en.keys()) - set(zh.keys())
zh_only = set(zh.keys()) - set(en.keys())

print(f'''
╔═══════════════════════════════════════╗
║         i18n 扫描结果报告             ║
╠═══════════════════════════════════════╣
║  总词条数:       {total:>6}              ║
║  已翻译:         {total - len(untranslated):>6} ({(total-len(untranslated))*100//total:>3}%)       ║
║  未翻译:         {len(untranslated):>6} ({len(untranslated)*100//total:>3}%)       ║
║  EN 独有 key:    {len(en_only):>6}              ║
║  ZH 独有 key:    {len(zh_only):>6}              ║
╚═══════════════════════════════════════╝
''')

# 将未翻译词条保存供翻译阶段使用
with open('/tmp/i18n_untranslated.json', 'w') as f:
    json.dump(untranslated, f, ensure_ascii=False, indent=2)
```

**决策点：**
- 未翻译 = 0 → 跳过翻译，直接进入同步校验
- 未翻译 > 0 → 进入翻译阶段

---

## 三、术语表管理

### 3.1 术语表加载（自动）

翻译开始前自动加载术语表。按以下顺序查找：

```python
import json, os

glossary = {}
glossary_sources = [
    # 1. 项目内术语表（持久化）
    f'{LOCALE_DIR}/glossary.json',
    f'{PROJECT_ROOT}/src/i18n/glossary.json',
    # 2. 临时术语表（上次从飞书拉取的）
    '/tmp/glossary.json',
    '/tmp/afsui_glossary.json',
]
for path in glossary_sources:
    if os.path.exists(path):
        with open(path) as f:
            loaded = json.load(f)
            glossary.update(loaded)
            print(f"已加载术语表: {path} ({len(loaded)} 条)")

print(f"术语表总计: {len(glossary)} 条")
```

### 3.2 从飞书多维表格拉取术语表

当用户提供飞书链接，或需要刷新术语表时执行。详见 [glossary-guide.md](references/glossary-guide.md)。

```bash
# 简要流程：
# 1. Wiki 链接 → 解析真实 base-token
lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}'

# 2. 获取记录
lark-cli base +record-list --base-token <obj_token> --table-id <table_id> --as user --limit 200

# 3. Python 提取 zh→en 映射 → 保存到 /tmp/glossary.json
```

### 3.3 术语表格式

```json
{
  "中文术语": "English Term",
  "文件系统": "File System",
  "网关集群": "Gateway Cluster"
}
```

---

## 四、自动翻译引擎

### 4.1 翻译优先级（自动执行，无需用户介入）

```
1. 术语表精确匹配      → 覆盖 ~5-10%
2. 模式规则批量翻译     → 覆盖 ~30-50%
3. 内置通用词翻译表     → 覆盖 ~10-20%
4. Agent 逐条智能翻译   → 覆盖剩余全部
```

### 4.2 完整自动翻译脚本

**Agent 必须生成并运行此脚本**，一次完成所有自动翻译。脚本中 `manual_translations` 由 Agent 根据剩余未翻译词条逐条填写。

```python
#!/usr/bin/env python3
"""i18n 自动翻译脚本 — 由 Agent 生成并执行"""
import json, re, os

# ============================================================
# 配置（由 Agent 根据环境检测结果填充）
# ============================================================
LOCALE_DIR = '<LOCALE_DIR>'  # Agent 替换为实际路径
GLOSSARY_PATHS = [
    f'{LOCALE_DIR}/glossary.json',
    '/tmp/glossary.json',
    '/tmp/afsui_glossary.json',
]

# ============================================================
# 1. 加载资源
# ============================================================
with open(f'{LOCALE_DIR}/en.json') as f:
    en = json.load(f)
with open(f'{LOCALE_DIR}/zh.json') as f:
    zh = json.load(f)

glossary = {}
for gp in GLOSSARY_PATHS:
    if os.path.exists(gp):
        with open(gp) as f:
            glossary.update(json.load(f))

# ============================================================
# 2. 识别未翻译词条
# ============================================================
untranslated_keys = [k for k, v in en.items()
                     if v == k and any('\u4e00' <= c <= '\u9fff' for c in k)]
total_before = len(untranslated_keys)
print(f"待翻译: {total_before} 条")

# ============================================================
# 3. 翻译函数库
# ============================================================

# --- 3.1 内置通用词映射（UI 高频词）---
BUILTIN_MAP = {
    # 操作动词
    '创建': 'Create', '修改': 'Modify', '删除': 'Delete', '编辑': 'Edit',
    '添加': 'Add', '移除': 'Remove', '查看': 'View', '搜索': 'Search',
    '导入': 'Import', '导出': 'Export', '上传': 'Upload', '下载': 'Download',
    '刷新': 'Refresh', '重试': 'Retry', '重置': 'Reset', '保存': 'Save',
    '取消': 'Cancel', '确认': 'Confirm', '提交': 'Submit', '复制': 'Copy',
    '启用': 'Enable', '禁用': 'Disable', '展开': 'Expand', '跳过': 'Skip',
    '暂停': 'Pause', '恢复': 'Recover', '返回': 'Back', '选择': 'Select',
    '关联': 'Associate', '执行': 'Execute', '验证': 'Validate', '部署': 'Deploy',
    '重启': 'Restart', '初始化': 'Initialize', '同步': 'Sync', '备份': 'Backup',
    '卸载': 'Uninstall', '更新': 'Update', '释放': 'Release', '生成': 'Generate',
    '安装': 'Install', '配置': 'Configuration', '设置': 'Settings',
    # 状态词
    '健康': 'Healthy', '异常': 'Abnormal', '活跃': 'Active',
    '在线': 'Online', '离线': 'Offline', '已启用': 'Enabled',
    '已停用': 'Disabled', '已挂载': 'Mounted', '未挂载': 'Unmounted',
    '已满': 'Full', '准备中': 'Preparing', '处理中': 'Processing',
    '执行中': 'Executing', '等待中': 'Waiting', '已完成': 'Completed',
    '已失败': 'Failed', '已取消': 'Cancelled', '已暂停': 'Paused',
    '已锁定': 'Locked', '未锁定': 'Unlocked', '已隔离': 'Isolated',
    '已确认': 'Confirmed', '未确认': 'Unconfirmed', '已连通': 'Connected',
    '已断开': 'Disconnected', '已移除': 'Removed', '已弃用': 'Deprecated',
    '已撤销': 'Revoked', '未分配': 'Unallocated', '未知': 'Unknown',
    '未设置': 'Not Set', '未运行': 'Not Running', '未配置': 'Not Configured',
    '不可用': 'Unavailable',
    # UI 通用词
    '名称': 'Name', '描述': 'Description', '状态': 'Status', '类型': 'Type',
    '操作': 'Operation', '详情': 'Details', '概况': 'Overview', '列表': 'List',
    '告警': 'Alert', '事件': 'Event', '日志': 'Log', '时间': 'Time',
    '成员': 'Member', '权限': 'Permission', '性能': 'Performance',
    '容量': 'Capacity', '利用率': 'Utilization', '吞吐量': 'Throughput',
    '带宽': 'Bandwidth', '快照': 'Snapshot', '配额': 'Quota',
    '客户端': 'Client', '服务': 'Service', '节点': 'Node',
    '集群': 'Cluster', '网关': 'Gateway', '首页': 'Home',
    '全部': 'All', '无': 'None', '是': 'Yes', '否': 'No',
    '确定': 'OK', '成功': 'Success', '失败': 'Failed',
    '完成': 'Completed', '停止': 'Stop', '开始': 'Start',
}

def translate_noun(zh):
    """翻译名词：术语表 → 内置映射 → 原文"""
    if zh in glossary:
        return glossary[zh]
    if zh in BUILTIN_MAP:
        return BUILTIN_MAP[zh]
    # 去空格模糊匹配
    stripped = zh.replace(' ', '')
    for gk, gv in {**glossary, **BUILTIN_MAP}.items():
        if gk.replace(' ', '') == stripped:
            return gv
    return zh

def translate_action(zh):
    """翻译动作短语（动词+宾语）"""
    verbs = {
        '创建': 'Create', '修改': 'Modify', '删除': 'Delete', '更新': 'Update',
        '同步': 'Sync', '恢复': 'Recover', '移除': 'Remove', '添加': 'Add',
        '刷新': 'Refresh', '启用': 'Enable', '禁用': 'Disable', '重启': 'Restart',
        '部署': 'Deploy', '卸载': 'Uninstall', '初始化': 'Initialize',
        '设置': 'Set', '检查': 'Check', '验证': 'Validate', '扩展': 'Expand',
        '回滚': 'Rollback', '备份': 'Backup', '编辑': 'Edit', '关联': 'Associate',
        '取消关联': 'Disassociate', '导入': 'Import', '导出': 'Export',
        '上传': 'Upload', '下载': 'Download', '接管': 'Takeover',
        '批量修改': 'Batch Modify', '批量删除': 'Batch Delete',
        '批量设置': 'Batch Set', '批量关联': 'Batch Associate',
        '批量取消关联': 'Batch Disassociate',
    }
    for zh_v, en_v in sorted(verbs.items(), key=lambda x: -len(x[0])):
        if zh.startswith(zh_v):
            rest = zh[len(zh_v):].strip()
            if rest:
                return f'{en_v} {translate_noun(rest)}'
            return en_v
    return None

def translate_phrase(zh):
    """翻译完整短语"""
    if zh in glossary: return glossary[zh]
    if zh in BUILTIN_MAP: return BUILTIN_MAP[zh]
    r = translate_action(zh)
    if r: return r.lower()
    return zh

def apply_patterns(key):
    """模式规则引擎：正则匹配翻译"""

    # --- 操作状态：XXX中 ---
    m = re.match(r'^(.{2,}?)(中)$', key)
    if m and not key.endswith('集中') and any('\u4e00' <= c <= '\u9fff' for c in m.group(1)):
        action = m.group(1)
        t = translate_action(action)
        if t:
            # Create → Creating, Set → Setting
            word = t.split()[0]
            rest = ' '.join(t.split()[1:])
            if word.endswith('e') and not word.endswith('ee'):
                ing = word[:-1] + 'ing'
            elif re.match(r'.*[^aeiou][aeiou][^aeiouwxy]$', word.lower()):
                ing = word + word[-1] + 'ing'
            else:
                ing = word + 'ing'
            return f'{ing} {rest}'.strip() if rest else ing

    # --- 操作结果：XXX失败 ---
    m = re.match(r'^(.+?)(失败)$', key)
    if m:
        t = translate_action(m.group(1)) or translate_phrase(m.group(1))
        if t and t != m.group(1):
            return f'{t} failed'

    # --- 操作结果：XXX成功 ---
    m = re.match(r'^(.+?)(成功)$', key)
    if m:
        t = translate_action(m.group(1)) or translate_phrase(m.group(1))
        if t and t != m.group(1):
            return f'{t} successfully'

    # --- 确认对话框：确定XXX吗？ ---
    m = re.match(r'^确定(.+?)吗？$', key)
    if m:
        t = translate_phrase(m.group(1))
        if t != m.group(1):
            return f'Are you sure you want to {t}?'

    # --- 确认删除：确定删除以下 {N} 个XXX吗？ ---
    m = re.match(r'^确定删除以下\s*\{(\w+)\}\s*个(.+?)吗？(.*)$', key, re.DOTALL)
    if m:
        n = translate_noun(m.group(2))
        tail = m.group(3).strip()
        base = f'Are you sure you want to delete the following {{{m.group(1)}}} {n}(s)?'
        return f'{base}\n{tail}'.strip() if tail else base

    # --- 表单：请输入XXX ---
    m = re.match(r'^请输入(.+)$', key)
    if m:
        f = translate_noun(m.group(1))
        if f != m.group(1):
            return f'Please enter {f}'

    # --- 表单：请选择XXX ---
    m = re.match(r'^请选择(.+)$', key)
    if m:
        f = translate_noun(m.group(1))
        if f != m.group(1):
            return f'Please select {f}'

    # --- 不支持XXX ---
    m = re.match(r'^不支持(.+)$', key)
    if m:
        t = translate_phrase(m.group(1))
        if t != m.group(1):
            return f'{t} not supported'

    # --- 数量：{var} 个XXX ---
    m = re.match(r'^\{(\w+)\}\s*个(.+)$', key)
    if m:
        n = translate_noun(m.group(2))
        return f'{{{m.group(1)}}} {n}(s)'

    # --- 共 {count} 条XXX ---
    m = re.match(r'^共\s*\{(\w+)\}\s*条(.*)$', key)
    if m:
        rest = m.group(2).strip()
        n = translate_noun(rest) if rest else 'entries'
        return f'Total {{{m.group(1)}}} {n}'

    # --- 动词+资源 组合 ---
    t = translate_action(key)
    if t:
        return t

    # --- 纯术语表/内置映射 ---
    if key in glossary:
        return glossary[key]
    if key in BUILTIN_MAP:
        return BUILTIN_MAP[key]

    return None

# ============================================================
# 4. Agent 手动翻译表（逐条补充复杂词条）
# ============================================================
manual_translations = {
    # Agent 在此处填入所有无法自动翻译的词条
    # '中文key': 'English value',
}

# ============================================================
# 5. 执行翻译
# ============================================================
auto_count = 0
manual_count = 0
remaining = []

for key in untranslated_keys:
    # 5a. 手动翻译表优先
    if key in manual_translations:
        en[key] = manual_translations[key]
        manual_count += 1
        continue
    # 5b. 自动翻译引擎
    result = apply_patterns(key)
    if result:
        en[key] = result
        auto_count += 1
        continue
    remaining.append(key)

# ============================================================
# 6. 同步 en.json ⇄ zh.json
# ============================================================
all_keys = set(en.keys()) | set(zh.keys())
for k in all_keys:
    if k not in en: en[k] = k
    if k not in zh: zh[k] = k

# ============================================================
# 7. 排序并写入
# ============================================================
for data, filename in [(en, 'en.json'), (zh, 'zh.json')]:
    sorted_data = dict(sorted(data.items(), key=lambda x: x[0]))
    with open(f'{LOCALE_DIR}/{filename}', 'w') as f:
        json.dump(sorted_data, f, ensure_ascii=False, indent=2)
        f.write('\n')

# ============================================================
# 8. 校验 & 报告
# ============================================================
final_untranslated = [k for k, v in en.items()
                      if v == k and any('\u4e00' <= c <= '\u9fff' for c in k)]

# 插值变量一致性检查
var_pat = re.compile(r'\{[^}]+\}')
var_mismatch = []
for k in en:
    if k in zh and en[k] != k:
        en_vars = set(var_pat.findall(en[k]))
        key_vars = set(var_pat.findall(k))
        if key_vars and not key_vars.issubset(en_vars):
            var_mismatch.append(k)

print(f'''
╔═══════════════════════════════════════════════╗
║           i18n 翻译完成报告                   ║
╠═══════════════════════════════════════════════╣
║  总词条数:           {len(en):>6}                    ║
║  本次待翻译:         {total_before:>6}                    ║
║  ├─ 自动翻译:        {auto_count:>6}                    ║
║  ├─ 手动翻译:        {manual_count:>6}                    ║
║  └─ 仍未翻译:        {len(remaining):>6}                    ║
║  已翻译总计:         {len(en)-len(final_untranslated):>6} ({(len(en)-len(final_untranslated))*100//len(en):>3}%)         ║
║  en⇄zh 已同步:       ✓                       ║
║  变量一致性问题:     {len(var_mismatch):>6}                    ║
╚═══════════════════════════════════════════════╝
''')

if remaining:
    print(f"未翻译词条已保存到 /tmp/i18n_remaining.json")
    with open('/tmp/i18n_remaining.json', 'w') as f:
        json.dump(remaining, f, ensure_ascii=False, indent=2)

if var_mismatch:
    print(f"变量不一致词条: {var_mismatch[:10]}")
```

### 4.3 Agent 执行流程

当未翻译词条数 > 0 时，Agent 必须按以下步骤执行：

1. **先运行一次空 `manual_translations` 的脚本** — 观察自动翻译能覆盖多少
2. **读取 `/tmp/i18n_remaining.json`** — 这些是自动翻译无法处理的
3. **按类别批量填写 `manual_translations`**（见 §4.4 分类策略）
4. **重新运行脚本**（含 `manual_translations`）— 完成全部翻译
5. **如果仍有剩余**，继续补充 `manual_translations` 并重跑

### 4.4 手动翻译分类策略

剩余词条按以下类别分批翻译，**每批不超过 300 条**：

| 优先级 | 类别 | 特征 | 示例 |
|--------|------|------|------|
| P1 | 短词条（≤4字） | 单词/短语 | `客户端`、`全选`、`水位` |
| P2 | 操作+资源 | 动词开头 | `创建域名`、`删除大数据服务` |
| P3 | 状态消息 | 以"中/失败/成功"结尾 | `元数据服务设置中` |
| P4 | 对话框/提示 | "确定/请/提示" 开头 | `确定删除网关分区吗？` |
| P5 | 表单验证 | "仅支持/不能/请输入" | `仅支持 1～80 个字符` |
| P6 | 长说明文本 | >30字 | 帮助提示、注意事项 |
| P7 | 含变量模板 | 含 `{var}`/`{{var}}` | `{name}创建共享数已达上限` |

---

## 五、翻译质量规则

**自动校验项（脚本已内置）：**
1. ✅ en/zh key 集合一致
2. ✅ 插值变量 `{var}` 在翻译中保留
3. ✅ 无中文残留在 en.json 值中

**Agent 必须遵循的翻译规则：**

| 规则 | 说明 | 示例 |
|------|------|------|
| 保留插值变量 | `{name}`, `{count}`, `{0}`, `{{0}}` 原样保留 | `共 {count} 条` → `Total {count} entries` |
| 保留 HTML 标签 | `<0>...</0>`, `<br/>` 保留 | `<0>{name}</0>的共享` → `Share of <0>{name}</0>` |
| 专有名词不翻译 | NFS, SMB, SFTP, POSIX, WORM, QoS, IOPS, iSCSI, NVMe, HDFS, S3 | 保持原样 |
| 单位不翻译 | MB/s, GB, KB, GiB, ops/s | 保持原样 |
| 术语一致性 | 同一术语全项目统一 | 以术语表为准 |
| 成功消息格式 | `"XXXed successfully"` | `创建 NFS 共享成功` → `Created NFS Share successfully` |
| 失败消息格式 | `"XXX failed"` 或 `"Failed to XXX"` | `删除失败` → `Delete failed` |
| 进行中格式 | `"XXXing"` | `创建中` → `Creating` |
| 英文标点 | 翻译值用英文标点 | `。` → `.`，`，` → `,` |

---

## 六、常见翻译对照表

### 操作动词

| 中文 | 英文 | 中文 | 英文 |
|------|------|------|------|
| 创建 | Create | 修改 | Modify |
| 删除 | Delete | 编辑 | Edit |
| 添加 | Add | 移除 | Remove |
| 查看 | View | 搜索 | Search |
| 导入 | Import | 导出 | Export |
| 上传 | Upload | 下载 | Download |
| 刷新 | Refresh | 重试 | Retry |
| 重置 | Reset | 保存 | Save |
| 取消 | Cancel | 确认 | Confirm |
| 启用 | Enable | 禁用 | Disable |
| 复制 | Copy | 展开 | Expand |
| 收起 | Collapse | 返回 | Back |
| 关联 | Associate | 取消关联 | Disassociate |

### 状态词

| 中文 | 英文 | 中文 | 英文 |
|------|------|------|------|
| 健康 | Healthy | 异常 | Abnormal |
| 活跃 | Active | 在线 | Online |
| 离线 | Offline | 已启用 | Enabled |
| 已停用 | Disabled | 已挂载 | Mounted |
| 已满 | Full | 处理中 | Processing |
| 执行中 | Executing | 等待中 | Waiting |
| 已锁定 | Locked | 已隔离 | Isolated |

### 存储/基础设施领域

| 中文 | 英文 | 中文 | 英文 |
|------|------|------|------|
| 文件系统 | File System | 存储池 | Storage Pool |
| 物理盘 | Physical Disk | 网关集群 | Gateway Cluster |
| 网关分区 | Gateway Zone | 虚拟 IP | Virtual IP |
| 配额 | Quota | 快照 | Snapshot |
| 吞吐量 | Throughput | 带宽 | Bandwidth |
| 延迟 | Latency | 利用率 | Utilization |
| 容量 | Capacity | 负载均衡 | Load Balancing |
| 元数据 | Metadata | 数据保护 | Data Protection |
| 大数据服务 | Big Data Service | 对象桶 | Object Bucket |

---

## 七、上下文分隔符

同一中文 key 不同场景有不同英文翻译时，使用 context：

```typescript
t('块存储', { context: 'navigation' })

// en.json
{
  "块存储": "Block Storage",
  "块存储_::_navigation": "Block Storage"
}
```

分隔符由项目配置决定（`contextSeparator`），常见为 `_::_`。

---

## 八、Agent 行为规范

### 流水线触发规则

| 用户指令 | Agent 动作 |
|----------|-----------|
| "扫描词条" / "i18n scan" | 执行 §1→§2 |
| "翻译" / "translate" | 执行 §1→§2→§3→§4 |
| "同步" / "sync" | 执行 §1→§4(仅同步) |
| "检查" / "validate" | 执行 §1→§4(仅校验) |
| "全部" / "一键完成" / 无特定指令 | 执行完整流水线 §1→§2→§3→§4 |

### 必须做
- 翻译前自动加载术语表
- 翻译前先扫描提取最新词条
- 自动翻译后统计覆盖率并报告
- 剩余词条分类后批量逐条翻译
- 翻译后自动同步 en.json ⇄ zh.json
- 每次保存后运行校验
- 保留所有插值变量和 HTML 标签

### 禁止做
- 不要凭猜测翻译专有名词（先查术语表）
- 不要修改 zh.json 中已有的值（除 key 同步外）
- 不要删除已有的正确翻译
- 不要把纯英文/数字 key 也当作"未翻译"处理
- 不要一次脚本中填入超过 500 条 manual_translations（分批运行）
- 不要改变 JSON 排序方式（保持 key 字母排序）

### 输出规范
- JSON：2 空格缩进，`ensure_ascii=False`，末尾换行 `\n`
- key 排序：按 Unicode 字母序（中文在后）
- zh.json 中 key 的值始终等于 key 本身

---

## 九、快速命令索引

| 任务 | 命令 |
|------|------|
| 完整流水线 | `pnpm i18n:scanner` + 运行翻译脚本 |
| 仅扫描 | `pnpm i18n:scanner` |
| 仅统计 | 运行 §2.2 统计脚本 |
| 拉取术语表 | `lark-cli base +record-list ...` → 提取 |
| 仅同步 | 运行 §4.2 脚本（清空 manual_translations） |
| 仅校验 | 运行 §4.2 脚本的 §8 校验部分 |

---

## 十、项目配置参考

### Scanner 配置要点

```javascript
// config.js 关键配置
{
    input: ['src/components/**', 'src/pages-ui/**', ...],
    removeUnusedKeys: true,  // 移除代码中不再使用的 key
    lngs: ['en', 'zh'],
    defaultLng: 'zh',
    defaultValue: (lng, ns, key) => key,  // 新 key 默认值=中文原文
    keySeparator: false,
    nsSeparator: false,
    contextSeparator: '_::_',
    interpolation: { prefix: '{{', suffix: '}}' },  // Scanner 用 {{}}
}
```

### 运行时 i18n 配置要点

```typescript
// i18nService.ts 关键配置
{
    interpolation: { prefix: '{', suffix: '}' },  // 运行时用 {}
    contextSeparator: '_::_',
    keySeparator: false,
    nsSeparator: false,
}
```

> ⚠️ **scanner 用 `{{var}}`，运行时用 `{var}`** — 这是设计如此，不要"修复"。
