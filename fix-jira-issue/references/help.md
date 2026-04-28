# Jira CLI Usage Reference

## Use This Reference

Read this file when you need Jira CLI commands for issue retrieval, field inspection, comments, attachments, or transition discovery.

## View a Single Issue

```bash
# View in the terminal
jira issue view PROJ-123

# View as markdown
jira issue view PROJ-123 --format markdown

# Save to a file
jira issue view PROJ-123 --output ./issue.md

# Save markdown to a file
jira issue view PROJ-123 --format markdown --output ./issue.md
```

## List Issues

```bash
# List recent issues
jira issue list

# List issues assigned to the current user
jira issue list --assignee=currentUser

# Filter by status
jira issue list --status=Open --limit=50

# Filter by project and type
jira issue list --project=TEST --type=Bug

# Use JQL
jira issue list --jql "project = TEST AND status = Open"
```

### Filter Options

| Option | Meaning | Example |
| --- | --- | --- |
| `--project` | Filter by project key | `--project=TEST` |
| `--assignee` | Filter by assignee | `--assignee=currentUser` |
| `--status` | Filter by status | `--status=Open` |
| `--type` | Filter by issue type | `--type=Bug` |
| `--reporter` | Filter by reporter | `--reporter=john.doe` |
| `--priority` | Filter by priority | `--priority=High` |
| `--created` | Filter by created date | `--created=-7d` |
| `--updated` | Filter by updated date | `--updated=-1d` |
| `--limit` | Limit result count | `--limit=50` |
| `--jql` | Custom JQL query | `--jql="..."` |

## Read Specific Fields

```bash
# Read a custom field
jira issue field PROJ-123 customfield_11103

# Read the description
jira issue field PROJ-123 description
```

## Manage Comments

```bash
# List comments
jira issue comment list PROJ-123

# Add a comment
jira issue comment add PROJ-123 "这是一条评论"
```

## Manage Attachments

```bash
# List attachments
jira issue attachment list PROJ-123

# Download all attachments
jira issue attachment download PROJ-123 --output ./tmp/
```

## Discover and Execute Transitions

```bash
# Show available transitions and required fields
jira issue transitions PROJ-123

# Filter detailed transition data, for example by version date
jira issue transitions PROJ-123 --filter-values versions=260316

# Show verbose transition details when necessary
jira issue transitions PROJ-123 --verbose

# Execute a transition with field JSON
jira issue transition PROJ-123 231 --field-json '{"customfield_12323": "审核结论", "assignee": {"name": "user@example.com"}}'
```

### Common Field JSON Shapes

| Field type | JSON shape | Example |
| --- | --- | --- |
| string | `"字段值"` | `"customfield_10309": "根因描述"` |
| option | `{"value": "选项名"}` | `"customfield_11501": {"value": "设计遗漏"}` |
| array | `[{"name": "版本名"}]` | `"versions": [{"name": "v1.0"}]` |
| user | `{"name": "用户名"}` | `"assignee": {"name": "user@example.com"}` |