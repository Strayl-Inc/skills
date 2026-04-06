---
name: coregit
description: |
  Serverless Git backend for AI-native products. Use this skill when the user wants to create git repos, commit code, read files, diff branches, or set up version control via API — without the git CLI. Handles user onboarding (signup, email verification, API key) entirely from the terminal.
metadata:
  author: coregit
  version: "1.0.0"
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

### Step 1: Ask for email

Ask the user for their email address.

### Step 2: Send verification code

```bash
curl -s -X POST https://app.coregit.dev/api/auth/agent/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "<USER_EMAIL>", "name": "<USER_NAME>"}'
```

Response: `{"ok": true, "message": "Verification code sent to <email>"}`

Tell the user to check their inbox for a 6-digit code.

### Step 3: Verify and get API key

```bash
curl -s -X POST https://app.coregit.dev/api/auth/agent/verify \
  -H "Content-Type: application/json" \
  -d '{"email": "<USER_EMAIL>", "code": "<6_DIGIT_CODE>"}'
```

Response:
```json
{
  "ok": true,
  "api_key": "cgk_live_...",
  "org": {"id": "...", "name": "...", "slug": "..."},
  "user": {"id": "...", "email": "..."},
  "api_base_url": "https://api.coregit.dev"
}
```

This auto-creates the account, organization, and API key. If the user already has an account, it creates a new key for their existing org.

### Step 4: Store credentials

```bash
cgt auth login --api-key <API_KEY>
```

Or set the environment variable:
```bash
export COREGIT_API_KEY=cgk_live_...
```

### Step 5: Output summary

After setup, output a summary:
- Email and org slug
- Remind the user to save the API key (it is shown only once)
- Link to docs: https://docs.coregit.dev

## Using the API

All requests require header `x-api-key: cgk_live_...`

### Create a repository

```bash
curl -X POST https://api.coregit.dev/v1/repos \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"slug": "my-project", "visibility": "private"}'
```

### Commit files (single API call, multiple files)

```bash
curl -X POST https://api.coregit.dev/v1/repos/my-project/commits \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "branch": "main",
    "message": "feat: initial code",
    "changes": [
      {"path": "src/index.ts", "content": "console.log(\"hello\")"},
      {"path": "README.md", "content": "# My Project"}
    ]
  }'
```

### Read a file

```bash
curl https://api.coregit.dev/v1/repos/my-project/blob/main/src/index.ts \
  -H "x-api-key: $COREGIT_API_KEY"
```

### List files

```bash
curl https://api.coregit.dev/v1/repos/my-project/tree/main \
  -H "x-api-key: $COREGIT_API_KEY"
```

### Create a branch

```bash
curl -X POST https://api.coregit.dev/v1/repos/my-project/branches \
  -H "x-api-key: $COREGIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "feature-x", "from": "main"}'
```

### Diff two branches

```bash
curl https://api.coregit.dev/v1/repos/my-project/diff/main/feature-x \
  -H "x-api-key: $COREGIT_API_KEY"
```

### Clone with standard git

```bash
git clone https://ORGSLUG:cgk_live_...@api.coregit.dev/ORGSLUG/my-project.git
```

## TypeScript SDK

```typescript
import { createCoregitClient } from "@coregit/sdk";

const git = createCoregitClient({ apiKey: process.env.COREGIT_API_KEY! });

// Create repo
await git.repos.create({ slug: "my-project" });

// Commit multiple files atomically
await git.commits.create("my-project", {
  branch: "main",
  message: "feat: add auth",
  changes: [
    { path: "src/auth.ts", content: "export function login() {}" },
    { path: "src/config.ts", content: '{"apiUrl": "https://api.example.com"}' },
  ],
});

// Read file
const { data } = await git.files.blob("my-project", "main", "src/auth.ts");
console.log(data.content);
```

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
| POST | /v1/repos/:slug/exec | Run shell command |
| POST | /v1/search | Cross-repo search |

## Important notes

- The API key is shown only once during onboarding. Always remind the user to save it.
- All file commits are atomic — multiple files in one API call.
- Use `parent_sha` in commit requests for conflict detection (returns 409 if branch moved).
- Snapshots are named restore points — create one before risky operations.
- Full docs at https://docs.coregit.dev
- Agent onboarding guide: https://docs.coregit.dev/docs/getting-started/agent-onboarding
- TypeScript SDK: https://docs.coregit.dev/docs/getting-started/typescript-sdk
- API reference: https://docs.coregit.dev/docs/api-reference
