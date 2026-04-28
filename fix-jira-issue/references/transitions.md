# Jira Issue Transition Reference

## Use This Reference

Read this file after the fix is complete and the user wants to move the Jira issue to the next workflow state.

## Transition Workflow

1. Show available transitions and field requirements:

```bash
jira issue transitions <issue-key>
```

2. Fetch more detailed option data when needed:

```bash
jira issue transitions <issue-key> --filter-values versions=<YYMMDD>
```

Use `--verbose` only when the regular output and filtered output are still insufficient, because it can add a lot of context.

```bash
jira issue transitions <issue-key> --verbose
```

3. Show the available transition list to the user and let them choose the target transition.

4. Pre-fill field values from the issue details and the code change. Confirm the final payload with the user before executing:

```bash
jira issue transition <issue-key> <transitionId> --field-json '{...}'
```

## Bug Fix Field Pre-Fill Rules

Derive candidate values from the issue description and the implemented change.

| Field | Pre-fill strategy |
| --- | --- |
| 影响版本 | Search for the version that matches the current date |
| 问题产生原因 | Infer from the issue type and fix nature |
| 问题根因 | Extract from the issue title and description |
| 解决方案 | Summarize from the code changes |
| 修改影响分析 | Assess from the change scope |
| 自验方案/结果 | Generate from the applied fix and validation steps |

### Rule for `问题产生原因 (customfield_11501)`

- Do not default to `设计遗漏`
- If the issue is caused by implementation mistakes, missing edge-case handling, state sync problems, indexing, caching, closures, or type handling, prefer `开发遗漏`
- Use `设计遗漏` only when the root cause is clearly missing product rules, UX rules, architecture decisions, or design logic
- If the issue details and code changes are not enough to judge reliably, stop and ask the user instead of guessing
- Before transition, self-check the root cause against the fix:
  - Implementation-layer cause and no design change -> usually `开发遗漏`
  - Design-layer cause and rule or scheme changed -> usually `设计遗漏`

## Common Field JSON Shapes

| Field type | JSON shape | Example |
| --- | --- | --- |
| string | `"字段值"` | `"customfield_10309": "根因描述"` |
| option | `{"value": "选项名"}` | `"customfield_11501": {"value": "设计遗漏"}` |
| array | `[{"name": "版本名"}]` | `"versions": [{"name": "v1.0"}]` |
| user | `{"name": "用户名"}` | `"assignee": {"name": "john"}` |