# GrayMatter Agent Patterns

A comprehensive reference repository for [GrayMatter](https://github.com/angelnicolasc/graymatter) - the persistent memory system for AI agents.

## What is This?

This repository contains the definitive `AGENTS.md` guide for working with GrayMatter memory workspaces. It's designed to help AI agents (Claude Code, Cursor, OpenCode, etc.) quickly understand and utilize GrayMatter's persistent memory capabilities.

## Quick Links

- **GrayMatter GitHub**: https://github.com/angelnicolasc/graymatter
- **Latest Version**: v0.5.1 (April 2026)
- **Go Docs**: https://pkg.go.dev/github.com/angelnicolasc/graymatter
- **MCP Docs**: https://docs.opencode.ai/graymatter

## Repository Structure

```
.
├── AGENTS.md           # Comprehensive agent instructions
├── README.md           # This file
└── .gitignore          # Excludes .graymatter/ data directory
```

## Getting Started

### For Agents

Read [`AGENTS.md`](./AGENTS.md) to understand:
- How GrayMatter memory system works
- Available MCP tools and their usage
- CLI commands and configuration
- Best practices for memory management

### For Users

```bash
# Install GrayMatter binary (macOS example)
curl -sSL -o graymatter.tar.gz https://github.com/angelnicolasc/graymatter/releases/download/v0.5.1/graymatter_0.5.1_darwin_arm64.tar.gz
tar -xzf graymatter.tar.gz
sudo mv graymatter /usr/local/bin/

# Initialize in any repository
cd your-project
grymatter init

# Restart your editor to activate MCP server
```

## Key Features Covered in AGENTS.md

- **90% token reduction** via intelligent memory consolidation
- **Hybrid retrieval** (vector + keyword + recency)
- **Self-curating memory** via `memory_reflect` MCP tool
- **Session continuity** with checkpoints
- **Cross-agent shared memory** support
- **Live TUI dashboard** for observability
- **Multiple embedding providers** (Ollama, OpenAI, Anthropic, keyword-only)

## Why This Repository?

GrayMatter is powerful but has many moving parts. This `AGENTS.md` serves as:

1. **Quick Reference** - All essential commands and patterns
2. **Onboarding Guide** - For agents new to the workspace
3. **Best Practices** - Memory management and configuration
4. **Troubleshooting** - Common issues and solutions

## License

MIT - See [GrayMatter repository](https://github.com/angelnicolasc/graymatter) for full license.

## Contributing

This is a reference repository. For GrayMatter contributions, see the [main repository](https://github.com/angelnicolasc/graymatter).

---

**Note**: The `.graymatter/` directory is excluded from git as it contains binary memory data specific to each workspace.
