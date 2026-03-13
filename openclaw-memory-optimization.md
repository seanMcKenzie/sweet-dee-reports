# OpenClaw Memory Optimization: Comprehensive Report

**Author:** Sweet Dee (Research Agent)  
**Date:** 2026-03-13  
**Scope:** Memory architecture, config tuning, QMD backend, multi-agent recommendations

---

## Table of Contents

1. [Overview: OpenClaw Memory Architecture](#1-overview-openclaw-memory-architecture)
2. [Memory Layers Deep Dive](#2-memory-layers-deep-dive)
3. [Config Tuning Options](#3-config-tuning-options)
4. [QMD Backend (Experimental)](#4-qmd-backend-experimental)
5. [Recommendations: 7-Agent Discord + Spring Boot Setup](#5-recommendations-7-agent-discord--spring-boot-setup)
6. [Quick-Reference Config Template](#6-quick-reference-config-template)

---

## 1. Overview: OpenClaw Memory Architecture

OpenClaw's memory system is intentionally simple at its foundation: **plain Markdown files on disk are the source of truth**. The model only "remembers" what has been written to disk. There's no magic in-process brain — just files and indexes.

Memory search tooling is provided by the active **memory plugin** (default: `memory-core`). You can disable it entirely with:

```json5
plugins: {
  slots: {
    memory: "none"
  }
}
```

The system has three functional layers:

| Layer | What it is | When it's used |
|---|---|---|
| **Daily logs** | `memory/YYYY-MM-DD.md` | Rolling operational context |
| **MEMORY.md** | Curated long-term notes | Main/private sessions only |
| **Vector search index** | SQLite + embeddings | Semantic recall across all memory files |

Each agent has its own isolated memory index at `~/.openclaw/memory/<agentId>.sqlite`.

---

## 2. Memory Layers Deep Dive

### 2.1 Daily Logs — `memory/YYYY-MM-DD.md`

- **Append-only** rolling log of what happened each day.
- OpenClaw reads **today's + yesterday's** file at session start.
- Lives under `agents.defaults.workspace` (default: `~/.openclaw/workspace`).
- Best for: task status, decisions made that day, running context, things that will age out.

### 2.2 `MEMORY.md` — Curated Long-Term Memory

- Optional but highly recommended for personal agents.
- **Security-sensitive**: should only be loaded in the main private session, **never** in group/channel contexts.
- Best for: preferences, durable facts, recurring patterns, lessons learned.
- The agent can read, edit, and update it freely in main sessions.

### 2.3 Vector Search — Semantic Recall

OpenClaw builds a per-agent SQLite vector index over all `MEMORY.md` and `memory/*.md` files. Two agent-facing tools are exposed:

- **`memory_search`** — semantic query over indexed chunks (~400 tokens each, 80-token overlap). Returns snippets (~700 chars max) with file path, line range, and relevance score.
- **`memory_get`** — targeted read of a specific memory file by path + optional line range.

**`memory_get` graceful degradation:** If a file doesn't exist yet (e.g., today's log before the first write), both the built-in manager and QMD backend return `{ text: "", path }` instead of throwing `ENOENT`. Agents don't need try/catch for this.

#### Auto-Select Embedding Provider

If `memorySearch.provider` is not set, OpenClaw auto-selects in this order:
1. `local` — if `memorySearch.local.modelPath` is configured and the file exists
2. `openai` — if an OpenAI key is available
3. `gemini` — if a Gemini key is available
4. `voyage` — if a Voyage key is available
5. `mistral` — if a Mistral key is available
6. Otherwise, memory search stays **disabled** until configured

#### Hybrid Search: BM25 + Vector

When enabled, OpenClaw combines two retrieval signals:

- **Vector similarity** — great for paraphrases: "Mac Studio gateway host" ≈ "the machine running the gateway"
- **BM25 keyword relevance** — great for exact tokens: commit hashes, env var names, error strings

The merge algorithm:
1. Collect a candidate pool from both sides (`maxResults × candidateMultiplier`)
2. Convert BM25 rank to a 0–1 score: `textScore = 1 / (1 + max(0, bm25Rank))`
3. Weighted merge: `finalScore = vectorWeight × vectorScore + textWeight × textScore`

**Configuration:**
```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4
        }
      }
    }
  }
}
```

#### Post-Processing Pipeline

After hybrid merge, two optional stages refine results:

```
Vector + Keyword → Weighted Merge → Temporal Decay → Sort → MMR → Top-K Results
```

**MMR (Maximal Marginal Relevance) — Diversity re-ranking:**
- Iteratively selects results that balance relevance with diversity
- Formula: `λ × relevance − (1−λ) × max_similarity_to_selected`
- Default `lambda: 0.7` (slight relevance bias)
- **Enable when:** daily notes produce near-duplicate snippets
- Both features are off by default

**Temporal Decay — Recency boost:**
- Applies exponential decay: `decayedScore = score × e^(-λ × ageInDays)`
- Default half-life: 30 days (score halves every 30 days)
- Evergreen files (`MEMORY.md`, non-dated files) are **never** decayed
- **Enable when:** months of daily notes cause stale info to outrank recent updates

---

## 3. Config Tuning Options

### 3.1 `agents.defaults.memorySearch`

The main knob for controlling memory indexing and retrieval behavior.

```json5
agents: {
  defaults: {
    memorySearch: {
      // Provider selection
      provider: "openai",          // or "gemini", "voyage", "mistral", "local", "ollama"
      model: "text-embedding-3-small",
      fallback: "none",            // fallback provider if primary fails

      // Remote API config
      remote: {
        apiKey: "...",
        baseUrl: "https://api.example.com/v1/",  // optional custom endpoint
        batch: {
          enabled: true,           // batch mode for large corpus (OpenAI/Gemini/Voyage)
          concurrency: 2
        }
      },

      // Local embedding mode
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf"
      },

      // Hybrid search
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4,
          mmr: {
            enabled: true,
            lambda: 0.7
          },
          temporalDecay: {
            enabled: true,
            halfLifeDays: 30
          }
        }
      },

      // Embedding cache (avoids re-embedding unchanged text)
      cache: {
        enabled: true,
        maxEntries: 50000
      },

      // Extra paths to index beyond default workspace
      extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"],

      // Session transcript indexing (experimental)
      experimental: { sessionMemory: false },
      sources: ["memory"],         // add "sessions" to include transcripts

      // File watcher + index freshness
      sync: {
        watch: true,
        sessions: {
          deltaBytes: 100000,
          deltaMessages: 50
        }
      },

      // SQLite vector acceleration
      store: {
        vector: {
          enabled: true,
          extensionPath: ""        // override bundled sqlite-vec path if needed
        }
      }
    }
  }
}
```

**Key tradeoffs:**
- `local` provider: fully offline, no API costs, but requires ~0.6 GB GGUF model and `node-llama-cpp` native build
- `openai` batch mode: fastest for large corpus backfills, ~50% cheaper than sync via OpenAI Batch API
- `cache.enabled`: strongly recommended — prevents re-embedding unchanged chunks on every sync

### 3.2 `compaction.memoryFlush`

Controls the **silent pre-compaction memory flush** — a background agentic turn that fires when the session is near the context limit, prompting the agent to write durable notes before history gets summarized away.

```json5
agents: {
  defaults: {
    compaction: {
      reserveTokensFloor: 20000,   // tokens to keep reserved for the response
      memoryFlush: {
        enabled: true,
        softThresholdTokens: 4000, // flush triggers when contextWindow - reserveTokensFloor - softThresholdTokens is crossed
        systemPrompt: "Session nearing compaction. Store durable memories now.",
        prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store.",
      }
    }
  }
}
```

**Important notes:**
- The flush is **silent** by default — `NO_REPLY` is the expected response
- Only fires **once per compaction cycle** (tracked in `sessions.json`)
- **Skipped** if `workspaceAccess` is `"ro"` or `"none"`
- You can point to a **dedicated model** for compaction summarization: `compaction.model: "ollama/llama3.1:8b"` (useful to save costs on summaries)
- `identifierPolicy: "strict"` (default) preserves opaque IDs (commit hashes, UUIDs) in summaries — don't change this unless you know what you're doing

### 3.3 `contextPruning`

Session pruning trims **old tool results** from the in-memory context **before each LLM call** — without rewriting the on-disk JSONL session history. It complements (but is separate from) compaction.

```json5
agents: {
  defaults: {
    contextPruning: {
      mode: "cache-ttl",           // "off" | "cache-ttl"
      ttl: "5m",                   // prune if last Anthropic call was older than this
      keepLastAssistants: 3,       // protect this many recent assistant turns
      softTrimRatio: 0.3,
      hardClearRatio: 0.5,
      minPrunableToolChars: 50000,
      softTrim: {
        maxChars: 4000,
        headChars: 1500,
        tailChars: 1500
      },
      hardClear: {
        enabled: true,
        placeholder: "[Old tool result content cleared]"
      },
      // Restrict to specific tools
      tools: {
        allow: ["exec", "read"],
        deny: ["*image*"]
      }
    }
  }
}
```

**Pruning vs Compaction:**
| | Compaction | Context Pruning |
|---|---|---|
| **Scope** | Entire conversation history | Tool results only |
| **Persistence** | Writes to JSONL | In-memory, per-request only |
| **When** | Near context window limit | Cache TTL expiry |
| **Effect** | Summarizes → keeps summary | Trims/clears large tool outputs |

**Cost impact:** `cache-ttl` mode reduces `cacheWrite` size on the first post-TTL request, avoiding full re-caching of bloated tool output history. The TTL resets after each prune so follow-up requests can reuse the freshly cached prompt.

---

## 4. QMD Backend (Experimental)

### 4.1 What is QMD?

QMD ([github.com/tobi/qmd](https://github.com/tobi/qmd)) is a **local-first search sidecar** that replaces OpenClaw's default SQLite memory indexer. It combines:

- **BM25** — keyword-based full-text search
- **Dense vectors** — semantic similarity via local GGUF embeddings
- **Reranking** — local neural reranker to re-score the merged candidate list

Markdown files remain the **source of truth** — QMD is purely a retrieval layer. OpenClaw shells out to the `qmd` binary for all searches.

### 4.2 How It Works

**Sidecar lifecycle:**
1. OpenClaw writes a self-contained QMD home at `~/.openclaw/agents/<agentId>/qmd/` (sets `XDG_CONFIG_HOME` and `XDG_CACHE_HOME`)
2. On gateway startup, collections are created via `qmd collection add` from `memory.qmd.paths` plus default workspace memory files
3. `qmd update` + `qmd embed` run at boot and on a configurable interval (default: 5 minutes)
4. Boot refresh runs **in the background** by default — chat startup is not blocked
5. Searches run via `qmd search --json` (default), `vsearch`, or `query` mode
6. If QMD fails or the binary is missing, OpenClaw **automatically falls back** to the built-in SQLite manager — no memory tool downtime

**Search flow:**
```
memory_search call
      ↓
QMD sidecar (BM25 + dense vectors + reranking)
      ↓ (if QMD fails or binary missing)
Automatic fallback to built-in SQLite indexer
      ↓
Snippets returned to agent
```

**First search caveat:** QMD may auto-download GGUF reranker/query-expansion models from HuggingFace on the first `qmd query` run. This can be slow. Pre-warm manually:

```bash
STATE_DIR="${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
export XDG_CONFIG_HOME="$STATE_DIR/agents/main/qmd/xdg-config"
export XDG_CACHE_HOME="$STATE_DIR/agents/main/qmd/xdg-cache"
qmd update
qmd embed
qmd query "test" -c memory-root --json >/dev/null 2>&1
```

### 4.3 Prerequisites

| Requirement | Notes |
|---|---|
| **Bun** | Required for QMD runtime |
| **QMD CLI** | `bun install -g https://github.com/tobi/qmd` or grab a release binary |
| **SQLite with extensions** | `brew install sqlite` on macOS (system SQLite doesn't allow extensions) |
| **`qmd` on PATH** | Must be accessible from the gateway process |
| **OS** | macOS + Linux: supported. Windows: WSL2 recommended |
| **Disk space** | GGUF models auto-download to `XDG_CACHE_HOME` — budget a few hundred MB |

### 4.4 Configuration Surface

Enable with `memory.backend = "qmd"`:

```json5
memory: {
  backend: "qmd",
  citations: "auto",
  qmd: {
    command: "qmd",                  // override executable path
    searchMode: "search",            // "search" | "vsearch" | "query"
    includeDefaultMemory: true,      // auto-index MEMORY.md + memory/**/*.md
    paths: [
      { name: "docs", path: "~/notes", pattern: "**/*.md" }
    ],

    // Update cadence
    update: {
      interval: "5m",
      debounceMs: 15000,
      onBoot: true,
      waitForBootSync: false,        // set true for blocking boot behavior
      embedInterval: "30m",
      commandTimeoutMs: 10000,
      updateTimeoutMs: 30000,
      embedTimeoutMs: 60000
    },

    // Result limits
    limits: {
      maxResults: 6,
      maxSnippetChars: 700,
      maxInjectedChars: 4000,
      timeoutMs: 4000
    },

    // Scope control — critical for multi-agent/channel setups
    scope: {
      default: "deny",
      rules: [
        { action: "allow", match: { chatType: "direct" } },
        { action: "deny", match: { keyPrefix: "discord:channel:" } },
        { action: "deny", match: { rawKeyPrefix: "agent:main:discord:" } }
      ]
    },

    // Session transcript indexing
    sessions: {
      enabled: false,
      retentionDays: 30,
      exportDir: ""
    }
  }
}
```

**Citations:** `memory.citations = "auto"` appends `Source: <path#line>` to snippets. Set to `"off"` to keep path metadata internal.

### 4.5 Risks and Tradeoffs vs Default SQLite Indexer

| Dimension | Default SQLite Indexer | QMD Backend |
|---|---|---|
| **Setup complexity** | Zero — works out of the box | Requires Bun + custom SQLite + QMD CLI install |
| **Cold start** | Fast | Potentially slow (GGUF model downloads on first use) |
| **Recall quality** | Good (hybrid BM25 + vector in SQLite) | Better (dedicated reranking step improves precision) |
| **Local/offline** | Depends on embedding provider | Fully local after model download |
| **Failure mode** | No fallback needed | Falls back to SQLite indexer automatically |
| **Disk footprint** | Lightweight (SQLite DB only) | Heavier (SQLite + GGUF models in cache) |
| **Update latency** | ~1.5s debounce | 5min interval + debounce (configurable) |
| **Windows support** | Full | WSL2 only |
| **Production stability** | Stable, shipped default | Experimental — may change |
| **Multi-agent isolation** | Per-agent SQLite files | Per-agent XDG directories |
| **Channel scope control** | N/A (builtin always runs) | Fine-grained `scope` rules per channel/chatType |

**When to use QMD:**
- You want the best possible retrieval quality and are willing to accept setup overhead
- Your agents accumulate large memory corpora (hundreds of daily notes + MEMORY.md)
- You need fully offline/local memory search without external API calls
- You want to surface older session transcripts via `sessions.enabled: true`

**When to stick with the default:**
- You want zero-config simplicity
- You're running on Windows without WSL2
- You need guaranteed fast cold starts (e.g., latency-sensitive Discord bots)
- You're still early in the project (memory corpus is small)

---

## 5. Recommendations: 7-Agent Discord + Spring Boot Setup

Our setup: **7 agents** (K2S0, Charlie, Dennis, Mac, Frank, Sweet Dee, Cricket), all Discord-channel based, building a Spring Boot project.

### 5.1 Memory Isolation

Each agent should have **isolated memory** — don't share a workspace between agents. The default per-agent SQLite index (`~/.openclaw/memory/<agentId>.sqlite`) handles this correctly out of the box.

**For shared project knowledge** (e.g., architecture decisions, API contracts), use a **shared extraPath** that all agents read but only designated agents write:

```json5
agents: {
  defaults: {
    memorySearch: {
      extraPaths: ["~/shared-project-docs"]
    }
  }
}
```

### 5.2 Don't Load MEMORY.md in Discord Channels

This is a security hard rule. `MEMORY.md` contains private user context. For all 7 agents operating in Discord channels, ensure their workspace setup only loads daily logs at session start — not MEMORY.md. Check your AGENTS.md per agent to confirm this is respected.

### 5.3 Enable Hybrid Search + Temporal Decay

As the project accumulates weeks of daily logs, semantic-only search will start surfacing stale context (e.g., old Spring Boot decisions that were overridden). Enable temporal decay with a 30-day half-life:

```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4,
          mmr: {
            enabled: true,
            lambda: 0.7
          },
          temporalDecay: {
            enabled: true,
            halfLifeDays: 30
          }
        }
      }
    }
  }
}
```

BM25 is especially valuable here — Spring Boot code symbols, class names, and API endpoint strings (`/api/v1/reps`, `MedicalRepService`) are exactly the kind of exact-token queries that vector search struggles with.

### 5.4 Enable Embedding Cache

With 7 agents all running on the same machine and potentially sharing overlapping memory content, the embedding cache prevents redundant API calls:

```json5
agents: {
  defaults: {
    memorySearch: {
      cache: {
        enabled: true,
        maxEntries: 50000
      }
    }
  }
}
```

### 5.5 Configure memoryFlush

With 7 agents in Discord channels, sessions can get long. Ensure the pre-compaction memory flush is enabled so important decisions survive compaction:

```json5
agents: {
  defaults: {
    compaction: {
      reserveTokensFloor: 20000,
      memoryFlush: {
        enabled: true,
        softThresholdTokens: 4000,
        systemPrompt: "Session nearing compaction. Store durable memories now.",
        prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
      }
    }
  }
}
```

### 5.6 Enable contextPruning for Long Discord Sessions

Discord-based agents accumulate `exec` and `read` tool results fast. `cache-ttl` pruning keeps the context lean:

```json5
agents: {
  defaults: {
    contextPruning: {
      mode: "cache-ttl",
      ttl: "5m",
      keepLastAssistants: 3,
      tools: {
        deny: ["*image*"]
      }
    }
  }
}
```

### 5.7 QMD for This Setup: Hold Off for Now

**Recommendation: Don't use QMD yet.** Here's why:

- The project is active development — memory corpus is still growing, not yet at the scale where QMD's reranking provides a clear win
- 7 agents × QMD sidecar = 7 QMD processes + 7 model download caches. That's non-trivial on a single Mac
- The automatic SQLite fallback means QMD failures are survivable, but debugging 7 parallel QMD processes in Discord threads is painful
- The default hybrid BM25 + vector search in SQLite is already quite good for this use case

**Revisit QMD when:**
- Memory corpus exceeds ~200 daily log files per agent
- You're getting stale/irrelevant `memory_search` results despite temporal decay
- You want fully local embeddings (no OpenAI API calls) — QMD's local GGUF path is compelling here

### 5.8 Recommended Provider for Embeddings

Given the Spring Boot project context (code symbols, class names, stack traces):

1. **`openai/text-embedding-3-small`** — best balance of quality and cost, great at code symbols with BM25 supplementing
2. **`local`** — if you want zero API costs; requires `pnpm approve-builds` + ~0.6 GB model download

Enable batch mode if doing large initial indexing:

```json5
memorySearch: {
  provider: "openai",
  model: "text-embedding-3-small",
  remote: {
    batch: { enabled: true, concurrency: 2 }
  }
}
```

### 5.9 Session Transcript Indexing: Optional Enhancement

For K2S0 (coordinator), enabling session transcript indexing lets it recall conversations with team members without relying solely on daily logs:

```json5
agents: {
  k2s0: {
    memorySearch: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"]
    }
  }
}
```

Leave this **off** for other agents unless they need cross-session recall. Index isolation keeps each agent focused.

---

## 6. Quick-Reference Config Template

A production-ready configuration for our multi-agent Discord setup:

```json5
{
  agents: {
    defaults: {
      // Memory search
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        fallback: "none",
        cache: { enabled: true, maxEntries: 50000 },
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            candidateMultiplier: 4,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 }
          }
        },
        sync: { watch: true },
        store: { vector: { enabled: true } }
      },

      // Pre-compaction memory flush
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      },

      // Context pruning for Discord sessions
      contextPruning: {
        mode: "cache-ttl",
        ttl: "5m",
        keepLastAssistants: 3,
        hardClear: { enabled: true },
        tools: { deny: ["*image*"] }
      }
    }
  }
}
```

---

## Summary

OpenClaw's memory system is powerful precisely because it's so simple — files on disk, not a black-box database. The optimization levers are:

1. **Write things down** — the system can only recall what's written. Encourage agents to use memory tools actively.
2. **Enable hybrid search** — BM25 + vectors is materially better than either alone, especially for codebases.
3. **Temporal decay + MMR** — essential for long-running agents with months of daily notes.
4. **memoryFlush** — prevents knowledge loss at compaction boundaries.
5. **contextPruning** — keeps Discord sessions lean without losing history.
6. **QMD** — compelling for large corpora and local-only setups, but adds operational complexity. Use when you've outgrown the SQLite indexer.

---

*Report compiled from official OpenClaw documentation: https://docs.openclaw.ai/concepts/memory*  
*Research conducted by Sweet Dee, Research Agent — March 13, 2026*
