# GitHub MCP Connectivity Test — Setup Instructions

**Purpose:** Verify that a GitHub repo you own can be reached via MCP (Model Context Protocol) before committing to a use case for the AURA M1 showcase.

**Can Claude Code run this for you?** Yes. Every step below is a shell command, a file edit, or a browser action. Open Claude Code in the folder you create and paste this whole document as your instruction — it can run the `npx`/`docker` commands, create the workflow file, and report back the results. The one thing it cannot do for you is the GitHub PAT creation and the OAuth/browser login — those require you personally, since they're tied to your GitHub identity.

---

## 0. Prerequisites

| Requirement | Why |
|---|---|
| Node.js (v22.7.5 or later recommended) | MCP Inspector runs via `npx` |
| A GitHub account with owner access to at least one repo (or ability to create a new dummy repo) | Target of the connectivity test |
| VS Code (optional but recommended) | Convenient place to run the terminal + edit files |
| Docker (optional — only needed for Path B below) | Only if you test the *local* GitHub MCP server instead of the *remote* one |

You do **not** need Docker for the fastest path (Path A). Start there.

---

## 1. Create your local working folder

```bash
mkdir github-mcp-test
cd github-mcp-test
```

Open this folder in VS Code:

```bash
code .
```

Open a terminal inside VS Code: **Terminal → New Terminal**. All commands below run from this terminal, inside this folder. Nothing needs to be inside a git repo — this folder is just your local scratch space for running the test tooling.

---

## 2. Create a dummy GitHub repo (if you don't already have one to test with)

1. Go to github.com → New repository.
2. Name it something like `aura-mcp-test`.
3. Make it a **private** repo (no need to expose test material publicly).
4. Initialize with a README so it isn't empty.

You said you already have owner access to some repos — you can reuse one of those instead of creating a new one, but a fresh dummy repo is safer since you'll be adding a workflow that's designed to fail on purpose (step 4).

---

## 3. Create a GitHub Personal Access Token (PAT)

**This step must be done by you, in your browser — not by Claude Code.**

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → **Generate new token**.
2. Set **Repository access** to "Only select repositories" → choose your dummy repo (or the repo(s) you're testing).
3. Under **Permissions**, grant at minimum:
   - Contents: Read
   - Issues: Read (Read and write if you want to test issue creation later)
   - Actions: Read (Read and write if you want to test triggering workflows later)
4. Generate the token and **copy it immediately** — GitHub only shows it once.
5. Store it somewhere safe (a password manager, or a local `.env` file that is **git-ignored** — never commit it).

> **Security note:** Never paste your PAT directly into a shared chat, a committed file, or a public repo. If you want Claude Code to use it, set it as an environment variable in your terminal session (see step 4/6) rather than typing it into a file it might save.

---

## 4. Add a sample GitHub Action workflow that fails on purpose

This gives you a real failed run to fetch logs from later — useful because your actual AURA use case is "triage a failure and pull logs," so you want to prove that specific capability works, not just that you can read the repo.

In your dummy repo, create `.github/workflows/test-failure.yml`:

```yaml
name: Test Failure Workflow

on:
  workflow_dispatch:

jobs:
  intentional-fail:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: This step will fail on purpose
        run: |
          echo "Simulating a build failure for MCP log-retrieval testing..."
          exit 1
```

Commit and push this file to the repo. Then trigger it manually:
- Go to the repo → **Actions** tab → "Test Failure Workflow" → **Run workflow**.
- Wait for it to fail (should take under a minute).

You now have a real failed workflow run in a repo you own — exactly what you'll ask the MCP connection to retrieve logs for.

---

## 5. Set your PAT as an environment variable

In the VS Code terminal, inside `github-mcp-test`:

```bash
export GITHUB_PAT="paste-your-token-here"
```

(This only lasts for the current terminal session — that's intentional, so it isn't accidentally saved to disk.)

---

## Path A — Test the Remote GitHub MCP Server (recommended, no Docker needed)

### 6A. Launch MCP Inspector

```bash
npx @modelcontextprotocol/inspector
```

This starts a local web UI (it will print a URL, typically `http://localhost:6274` or similar — open it in your browser if it doesn't open automatically).

### 7A. Connect Inspector to the remote GitHub MCP server

In the Inspector web UI connection pane:
- **Transport:** Streamable HTTP (or "HTTP")
- **URL:** `https://api.githubcopilot.com/mcp/`
- **Headers:** add `Authorization: Bearer <your PAT>` (paste the same token from step 3)
- Click **Connect**

### 8A. Run the verification checklist

In the Inspector UI, go to the **Tools** tab and run these in order:

1. **`get_me`** — confirms which GitHub identity this token resolves to. ✅ if it returns your username.
2. **List repositories** (tool name may be `list_repos` or similar — check the Tools tab for the exact name) — confirms your dummy/owned repo appears. ✅ if you see it in the results.
3. **Get repo contents / file tree** on your dummy repo — confirms read access reaches the specific repo. ✅ if it returns the README and the workflow file you added.
4. **List workflow runs** for your dummy repo — confirms Actions visibility. ✅ if you see the "Test Failure Workflow" run from step 4.
5. **Get logs for that failed run** — this is the critical one for your use case. ✅ if it returns the log output containing "Simulating a build failure..." and the exit code 1.

If all five pass, your connectivity is proven end-to-end for the CI-triage use case.

### 9A. (Optional) Script it for repeatable checks

Instead of clicking through the UI each time, use CLI mode:

```bash
npx @modelcontextprotocol/inspector --cli https://api.githubcopilot.com/mcp/ --method tools/list
```

---

## Path B — Test the Local GitHub MCP Server (only if you need Docker/on-prem, e.g. for GitHub Enterprise Server)

### 6B. Confirm Docker is running

```bash
docker --version
docker ps
```

### 7B. Launch Inspector against the local Dockerized server

```bash
npx @modelcontextprotocol/inspector \
  docker run -i --rm \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=$GITHUB_PAT \
  ghcr.io/github/github-mcp-server
```

Add `-e GITHUB_READ_ONLY=1` right after the PAT line if you want to guarantee no write actions are possible during this test.

### 8B. Run the same verification checklist as step 8A above.

---

## 10. What to do with the results

| Outcome | Next step |
|---|---|
| All 5 checks pass | Connectivity is proven. Finalize the GitHub CI-triage use case for M1 and move to scoping AURA's own MCP client layer. |
| `get_me` fails | PAT issue — regenerate, check it wasn't truncated when copied. |
| Repo doesn't appear in list | PAT repository-access scope doesn't include that repo — go back to step 3 and re-select it. |
| Repo/file read works but workflow-run or log tools fail/are missing | The Actions toolset may need explicit enabling depending on server version — check the Tools tab for an Actions-specific toolset flag, or note this as a gap to flag in your use-case writeup. |
| Everything works except write actions (if you tested any) | Expected if you scoped the PAT to read-only — that's the safer default for a first demo anyway. |

---

## Reference links

- MCP Inspector (official): https://github.com/modelcontextprotocol/inspector
- GitHub MCP Server (official): https://github.com/github/github-mcp-server
- Remote server endpoint: `https://api.githubcopilot.com/mcp/`

---

*Prepared for: AURA M1 use-case validation — GitHub connectivity check*
