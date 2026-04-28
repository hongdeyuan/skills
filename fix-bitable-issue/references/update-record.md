# Update Record Reference

只有在用户明确确认修复代码和行为符合预期之后，才使用这里的步骤回写 bitable record。

## 输入要求

- `table-url`: 可选。和 `record` 一样，未传时先让 CLI 自己从环境变量或 `.env.*` 文件读取
- `record-id`: 记录 ID，例如 `recxxxxxxxx`

`update-record` 用的是 `record-id`，不是 `record-url`。优先从 `feishu-bitable record` 的输出里读取 `recordId`。

## 先查字段定义

先用 `fields` 查询当前表的字段定义，再决定回写哪些字段，不要直接从 `record.json` 猜字段名：

```bash
feishu-bitable fields \
  --output ./tmp/bitable-record-<record-id-or-date>/fields.json
```

如果 CLI 自动读取失败，或者用户明确提供了 `table-url`，再显式传入：

```bash
feishu-bitable fields \
  "<table-url>" \
  --output ./tmp/bitable-record-<record-id-or-date>/fields.json
```

注意：

- `fields.json` 的顶层通常是对象，不是数组；字段列表通常在 `items` 里。
- 字段名通常读取 `field_name`，不是 `fieldName`。
- 单选字段的可写值必须从 `property.options[].name` 里读取，不要凭经验猜。

重点确认这些信息：

- 当前记录里哪个字段表示状态
- 目标状态值是什么，不要臆造枚举值
- 是否存在 MR 链接、MR 编号、处理备注、处理人、完成时间等也需要同步回写的字段
- 字段类型是否允许直接写入当前值格式

如果用户没有明确给出状态字段名或目标值，先结合 `fields.json` 和 `record.json` 一起判断；仍不明确时，向用户确认。

如果表里有 MR 链接、MR 编号、处理备注等字段，也在这一步一并更新。

如果用户只是说“回写状态”或“同步 bitable”，不要默认把状态写成“已回归”。这类表里“修复完成”和“测试回归完成”往往是两个不同阶段，必须按字段选项和用户原话判断。

## 更新记录

默认先直接使用下面的命令：

```bash
feishu-bitable update-record \
  "<record-id>" \
  --fields '{"状态":"已修复"}' \
  --output ./tmp/bitable-record-<record-id-or-date>/updated-record.json
```

如果 CLI 自动读取失败，或者用户明确提供了 `table-url`，再显式传入。字段较多时，优先写入 JSON 文件再更新：

```bash
feishu-bitable update-record \
  "<table-url>" \
  "<record-id>" \
  --fields-file ./tmp/bitable-record-<record-id-or-date>/update-fields.json \
  --output ./tmp/bitable-record-<record-id-or-date>/updated-record.json
```

补充规则：

- 只改一个简单字段时，可以用 `--fields`。
- 只要字段值包含多行文本、引号、`!`、URL，或者一次更新多个字段，就优先使用 `--fields-file`。
- 文本字段在 `record.json` 里可能显示为富文本数组，但回写时很多场景直接传字符串即可；是否生效以后续回读结果为准。

仅当用户明确接受最终一致性风险时，才使用 `--ignore-consistency-check`。

## 回写后验收

`update-record` 成功只表示接口调用成功，不代表你已经正确理解了返回结构。回写后至少做这两步：

1. 先查看 `updated-record.json`，确认命令没有报错。
2. 再重新执行一次 `feishu-bitable record` 拉取同一条记录，直接检查目标字段的真实值。

推荐做法：

```bash
feishu-bitable record \
  "<record-url>" \
  --output ./tmp/bitable-record-<record-id-or-date>-verify
```

重点核对：

- 目标状态字段是否真的是预期值
- MR 链接、研发备注、处理人等关联字段是否仍然存在
- 不要因为 `updated-record.json` 里出现 `null` 就断定字段被清空，先以重新拉取的 `record.json` 为准

## 清理

回写完成后，只清理本次任务生成的临时目录，例如：

- `./tmp/bitable-record-<record-id>`
- `./tmp/bitable-record-<record-id>-verify`

不要清空整个 `./tmp/`。如果通用删除命令被策略拦截，改用更精确的目录删除方式继续完成。
