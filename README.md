# Confluence pages — DevOps AI Agent Stack

Seven pages, meant as one parent page and six children, in this order:

| File | Confluence page title | Parent |
|---|---|---|
| `00-overview.md` | DevOps AI Agent Stack — Overview | (space root) |
| `01-architecture.md` | Architecture | Overview |
| `02-investigation.md` | How an Investigation Works | Overview |
| `03-remediation-and-safety.md` | Remediation and Safety Guardrails | Overview |
| `04-deployment-and-configuration.md` | Deployment and Configuration | Overview |
| `05-operations.md` | Operations Guide | Overview |
| `06-slack-user-guide.md` | Slack User Guide | Overview |

## Importing

**Confluence Cloud:**
1. Create the page.
2. Paste the file's content into the editor. Markdown is converted on paste: headings, tables, lists and code blocks.
3. Remove the first `# Title` line if the page title already says it.

**Alternatively**, insert a *Markdown* macro (`/markdown`) and paste the file into it. This keeps the source as-is.

The diagrams are plain-text code blocks, so they render the same everywhere. Each page's top line names its audience.

Facts are as of 2026-10-21: agent `f423d11`, mcp-server `9989bcb`, chart `0.7.0`.
