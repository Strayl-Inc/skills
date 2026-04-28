---
name: coregit
description: |
  Serverless Git backend for AI-native products. Use this skill when the user wants to create git repos, commit code, read files, diff branches, mount a versioned filesystem for an agent (fs.mount), search code (full-text, semantic, agentic, or hybrid), run shell commands against a repo at any point in history, or build LLM Wiki knowledge bases — without the git CLI. Handles user onboarding (signup, email verification, API key) entirely from the terminal.
metadata:
  author: coregit
  version: "1.1.0"
allowed-tools: Bash(cgt:*) Bash(curl:*) Bash(npx:*)
---

# Coregit

Coregit is a serverless Git hosting platform. Create repos, commit files, read code, diff branches, merge, cherry-pick, search — all via REST API. No git CLI needed.

- **API base:** `https://api.coregit.dev`
- **Dashboard:** `https://app.coregit.dev`
- **Docs:** `https://docs.coregit.dev`
- **SDK:** `npm install @coregit/sdk`
- **CLI:** `npm install -g @coregit/cli` (binary: `cgt`)

## When to use

- User asks to set up a git backend or version control for their app
- User wants to commit, read, or diff files via API (not git CLI)
- User mentions "coregit" by name
- User needs a serverless git repo for an AI agent workflow

## Onboarding a new user

If the user does not have a Coregit API key, run this flow:

### Step 1: Ask for email and name, send verification code

```bash
npx coregit-wizard@latest --email <USER_EMAIL> --name <USER_NAME>
```

This installs the CLI and skill, then sends a 6-digit verification code to the email. Ask the user to check their inbox.

### Step 2: Verify with code and get API key

```bash
npx coregit-wizard@latest --email <USER_EMAIL> --code <6_DIGIT_CODE>
```

This verifies the code, creates the account/org/API key, and stores credentials via `cgt auth login`. If the user already has an account, it creates a new key for their existing org.

### Step 3: Output summary

After setup, output a summary:
- Email and org slug
- Remind the user to save the API key (it is shown only once)
- Link to docs: https://docs.coregit.dev

## Using the CLI (preferred for agents — saves tokens)

After `cgt auth login`, all commands use stored credentials automatically. Output is JSON.

### Repositories

```bash
cgt repos create my-project
cgt repos list
cgt repos get my-project
cgt repos delete my-project
```

### Commit files

```bash
# From local files (path_in_repo:local_file)
cgt commit my-project -b main -m "feat: init" \
  -f src/index.ts:./index.ts -f README.md:./README.md

# Inline content (path:=content)
cgt commit my-project -b main -m "add config" \
  -f config.json:='{"port": 3000}'

# From stdin (JSON array)
echo '[{"path":"src/app.ts","content":"console.log(1)"}]' | cgt commit my-project -b main -m "add app"
```

### Read files

```bash
cgt tree my-project main              # list files
cgt blob my-project main src/app.ts   # read file (JSON)
cgt blob my-project main src/app.ts --raw  # raw content
```

### Branches

```bash
cgt branches create my-project feature-x --from main
cgt branches list my-project
cgt branches merge my-project feature-x
cgt branches delete my-project feature-x
```

### Diff & Compare

```bash
cgt diff my-project main feature-x --patch
cgt compare my-project main feature-x
```

### Commits

```bash
cgt log my-project --ref main --limit 10
```

### Search (full-text)

```bash
cgt search "TODO" --repos my-project --ref main
cgt search "function.*Async" --repos my-project --regex
```

`--ref` accepts branch name or commit SHA.

### Semantic search (AI-powered)

```bash
# 1. Index the repo
cgt index create my-project -b main

# 2. Check status (poll until "ready")
cgt index status my-project

# 3. Search by natural language
cgt semantic-search my-project "user authentication with password hashing" --ref main

# Search at a specific commit
cgt semantic-search my-project "auth logic" --ref 8b7d315321...
```

### Snapshots

```bash
cgt snapshots create my-project v1-backup -b main
cgt snapshots list my-project
cgt snapshots restore my-project v1-backup
```

### Workspace (execute commands)

```bash
cgt exec my-project "npm install && npm run build" -b main --commit --commit-message "build output"
```

### Versioned filesystem mount (SDK only — agent-friendly)

Higher-level wrapper over `/blob`, `/tree`, `/commits`, and `/exec`. Returns a handle with `writeFile`/`readFile`/`bash().exec()`/`commit()` and three commit modes. Use this in agent code where you'd otherwise build commits by hand.

```typescript
import { CoregitClient } from "@coregit/sdk";

const cg = new CoregitClient({ apiKey: process.env.COREGIT_API_KEY! });

const fs = await cg.fs.mount({
  repos: [{ slug: "my-project" }],
  mode: "rw",
  author: { name: "Agent", email: "agent@example.com" },
});

await fs.writeFile("/my-project/src/feature.ts", "export const x = 1;");
const r = await fs.bash().exec("npm test");
console.log(r.stdout, r.exitCode);

await fs.unmount();
```

Three modes for `commitMode`:

- `auto` (default) — every `writeFile` / `rm` / `mv` commits immediately.
- `manual` — writes accumulate in memory; `commit({ message })` flushes them.
- `on-exec` — like `manual`, but `bash().exec()` flushes the buffer into the same exec call so the shell sees buffered writes.

Up to 10 repos per mount; multi-repo paths are namespaced as `/<slug>/<rest>`. Throws `MountError` / `MountReadOnlyError` / `MountPathError` / `MountConflictError` instead of returning `{data, error}`. Full reference: https://docs.coregit.dev/docs/api-reference/filesystem

### Clone with standard git

```bash
git clone https://ORGSLUG:cgk_live_...@api.coregit.dev/ORGSLUG/my-project.git
```

## REST API (alternative to CLI)

All requests require header `x-api-key: cgk_live_...`. Use the CLI above when possible — it's shorter and handles auth automatically.

<details>
<summary>curl examples (click to expand)</summary>

```bash
# Create repo
curl -X POST https://api.coregit.dev/v1/repos \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"slug": "my-project"}'

# Commit files
curl -X POST https://api.coregit.dev/v1/repos/my-project/commits \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"branch":"main","message":"init","author":{"name":"Agent","email":"a@a.dev"},"changes":[{"path":"app.ts","content":"console.log(1)"}]}'

# Search
curl -X POST https://api.coregit.dev/v1/search \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"TODO","repos":["my-project"],"ref":"main"}'

# Semantic search
curl -X POST https://api.coregit.dev/v1/repos/my-project/semantic-search \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"auth logic","ref":"main"}'
```

</details>

## Key API endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /v1/repos | Create repo |
| GET | /v1/repos | List repos |
| POST | /v1/repos/:slug/commits | Commit files |
| GET | /v1/repos/:slug/commits | List commits |
| GET | /v1/repos/:slug/tree/:ref | List files |
| GET | /v1/repos/:slug/blob/:ref/:path | Read file |
| POST | /v1/repos/:slug/branches | Create branch |
| GET | /v1/repos/:slug/branches | List branches |
| POST | /v1/repos/:slug/branches/:name/merge | Merge |
| GET | /v1/repos/:slug/diff/:ref1/:ref2 | Diff |
| POST | /v1/repos/:slug/snapshots | Create snapshot |
| POST | /v1/repos/:slug/exec | Run shell command (accepts `pre_apply_changes`) |
| POST | /v1/workspace/exec | Run shell command across up to 10 repos |
| POST | /v1/repos/:slug/agentic-search | Multi-turn natural-language code Q&A (paid) |
| POST | /v1/search | Cross-repo full-text search |
| POST | /v1/repos/:slug/semantic-search | AI semantic search |
| POST | /v1/repos/:slug/index | Trigger semantic index |
| GET | /v1/repos/:slug/index/status | Index status |
| POST | /v1/repos/:slug/hybrid-search | Fused semantic + graph + keyword search |
| POST | /v1/repos/:slug/graph/query | Code graph queries (callers, callees, impact, …) |

## LLM Wiki (knowledge bases)

Persistent, version-controlled knowledge bases maintained by LLM agents. Based on the LLM Wiki pattern by Andrej Karpathy. A wiki is a regular Coregit repo with this layout:

```
wiki.json   # Configuration (read by /llms.txt)
schema.md   # LLM instructions (optional, free-form)
raw/        # Immutable source documents
raw/docs/   # Markdown converted from uploaded files
wiki/       # LLM-generated pages
```

### Create a wiki

There's no dedicated `wiki init` endpoint — create a regular repo, then commit the initial layout:

```bash
cgt repos create my-research

cat > /tmp/wiki.json <<'EOF'
{
  "version": 1,
  "title": "AI Research",
  "description": "Deep dive into transformers",
  "llms_txt": { "include_sources": false, "max_pages": 500, "sort": "updated" }
}
EOF

cgt commit my-research -b main -m "init: wiki layout" \
  -f wiki.json:/tmp/wiki.json \
  -f raw/.gitkeep:='' \
  -f wiki/.gitkeep:=''
```

### Wiki read operations (CLI)

```bash
# List pages (with parsed frontmatter)
cgt wiki pages my-research

# Read a page (with or without leading "wiki/")
cgt wiki page my-research wiki/transformers.md
cgt wiki page my-research transformers.md --raw

# List and read raw sources
cgt wiki sources my-research
cgt wiki source my-research raw/attention-paper.md --raw

# Generate llms.txt for external LLM consumption
cgt wiki llms-txt my-research --format full

# Knowledge graph (nodes, edges, tag clusters)
cgt wiki graph my-research

# Health stats (pages, sources, orphans, word counts)
cgt wiki stats my-research
```

### Querying the wiki with natural language

There's no `wiki search` — for Q&A across both wiki pages and raw sources, use **agentic search** (paid tier). It runs a multi-turn grep/read loop and returns an answer with file locations:

```bash
cgt agentic my-research "how does multi-head attention work?"
```

Or via SDK:

```typescript
const { data } = await cg.search.agentic("my-research", {
  q: "how does multi-head attention work?",
});
console.log(data.answer, data.locations);
```

### Document upload (PDF / DOCX / image → markdown)

Convert a binary document via Cloudflare Workers AI, commit it to `raw/docs/`, and auto-enqueue an ingest workflow run. Multipart form, 25 MB cap. Supported: pdf, docx, xlsx, csv, html, png, jpg, etc.

```bash
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/documents \
  -H "x-api-key: $COREGIT_API_KEY" \
  -F "file=@./paper.pdf" \
  -F "description=Attention Is All You Need"
# → { path, original_filename, bytes, markdown_length, run_id }
```

### Workflow runs (sync / dream / lint)

```bash
# Start a sync run — re-synthesize wiki pages from sources
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/run \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode":"sync","source_ids":["raw/attention-paper.md"]}'
# → { run_id, status: "queued" }

# Poll status
curl https://api.coregit.dev/v1/repos/my-research/wiki/runs/<run_id> \
  -H "x-api-key: $COREGIT_API_KEY"

# Cancel
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/cancel \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_id":"<run_id>"}'
```

Modes: `sync` (re-synthesize from sources), `dream` (LLM proposes new pages), `lint` (cleanup suggestions).

### Page history & restore

```bash
# Activity log for a single page
curl "https://api.coregit.dev/v1/repos/my-research/wiki/pages/wiki/transformers.md/history?limit=10" \
  -H "x-api-key: $COREGIT_API_KEY"

# Read page as it was at a date
curl "https://api.coregit.dev/v1/repos/my-research/wiki/pages/wiki/transformers.md/as-of?as_of=2026-03-15T00:00:00Z" \
  -H "x-api-key: $COREGIT_API_KEY"

# Restore page to a past version
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/pages/wiki/transformers.md/restore \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"version_id":"5e7a2c8b..."}'
```

### Sandboxes (review-before-promote)

A sandbox is a `wiki-sandbox/<name>` branch — agent drafts changes there, human reviews, then promotes to `main`.

```bash
# Create
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/sandboxes \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"draft-attention-rewrite","from":"main"}'

# List, promote (merge into main), delete
curl https://api.coregit.dev/v1/repos/my-research/wiki/sandboxes -H "x-api-key: $COREGIT_API_KEY"
curl -X POST https://api.coregit.dev/v1/repos/my-research/wiki/sandboxes/draft-attention-rewrite/promote -H "x-api-key: $COREGIT_API_KEY"
curl -X DELETE https://api.coregit.dev/v1/repos/my-research/wiki/sandboxes/draft-attention-rewrite -H "x-api-key: $COREGIT_API_KEY"
```

### Ingest workflow (agent writes wiki pages via commits)

```bash
# 1. Add a source to raw/
cgt commit my-research -b main -m "add source" \
  -f raw/paper.md:./paper.md

# 2. Agent processes source → creates wiki pages in one atomic commit
echo '[
  {"path":"wiki/transformers.md","content":"---\ntitle: \"Transformers\"\nsummary: \"...\"\ntags: [deep-learning]\ntype: concept\nsources: [raw/paper.md]\nupdated: \"2026-04-10\"\n---\n\n## Truth\n\nContent...\n\n## Timeline\n\n- **2026-04-10 (ingest)**: created from raw/paper.md."}
]' | cgt commit my-research -b main -m "ingest: paper"
```

The dual-section `## Truth` / `## Timeline` markers split the page into a canonical "current state" (`compiled_truth`) and an append-only history (`timeline`). Both are returned by `GET /wiki/pages/:path`.

### Wiki API endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /v1/repos/:slug/wiki/pages | List pages with frontmatter |
| GET | /v1/repos/:slug/wiki/pages/:path | Read single page (parsed) |
| GET | /v1/repos/:slug/wiki/pages/:path/history | Activity log |
| GET | /v1/repos/:slug/wiki/pages/:path/as-of | Read at past timestamp |
| POST | /v1/repos/:slug/wiki/pages/:path/restore | Restore past version |
| GET | /v1/repos/:slug/wiki/sources | List raw sources |
| GET | /v1/repos/:slug/wiki/sources/:path | Read raw source |
| GET | /v1/repos/:slug/wiki/llms.txt | Auto-generated llms.txt |
| GET | /v1/repos/:slug/wiki/graph | Knowledge graph |
| GET | /v1/repos/:slug/wiki/stats | Wiki health stats |
| POST | /v1/repos/:slug/wiki/documents | Upload PDF/DOCX/etc → markdown |
| GET | /v1/repos/:slug/wiki/documents | List uploaded docs |
| POST | /v1/repos/:slug/wiki/run | Start sync/dream/lint workflow |
| POST | /v1/repos/:slug/wiki/cancel | Cancel a run |
| GET | /v1/repos/:slug/wiki/runs | List recent runs |
| GET | /v1/repos/:slug/wiki/runs/:id | Single run detail |
| POST | /v1/repos/:slug/wiki/sandboxes | Create sandbox branch |
| GET | /v1/repos/:slug/wiki/sandboxes | List sandboxes |
| POST | /v1/repos/:slug/wiki/sandboxes/:name/promote | Merge sandbox → main |
| DELETE | /v1/repos/:slug/wiki/sandboxes/:name | Drop sandbox |

All routes also accept `/v1/repos/:namespace/:slug/wiki/*` for namespaced repos.

## Important notes

- The API key is shown only once during onboarding. Always remind the user to save it.
- All file commits are atomic — multiple files in one API call.
- Use `parent_sha` in commit requests for conflict detection (returns 409 if branch moved).
- Snapshots are named restore points — create one before risky operations.
- LLM Wiki pages use YAML frontmatter (title, summary, tags, sources, type, related); the dual-section pattern splits page body into `## Truth` (current state) and `## Timeline` (append-only history).
- Wiki writes go through the standard commits API — wiki endpoints are read/lifecycle only.
- For natural-language Q&A across a wiki (or any repo), use **agentic search** (`POST /agentic-search`, paid tier). There is no `wiki search` endpoint.
- For agent code that does many small writes, prefer **`coregit.fs.mount()`** in `manual` or `on-exec` mode over building commits by hand — it batches writes into one atomic commit per flush.
- The `/exec` and `/workspace/exec` endpoints accept an optional `pre_apply_changes` field that layers buffered writes onto the in-memory overlay before bash runs (used by `fs.mount()` in `manual`/`on-exec` mode).
- Full docs at https://docs.coregit.dev
- LLM Wiki guide: https://docs.coregit.dev/docs/guides/llm-wiki
- Agent onboarding guide: https://docs.coregit.dev/docs/getting-started/agent-onboarding
- TypeScript SDK: https://docs.coregit.dev/docs/getting-started/typescript-sdk
- Filesystem mount (fs.mount): https://docs.coregit.dev/docs/api-reference/filesystem
- API reference: https://docs.coregit.dev/docs/api-reference
