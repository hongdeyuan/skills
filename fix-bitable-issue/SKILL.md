---
name: fix-bitable-issue
description: 当用户提供 Feishu/Lark Bitable 的 table URL、record URL，或者明确要求“根据 bitable record 修复代码”时使用。先用 `feishu-bitable record` 获取单条记录和附件，再结合仓库代码定位问题、实施修复、验证；修复完成后先向用户汇报并等待确认，不要自动提交 MR 或回写 record。
---

# Fix Bitable Issue

这个 skill 用于“读取 bitable 记录并修代码”，不是通用的 CLI 参数手册。重点是尽快拿到单条 record，把记录里的字段、截图、附件转成可执行的修复任务。修复完成后先向用户汇报结果并等待确认；不要默认回写 bitable record，也不要默认提交 commit / 推送分支 / 创建 MR。

## 前置检查

先检查 CLI 和认证是否可用：

```bash
feishu-bitable --help
echo "FEISHU_ACCESS_TOKEN set: $([ -n "$FEISHU_ACCESS_TOKEN" ] && echo yes || echo no)"
echo "LARK_ACCESS_TOKEN set: $([ -n "$LARK_ACCESS_TOKEN" ] && echo yes || echo no)"
echo "FEISHU_APP_ID set: $([ -n "$FEISHU_APP_ID" ] && echo yes || echo no)"
echo "FEISHU_APP_SECRET set: $([ -n "$FEISHU_APP_SECRET" ] && echo yes || echo no)"
```

如果 CLI 未安装，执行：

```bash
npm install -g @acring/feishu-bitable-cli
```

如果没有 access token，也没有 app 凭证，不要猜测数据内容，先让用户补齐认证信息。

## 输入要求

执行 `record` 子命令时只需要保证最终能拿到对应的表格上下文：

- `table-url`: 可选。CLI 会优先从显式参数读取；如果未传，会自动从环境变量或 `.env.*` 文件获取
- `record-url`: 单条记录的分享链接，形如 `https://.../record/<shareToken>`

如果用户只给了 `record-url`，默认先直接调用 CLI，不要预先解析环境变量或 `.env.*` 文件。

只有在 CLI 报错提示拿不到 `table-url`、表格上下文或相关配置时，再按这个顺序排查：

1. 先看当前 shell 环境里是否已有 `FEISHU_BITABLE_TABLE_URL`
2. 再看仓库里的 `.env.local`、`.env` 等本地配置文件
3. 仍然查不到，再向用户要，不要臆造

如果后续需要回写 record，先读取 `references/update-record.md`。`update-record` 用的是 `record-id`，不是 `record-url`；优先从 `feishu-bitable record` 的输出里读取 `recordId`。

## 这类任务里最容易踩的坑

- 不要复用固定的 `./tmp/bitable-record` 目录名。优先为当前任务创建唯一目录，例如 `./tmp/bitable-record-<record-id>` 或 `./tmp/bitable-record-<date>`，避免旧文件污染本次判断。
- 不要看到 `fields.json` 就直接按数组处理。`feishu-bitable fields` 的输出顶层通常是对象，字段列表在 `items` 里，字段名是 `field_name`，不是 `fieldName`。
- 不要把 `update-record` 输出的 `updated-record.json` 当作唯一验收依据。它的结构可能和 `record.json` 不一致，字段类型展示也可能不同。回写后应重新执行一次 `feishu-bitable record` 来确认真实写入结果。
- 不要臆造状态值。单选字段必须先从 `fields.json` 里读取实际可选项，例如这次 `修复状态` 的合法值是 `待回归` / `已回归`。
- 不要默认认为文本字段在 `record.json` 里显示为富文本数组，就必须按数组格式回写。很多文本字段回写时直接传字符串即可，但仍要通过回读验证最终落库形态。
- 不要为了清理临时目录直接写死 `rm -rf ./tmp/`。只清理本次任务生成的目录；如果命令受策略限制，改用更精确的删除方式。
- 不要在 shell 里把包含 `!`、引号或多行内容的 JSON 直接内联到命令里。字段较多或内容复杂时，优先写 `update-fields.json` 再用 `--fields-file`。

## 标准流程

### 1. 拉取单条记录

默认先直接使用下面的命令：

```bash
feishu-bitable record \
  "<record-url>" \
  --output ./tmp/bitable-record-<record-id-or-date>
```

如果用户已经明确给了 `table-url`，或者 CLI 自动读取失败后你需要显式覆盖，再传双参数形式：

```bash
feishu-bitable record \
  "<table-url>" \
  "<record-url>" \
  --output ./tmp/bitable-record-<record-id-or-date>
```

`--output` 目录下通常会包含：

- `record.json`: 单条记录的结构化数据
- `files/`: 下载下来的附件

### 2. 读取记录并整理修复线索

优先从 `record.json` 中提取信息，再结合附件确认真实问题。

这一步顺手记下：

- `recordId`
- 当前任务使用的临时目录
- 可能用于分支名 / MR / 回写备注的 issue key、record 链接或主字段文案

如果 record 没有 Jira / NTOS 编号，不要强行虚构。分支名可退化为 `fix/bitable-<record-id>`，MR 标题和描述里直接引用 bitable record。

### 3. 结合附件确认真实问题

如果 `files/` 下有截图、录屏、日志或导出文件，先阅读这些附件，再开始改代码。很多问题的真实边界只在附件里，不在字段摘要里。

如果记录信息不足以支撑实现，明确指出缺失项，例如：

- 缺少复现步骤
- 缺少期望行为
- 缺少具体页面或接口范围
- 截图无法反映交互前后状态

不要在上下文不足时自行补需求。

### 4. 创建修复分支

根据 issue 类型选择分支前缀：

- Bug 修复: `fix/<issue-key>` (如 `fix/NTOS-3705`)
- 新功能: `feat/<issue-key>` (如 `feat/NTOS-3705`)

```bash
git checkout -b fix/<issue-key>
```

### 5. 实施修复

修复时遵循这些原则：

- 修改范围尽量小，但要覆盖问题根因
- 保持现有代码风格和组件模式
- 不要因为 record 提到一个现象，就跳过对真实触发路径的核对
- 如果 record 里包含验收条件，代码结果必须对齐这些条件

### 6. 验证结果

至少完成与改动直接相关的验证：

- 单测
- 受影响模块的构建
- 相关页面或流程的本地验证

如果无法运行验证，明确说明原因和未验证范围。

### 7. 汇报结果并等待用户确认

完成修复和必要验证后，先停下来向用户汇报，不要立刻回写 record，也不要立刻提交 commit / 推送分支 / 创建 MR。

汇报时至少包括：

- 这次使用了哪条 bitable record 作为上下文
- 从 record 中提取了哪些关键信息
- 具体修改了什么代码
- 做了哪些验证
- 还有哪些风险、假设或待确认项

明确告诉用户：代码已修复并完成当前验证，等待其确认“代码和行为符合预期”后，再继续执行后续收尾动作。

### 8. 完成修复后

向用户确认是否需要：

- 提交代码 (`git commit`)
- 推送到远程并创建 MR（使用 `/gitlab` skill 执行 MR 创建流程）
- 清理本次任务的临时文件（仅删除本次生成的目录，不要清空整个 `./tmp/`）

#### MR 格式规范

**标题格式**：`<type>(<issue-key>): 简要描述`

- Bug 修复: `fix(SPRING-9257): 修复共享路径校验失败的问题`
- 新功能: `feat(NTOS-3705): 新增快照克隆功能`

如果没有 issue key：

- 分支名可用 `fix/bitable-<record-id>`
- MR 标题可退化为 `fix(<scope>): 简要描述` 或 `fix: 简要描述`
- MR 描述的 `Issue` 段落直接放 bitable record 链接，不要伪造 Jira 链接

**描述模板**：

```markdown
## Summary

- 简要描述本次变更的目的和内容

## Bitable Record

- [<record-url>](record-url)

## Changes

- 具体变更 1
- 具体变更 2

## Test plan

- [ ] 测试用例 1
- [ ] 测试用例 2
- [ ] 需要进行的手动测试 1
- [ ] 需要进行的手动测试 2

🤖 Generated with [Agent]
```

### 9. 用户确认后，再决定是否回写 record 状态

只有在用户明确确认修复代码和行为符合预期，并且明确要求同步单据状态时，才更新 bitable record。

具体字段检查、命令示例和回写注意事项，见 `references/update-record.md`。先查字段定义，再确定状态字段、目标值，以及是否需要同时回写 MR 链接、备注、处理人等字段。

执行回写时，补充遵循这些规则：

1. 先用 `feishu-bitable fields` 查字段定义，再从 `items` 里读取 `field_name`、`type` 和选项值。
2. 状态字段如果有多种合理取值，不要替用户做业务判断；先结合字段选项和用户原话确认。例如“代码修复完成”并不等于“已回归”。
3. 字段值较多时，优先写 `update-fields.json` 并通过 `--fields-file` 更新，避免 shell 转义、历史展开和多行文本问题。
4. `update-record` 成功后，不要只看 `updated-record.json`；必须重新执行一次 `feishu-bitable record`，确认目标字段真实值是否符合预期。
5. 如果只改一个字段，也要检查关键关联字段是否仍然保留，例如 MR 链接、研发备注、处理人等，不要误以为 CLI 输出里的 `null` 就是实际字段被清空。
6. 清理时仅删除本次任务创建的临时目录；如果常规删除命令被策略拦截，换更精确的删除方式继续完成，不要把清理遗留给用户。
