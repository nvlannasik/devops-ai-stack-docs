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

Each page's top line names its audience.

## Diagrams

Five diagrams, made with [archify](https://github.com/tt-a1i/archify) (v3.2.0), live under `diagrams/<name>/`.

| Diagram | Used on | Files |
|---|---|---|
| Common issue-handling flow | Overview | `issue-flow.png` · `.svg` · `.html` |
| Architecture | Overview, Architecture | `architecture.png` · `.svg` · `.html` |
| Reaching a private LLM | Overview | `private-llm.png` · `.svg` · `.html` |
| How an investigation works | How an Investigation Works | `investigation.png` · `.svg` · `.html` |
| Guarded remediation | Remediation and Safety Guardrails | `remediation.png` · `.svg` · `.html` |

**Demo screenshots and videos.** The Overview's *See it in action* section has a 🖼️ placeholder for each screenshot, named `images/demo-01-…png` to `demo-05-…png`, and a 🎬 *add link* cell for each video. Replace each placeholder line with the image, and each cell with the video link or an embedded video.

**Getting them into Confluence.** The pages reference the PNGs with relative paths, which Confluence does not follow. On each page:
1. Upload the PNG as an attachment, or drag it into the editor.
2. Put it where the `![…](diagrams/…png)` line was.

The `.svg` is the vector version, light theme. The `.html` is the interactive original: zoom, focus and dark mode. Link to it from Confluence if you host it somewhere.

**Changing a diagram.** Edit its `candidate.json`, then re-render and re-check it. Run from this folder, with the archify skill installed in the workspace:

```bash
ARCHIFY_CHROME_NO_SANDBOX=1 ARCHIFY_CHROME=<chrome-or-chromium> \
  node ../.claude/skills/archify/bin/archify.mjs finalize <architecture|workflow> \
  diagrams/<name>/candidate.json diagrams/<name>/<name>.html --quality showcase --json
```

Then export PNG and light SVG from the HTML's **Export** menu. The receipts that `finalize` writes are git-ignored.

Facts are as of 2026-10-21: agent `f423d11`, mcp-server `9989bcb`, chart `0.7.0`.
