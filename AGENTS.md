# AGENTS.md — GrayMatter Agent Memory Guide

## Philosophy

GrayMatter is your long-term memory. Unlike conversation context which disappears when the session ends, GrayMatter facts persist across sessions, projects, and even agent restarts. Use it to accumulate knowledge that makes you more effective over time.

**Key principle**: Store *conclusions*, not *conversations*. A good memory is something you would want injected into your system prompt on day 1 of a new session.

---

## When to Use Memory

### ALWAYS Store

- **User preferences** — coding style, communication preferences, tool choices
- **Project conventions** — "this repo uses tabs not spaces", "never use X library"
- **Architecture decisions** — "chose PostgreSQL over MySQL because..."
- **Bug fixes & workarounds** — "fixed by upgrading to v2.3, don't downgrade"
- **Recurring patterns** — "user always asks for TypeScript examples first"
- **Environment quirks** — "needs NODE_OPTIONS=--max-old-space-size=4096"
- **Stakeholder info** — "CTO prefers detailed explanations, CEO wants summaries"

### NEVER Store

- **Transient state** — current file being edited, temporary variables
- **Conversation logs** — raw back-and-forth without conclusions
- **Duplicate information** — already in README, AGENTS.md, or code comments
- **Speculative thoughts** — "maybe we should try X" (store after decision)
- **Secrets or credentials** — use proper secret management
- **Large outputs** — store the insight, not the 500-line stack trace
- **Session-specific context** — "line 42 has an error" (use checkpoint instead)

### Decision Framework

```
About to store something?
├── Is it a conclusion/fact/preference?          → YES, store it
├── Is it raw conversation without insight?      → NO, extract insight first
├── Is it already documented in code/README?     → NO, reference existing docs
├── Will this still matter in 10 sessions?       → YES, store it
├── Is it temporary debugging state?             → NO, use checkpoint
└── Is it a secret/credential?                   → NO, never store in memory
```

---

## Memory Operations

### `memory_add` — Store a Clean Fact

Use when you have a single, atomic, well-formed fact.

**Good examples:**
```python
graymatter_memory_add(
    agent_id="frontend-agent",
    text="User prefers Tailwind CSS over styled-components"
)

graymatter_memory_add(
    agent_id="backend-agent", 
    text="API rate limit: 100 req/min, exceeded returns 429 with Retry-After header"
)
```

**Bad examples:**
```python
# TOO VAGUE
graymatter_memory_add(agent_id="agent", text="user likes things")

# CONVERSATION LOG
graymatter_memory_add(agent_id="agent", text="User: Can you help? Agent: Sure, what do you need?")

# DUPLICATE
graymatter_memory_add(agent_id="agent", text="Project uses React (already in README)")
```

### `memory_search` — Retrieve Relevant Context

Always search before acting on ambiguous requests. The query should describe what you're trying to do, not be a keyword search.

**Good queries:**
```python
# Describe the task, not search terms
graymatter_memory_search(
    agent_id="frontend-agent",
    query="how should I style this component",
    top_k=5
)

# Include context about current work
graymatter_memory_search(
    agent_id="backend-agent",
    query="authentication middleware patterns for this project",
    top_k=8
)
```

**How it works internally:**
- Hybrid ranking: vector similarity (50%) + keyword relevance (50%) + recency (25%)
- Returns top-K facts, deduplicated by text
- Access metadata updated asynchronously (AccessCount++, AccessedAt=now)
- If knowledge graph is enabled, neighbor entities are appended

**Query strategies:**
```python
# Strategy 1: Broad context gathering (top_k=8)
# Use when starting a new task to gather all relevant background

# Strategy 2: Focused lookup (top_k=3)
# Use when you need a specific piece of information

# Strategy 3: Multi-query fusion
# Issue 2-3 related queries and merge unique results
results1 = memory_search(agent_id, "user preference for error handling")
results2 = memory_search(agent_id, "user preference for logging")
combined = list(dict.fromkeys(results1 + results2))  # dedupe, preserve order
```

### `memory_reflect` — Self-Curation

This is the most powerful tool. Use it to maintain memory quality over time.

**Actions:**

| Action | When to Use |
|--------|-------------|
| `add` | Same as `memory_add` — store a new fact |
| `update` | Found a better/corrected version of an existing fact |
| `forget` | A fact is wrong, outdated, or no longer relevant |
| `link` | Two entities are related (knowledge graph) |

**Update workflow:**
```python
# 1. Search for the old fact
old_facts = memory_search(agent_id, "old information about X")

# 2. Update it (zeros old weight, adds new)
memory_reflect(
    agent_id="agent",
    action="update",
    text="CORRECTED: API base URL is https://api.v2.example.com (not v1)"
)

# 3. Old fact still exists with weight=0 until next consolidation prunes it
```

**Forget workflow:**
```python
# When a workaround is no longer needed
memory_reflect(
    agent_id="agent",
    action="forget",
    text="Workaround for Node 14 bug (project now uses Node 18)"
)
```

**Link workflow (knowledge graph):**
```python
# Connect related concepts
memory_reflect(
    agent_id="agent",
    action="link",
    text="Authentication service",
    target="User database"
)
```

### `checkpoint_save` / `checkpoint_resume` — Session Continuity

Use for long-running tasks that might span multiple sessions or get interrupted.

**What checkpoints capture:**
- Arbitrary state dict (your choice of structure)
- Conversation history (messages array)
- Metadata (timestamps, tags)

**What they DON'T capture:**
- Memory facts (separate system)
- File system state
- External service state

**Pattern: Task Progress Tracking**
```python
# Before starting complex task
checkpoint_save(
    agent_id="migration-agent",
    state={"task": "database migration", "step": 0, "tables_done": []}
)

# After each step
checkpoint_save(
    agent_id="migration-agent", 
    state={"task": "database migration", "step": 3, "tables_done": ["users", "orders"]}
)

# On session start, always check for existing checkpoint
cp = checkpoint_resume(agent_id="migration-agent")
if cp and cp["state"]["step"] < 5:
    continue_from_step(cp["state"]["step"])
```

**Pattern: Conversation Recovery**
```python
# Save before user goes away
checkpoint_save(
    agent_id="support-agent",
    state={"issue": "BUG-1234", "status": "awaiting_repro", "customer": "Acme Corp"}
)

# Resume next session
cp = checkpoint_resume(agent_id="support-agent")
# "Welcome back! We were investigating BUG-1234 for Acme Corp..."
```

---

## Memory Hygiene

### Fact Quality Checklist

Before storing, verify your fact meets these criteria:

- [ ] **Atomic** — One idea per fact, not a paragraph
- [ ] **Timeless** — Will still be true in 3 months
- [ ] **Actionable** — Helps future you make better decisions
- [ ] **Specific** — "prefers tabs" not "has preferences"
- [ ] **Self-contained** — Makes sense without conversation context

### Consolidation Awareness

Facts decay over time. A fact you never recall will eventually be pruned.

**Decay mechanics:**
- Weight starts at 1.0
- Decays exponentially based on time since last access
- Half-life: 30 days by default
- Pruned when weight < 0.01
- Accessing a fact resets its decay clock

**Implications for agents:**
```python
# BAD: Store once, never reference → will be pruned in ~60 days
memory_add(agent_id, "Critical security policy: ...")
# Then never search for it

# GOOD: Periodically recall important facts to keep them fresh
# Include them in your regular context gathering

# GOOD: Use shared memory for truly permanent facts
memory_add(agent_id="__shared__", text="Company security policy: ...")
```

### Memory Cleanup Schedule

Every 10-20 sessions, review and clean:

```python
# 1. List all memories (via CLI or inspection)
# graymatter recall <agent_id> "*"

# 2. Identify low-quality entries
# - Vague facts
# - Outdated workarounds  
# - Duplicate information

# 3. Clean up
memory_reflect(agent_id, action="forget", text="vague fact")
memory_reflect(agent_id, action="update", text="corrected fact")
```

---

## Session Continuity Patterns

### Pattern 1: Memory-First Boot

On every session start:

```python
# 1. Check for checkpoint (was I interrupted?)
cp = checkpoint_resume(agent_id="my-agent")
if cp:
    context = f"Resuming from checkpoint {cp['id']}. Last state: {cp['state']}"

# 2. Search for relevant memories about current task
memories = memory_search(
    agent_id="my-agent",
    query=current_task_description,
    top_k=8
)

# 3. Combine into system prompt
system_prompt = f"""{base_identity}

## Previous Session State
{context if cp else 'No previous checkpoint'}

## Relevant Past Memories
{chr(10).join(f'- {m}' for m in memories)}

## Instructions
Proceed with the task using the context above.
"""
```

### Pattern 2: Continuous Learning

During a session, continuously extract and store insights:

```python
def process_interaction(user_input, agent_response, task_context):
    # After significant interactions, extract learnings
    if is_significant_event(user_input, agent_response):
        # Extract atomic facts from the exchange
        learning = extract_learning(user_input, agent_response)
        if learning:
            memory_add(agent_id, text=learning)
    
    # Periodically consolidate (if async is off)
    if should_consolidate():
        trigger_consolidation(agent_id)
```

### Pattern 3: Multi-Agent Coordination

When multiple agents work on the same project:

```python
# Agent A discovers a convention
memory_add(agent_id="agent-a", text="Use async/await, not callbacks")

# Also store in shared memory so Agent B knows
memory_add(agent_id="__shared__", text="Project convention: Use async/await, not callbacks")

# Agent B recalls with merged view
facts = memory_search(agent_id="agent-b", query="async patterns")
# Gets agent-b specific + shared facts, deduplicated
```

**Shared memory best practices:**
- Store **project-wide** conventions, not agent-specific preferences
- Use clear prefixes: "Project convention:", "Team rule:", "All agents:"
- Keep shared memory small and high-signal (under 50 facts ideally)

---

## Advanced Patterns

### Fact Extraction from LLM Output

When an LLM produces a complex response with multiple insights:

```python
# Option 1: Manual extraction (most control)
response = call_llm(prompt)
facts = extract_atomic_facts(response)  # Your extraction logic
for fact in facts:
    memory_add(agent_id, text=fact)

# Option 2: Automatic extraction (GrayMatter handles it)
# Uses ANTHROPIC_API_KEY to extract facts via LLM
# Falls back to storing raw text if no API key
mem.RememberExtracted(ctx, agent_id, response)
```

**When to use automatic extraction:**
- Long LLM responses with scattered insights
- You don't want to manually parse
- You have ANTHROPIC_API_KEY set (uses Claude Haiku, cheap)

**When to use manual extraction:**
- You need precise control over what gets stored
- The response has a clear structure you can parse
- You want to avoid API costs

### Query Refinement

If initial search returns poor results:

```python
# Strategy 1: Rephrase the query
results = memory_search(agent_id, "database connection issues")
if not results:
    results = memory_search(agent_id, "postgres timeout errors")

# Strategy 2: Broader then narrower
broad = memory_search(agent_id, "backend problems", top_k=10)
narrow = memory_search(agent_id, "specific auth error", top_k=3)

# Strategy 3: Use entity names from knowledge graph
entities = extract_entities(current_task)
for entity in entities:
    results.extend(memory_search(agent_id, entity))
```

### Context Window Management

GrayMatter reduces token usage, but you still need to be smart about context:

```python
# BAD: Injecting all memories into every prompt
all_memories = memory_search(agent_id, "*", top_k=100)  # Too many!

# GOOD: Targeted recall for current task
relevant = memory_search(agent_id, current_task_description, top_k=8)

# GOOD: Two-tier approach
# Tier 1: Critical facts (always included)
critical = memory_search(agent_id, "critical preferences", top_k=3)

# Tier 2: Task-specific (included when relevant)
task_specific = memory_search(agent_id, current_task, top_k=5)

context = f"Critical context:\n{critical}\n\nTask context:\n{task_specific}"
```

---

## Anti-Patterns

### 1. The Dumping Ground

```python
# BAD: Storing everything
memory_add(agent_id, text="User said hello")
memory_add(agent_id, text="User asked about weather")
memory_add(agent_id, text="I responded with the forecast")
# Result: 1000 low-signal facts, important ones buried

# GOOD: Store conclusions only
memory_add(agent_id, text="User is planning outdoor event, needs weather updates")
```

### 2. The Self-Fulfilling Prophecy

```python
# BAD: Reinforcing incorrect assumptions
memory_add(agent_id, text="User likes X")
# Later, user changes preference
# But you keep recalling "User likes X" and acting on it

# GOOD: Update when preferences change
memory_reflect(agent_id, action="update", text="User now prefers Y (changed from X)")
```

### 3. The Orphaned Fact

```python
# BAD: Fact with no clear connection to anything
memory_add(agent_id, text="Blue")

# GOOD: Contextual fact
memory_add(agent_id, text="User's preferred UI theme: blue color scheme")
```

### 4. The Over-Specific Fact

```python
# BAD: Too specific to be reusable
memory_add(agent_id, text="On 2024-01-15 at 3:42pm, fixed bug in line 47 of auth.js")

# GOOD: Generalized learning
memory_add(agent_id, text="Auth.js: JWT validation fails when clock skew > 5 minutes")
```

### 5. The Memory Leak

```python
# BAD: Storing transient state as facts
memory_add(agent_id, text="Current file being edited: src/components/Button.tsx")
# This will persist forever but is only relevant right now

# GOOD: Use checkpoint for transient state
checkpoint_save(agent_id, state={"current_file": "src/components/Button.tsx"})
```

### 6. Ignoring Shared Memory

```python
# BAD: Every agent stores the same project conventions
memory_add(agent_id="agent-a", text="Use TypeScript")
memory_add(agent_id="agent-b", text="Use TypeScript")
memory_add(agent_id="agent-c", text="Use TypeScript")

# GOOD: Store once in shared namespace
memory_add(agent_id="__shared__", text="Project convention: Use TypeScript")
```

---

## Performance Considerations

### Token Budget

| Sessions | Full History | GrayMatter | Savings |
|----------|-------------|------------|---------|
| 1 | ~80 tokens | ~80 tokens | 0% |
| 10 | ~630 tokens | ~550 tokens | 12% |
| 30 | ~1,880 tokens | ~550 tokens | 71% |
| 100 | ~6,960 tokens | ~670 tokens | **90%** |

**Implication**: GrayMatter pays off after ~10 sessions. For short-lived agents (<5 sessions), the overhead may not be worth it.

### Latency

| Operation | Typical Latency | Notes |
|-----------|----------------|-------|
| `memory_add` | 5-20ms | bbolt write + optional vector upsert |
| `memory_search` | 10-50ms | Keyword + vector + RRF fusion |
| `checkpoint_save` | 5-15ms | Single bbolt transaction |
| `checkpoint_resume` | 5-10ms | Direct key lookup |

**Implication**: Safe to call multiple times per interaction. No need to batch.

### Storage Growth

```
Per fact: ~text_bytes + (embedding_dim × 4 bytes)
With nomic-embed-text (768 dim): ~3KB per fact
1000 facts: ~3MB text + embeddings
```

**Implication**: Even large memory stores are tiny. Don't worry about storage limits.

---

## Configuration Quick Reference

### Environment Variables

```bash
# Embedding providers (auto-detected in this order)
export OPENAI_API_KEY=sk-...        # OpenAI embeddings
export ANTHROPIC_API_KEY=sk-ant-... # Anthropic embeddings + consolidation
# Or run Ollama locally (default, recommended)
ollama pull nomic-embed-text

# Optional Ollama config
export GRAYMATTER_OLLAMA_URL=http://localhost:11434
export GRAYMATTER_OLLAMA_MODEL=nomic-embed-text
```

### Key Config Options

```go
graymatter.Config{
    DataDir:              ".graymatter",     // Storage path
    TopK:                 8,                  // Facts per recall
    EmbeddingMode:        graymatter.EmbeddingAuto,  // Auto-detect
    DecayHalfLife:        30 * 24 * time.Hour, // 30 days
    AsyncConsolidate:     true,               // Background cleanup
    ConsolidateThreshold: 20,                 // Trigger at 20 facts
    MaxAsyncConsolidations: 2,                // Max concurrent
}
```

**When to tune:**
- `TopK`: Increase to 12 if you have very dense memory; decrease to 5 if facts are highly specific
- `DecayHalfLife`: Decrease to 7 days for fast-changing domains; increase to 90 days for stable conventions
- `ConsolidateThreshold`: Lower to 10 for aggressive consolidation; raise to 50 for more retention

---

## Quick Decision Trees

### Should I Store This?

```
Is it a conclusion/decision/preference?
├── YES → Is it already in code/README?
│   ├── YES → Don't store (reference docs instead)
│   └── NO → Store it
└── NO → Is it a temporary state?
    ├── YES → Use checkpoint
    └── NO → Don't store
```

### Which Tool Should I Use?

```
Need to store a fact?
├── Have exact atomic fact → memory_add
├── Raw LLM response with multiple facts → RememberExtracted
├── Need to fix/update existing fact → memory_reflect update
├── Need to remove bad fact → memory_reflect forget
└── Need to link entities → memory_reflect link

Need to retrieve context?
├── Agent-specific only → memory_search (agent_id)
├── Shared only → memory_search (__shared__)
├── Both merged → memory_search (agent_id) + search __shared__
└── Need checkpoint → checkpoint_resume
```

### Session Start Checklist

```python
[ ] checkpoint_resume(agent_id) — was I interrupted?
[ ] memory_search(agent_id, current_task, top_k=8) — relevant memories
[ ] memory_search("__shared__", current_task, top_k=5) — shared context
[ ] Combine into system prompt
[ ] Proceed with task
```

### Session End Checklist

```python
[ ] Extract key learnings from session
[ ] memory_add(agent_id, learning) for each insight
[ ] Update any changed preferences via memory_reflect
[ ] checkpoint_save(agent_id, final_state) if task incomplete
[ ] Forget any temporary/transient facts
```

---

## Resources

- **GrayMatter GitHub**: https://github.com/angelnicolasc/graymatter
- **Go Docs**: https://pkg.go.dev/github.com/angelnicolasc/graymatter
- **MCP Docs**: https://docs.opencode.ai/graymatter
- **Latest Release**: https://github.com/angelnicolasc/graymatter/releases

---

*Remember: Good memory makes good agents. Store conclusions, not conversations.*
