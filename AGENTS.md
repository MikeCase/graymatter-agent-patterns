# AGENTS.md

## What This Repository Is

This is a **GrayMatter memory workspace** - a local-first knowledge base that persists across OpenCode sessions using SQLite + vector embeddings. Not an application codebase.

**GrayMatter** (the underlying system) is an MCP server + Go library that gives AI agents persistent memory while reducing token usage by ~90%.

- **GitHub**: https://github.com/angelnicolasc/graymatter
- **Latest**: v0.5.1 (April 2026)
- **Language**: Go (100%)
- **Storage**: bbolt (SQLite-like) + chromem-go (vectors)
- **License**: MIT

## Quick Start

If GrayMatter binary is not installed:

```bash
# macOS (Apple Silicon)
curl -sSL -o graymatter.tar.gz https://github.com/angelnicolasc/graymatter/releases/download/v0.5.1/graymatter_0.5.1_darwin_arm64.tar.gz
tar -xzf graymatter.tar.gz
sudo mv graymatter /usr/local/bin/

# Initialize in this repo
graymatter init
```

Then **restart your editor** to pick up the MCP server.

## GrayMatter Memory System

GrayMatter provides persistent memory for agents through these MCP tools:

| Tool | Purpose |
|------|---------|
| `memory_search` | Recall facts for a query (semantic + keyword + recency) |
| `memory_add` | Store a new fact |
| `checkpoint_save` | Snapshot current session state |
| `checkpoint_resume` | Restore last checkpoint |
| `memory_reflect` | Add / update / forget / link memories (agent self-edit) |

### Memory Storage Location

```
.graymatter/
├── gray.db      # SQLite database (notes, metadata, facts)
└── vectors/     # Vector embeddings for semantic search (chromem-go)
```

**Storage Details**:
- Single file: `.graymatter/gray.db` (bbolt - pure Go, ACID)
- Single folder: `.graymatter/vectors/` (chromem-go embeddings)
- No migrations, no schema versions
- Append-only with decay-based eviction

### When to Use Memory

**DO store:**
- User preferences and workflow patterns
- Project conventions learned during session
- Bug fixes or workarounds discovered
- Architecture decisions and their rationale

**DO NOT store:**
- Information already in AGENTS.md or code
- Temporary debugging state
- Anything that should be in version-controlled docs

### Memory Query Patterns

```python
# Search for relevant context before acting
graymatter_memory_search(
    agent_id="{your_agent_id}",
    query="user preference for error handling",
    top_k=5
)

# Store new learnings
graymatter_memory_add(
    agent_id="{your_agent_id}",
    text="User prefers explicit error types over exceptions"
)

# Self-curation: update or forget memories
graymatter_memory_reflect(
    agent_id="{your_agent_id}",
    action="update",  # or "add", "forget", "link"
    text="Updated preference: user now likes exceptions"
)
```

### How Memories Get Stored

There are **four** paths into the store:

| Path | Caller | Use Case |
|------|--------|----------|
| `memory_add` (MCP) | Agent/you | Known string worth keeping |
| `memory_extract` | GrayMatter | Pull atomic facts from LLM output automatically |
| `memory_reflect` (MCP) | LLM itself | Agent self-curates during session |
| `Consolidate` (async) | Background | Summarise, decay, prune over time |

## Working With This Repo

### No Build/Test Commands

This repo has no build system - it's pure data. Do not run `npm install`, `pip install`, etc.

The actual GrayMatter codebase lives at https://github.com/angelnicolasc/graymatter.

### File Operations

- The `.graymatter/` directory contains binary data - do not edit manually
- Safe to read `gray.db` with SQLite tools if needed
- The `vectors/` directory will populate as memories are added

### Repository Purpose

This is a workspace for testing/exploring GrayMatter integration with agents. The GrayMatter implementation itself lives in the OpenCode platform and the GitHub repo above.

## Session Continuity

When continuing work:

1. Check existing memories first: `graymatter_memory_search(agent_id="...", query="...")`
2. Restore checkpoint if available: `graymatter_checkpoint_resume(agent_id="...")`
3. Add new observations before session ends

## CLI Reference

Available GrayMatter CLI commands:

```bash
graymatter init                                    # create .graymatter/ + .mcp.json
graymatter remember "agent" "text to remember"    # store a fact
graymatter remember --shared "text"               # store in shared namespace (all agents)
graymatter recall   "agent" "query"               # print context
graymatter recall   --all "agent" "query"         # merge agent + shared memory
graymatter checkpoint list    "agent"             # show saved checkpoints
graymatter checkpoint resume  "agent"             # print latest checkpoint as JSON
graymatter mcp serve                              # start MCP server (stdio)
graymatter mcp serve --http :8080                 # HTTP transport
graymatter export --format obsidian --out ~/vault # dump to Obsidian vault
graymatter tui                                    # 4-view terminal UI (live dashboard)
graymatter run agent.md [--background]            # run a SKILL.md agent file
graymatter sessions list                          # list managed agent sessions
graymatter plugin install manifest.json           # install a plugin
graymatter server --addr :8080                    # REST API server
```

Global flags: `--dir` (data dir), `--quiet`, `--json`

## Configuration

GrayMatter auto-detects embedding providers in this order:
1. **Ollama** (default) - if running with `nomic-embed-text`
2. **OpenAI** - if `OPENAI_API_KEY` set
3. **Anthropic** - if `ANTHROPIC_API_KEY` set
4. **Keyword-only** - TF-IDF + recency, zero deps (fallback)

### Setup Commands

```bash
# Pull embedding model (Ollama - recommended)
ollama pull nomic-embed-text

# Or use API keys
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=sk-ant-...
```

## Memory Lifecycle

```
Recall(agent, task)          ← hybrid: vector + keyword + recency → top-8 facts
    ↓
Inject into system prompt    ← automatic via MCP
    ↓
Agent runs
    ↓
Remember(agent, observation) ← store key facts
    ↓
Consolidate() [async]        ← summarise + decay + prune (optional LLM)
```

- **Consolidation** auto-enables when `ANTHROPIC_API_KEY` is set
- Uses `ConsolidateThreshold` (default: 20 facts) to trigger
- Reduces tokens ~90% vs full-history injection at 100 sessions

## Token Efficiency

| Sessions | Full Injection | GrayMatter | Reduction |
|----------|---------------|------------|-----------|
| 1        | ~80 tokens    | ~80 tokens | 0%        |
| 10       | ~630 tokens   | ~550 tokens | 12%      |
| 30       | ~1,880 tokens | ~550 tokens | 71%      |
| 100      | ~6,960 tokens | ~670 tokens | **90%**  |

## TUI Dashboard

Run `graymatter tui` for live observability:

- **Facts** - total stored, distributed across agents
- **Memory cost** - KB on disk (not tokens)
- **Recalls** - cumulative access count
- **Health** - % of facts above relevance threshold (weight > 0.5)
- **Token cost (30d)** - real spend by model
- **Agent activity** - facts vs recalls per agent
- **Weight distribution** - consolidation over time
- **Activity timeline** - facts created per day

Keys: `1-4` switch tabs, `r` refresh, `q` quit. Auto-refreshes every 5s.

## What GrayMatter Is NOT

- **Not** a framework or agent runner
- **Not** a hosted service or SaaS
- **Not** tied to any vendor (MCP standard)
- **Not** a knowledge base UI (Notion/Obsidian alternative)

It is: **the missing stateful layer for Go CLI agents**.

## Resources

- **GitHub**: https://github.com/angelnicolasc/graymatter
- **Go Docs**: https://pkg.go.dev/github.com/angelnicolasc/graymatter
- **MCP Docs**: https://docs.opencode.ai/graymatter
- **Releases**: https://github.com/angelnicolasc/graymatter/releases
