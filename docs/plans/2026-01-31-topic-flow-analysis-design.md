# Topic Flow Analysis - Design Document

**Date:** 2026-01-31
**Author:** Vishnu + Claude
**Status:** Ready for Implementation

---

## Overview

Add a new analysis mode `TopicFlowAnalysis` to the transcription-analyzer skill that:
1. Extracts hierarchical topics (themes → sub-topics) from long transcripts
2. Detects three types of deviations (unrelated jumps, rabbit holes, depth spirals)
3. Tracks who caused deviations and who brought the group back on topic
4. Analyzes filler words as confidence signals per topic
5. Generates Mermaid visualizations (Sankey + Timeline)

## Goals

- **Self-awareness**: See how often each person went off-topic vs. stayed focused
- **Group dynamics**: Identify who drives tangents vs. who refocuses
- **Content organization**: Generate structured summary from unstructured 3-hour conversations
- **Confidence mapping**: Correlate filler word density with topic uncertainty

---

## Architecture

### Three-Phase Map-Reduce

**Phase 1: Skeleton Extraction (Light Pass)**
- Extract timestamps, speaker changes, topic keywords
- Detect natural break points
- Model: Haiku (or regex preprocessing)
- Token cost: ~10% of full transcript

**Phase 2: Chunked Deep Analysis (Parallel)**
- Chunk size: 15-20 minutes with 2-minute overlap
- Parallel agents via Task tool
- Each chunk produces: local topic hierarchy, deviations, filler counts
- Model: Haiku

**Phase 3: Synthesis (Merge)**
- Merge chunk results into global topic tree
- Dedupe overlapping sections
- Cross-chunk pattern detection
- Generate visualizations
- Model: Sonnet

### Agent Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR (Main Agent)                     │
│  - Runs Phase 1 (skeleton extraction)                           │
│  - Spawns parallel chunk agents                                  │
│  - Merges results in Phase 3                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Chunk Agent  │   │  Chunk Agent  │   │  Chunk Agent  │
│   0:00-0:20   │   │   0:18-0:38   │   │   0:36-0:56   │
└───────────────┘   └───────────────┘   └───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │  Synthesis Agent  │
                    └───────────────────┘
```

---

## Speaker Detection (Layered)

1. **Inline detection first**: Patterns like "Vishnu:", "[Amy]", "**Victor:**", "Vishnu [0:05]:"
2. **Header metadata**: Look for "Participants:" in frontmatter or header
3. **Interactive fallback**: Ask user "Who participated in this conversation?"

---

## Topic Hierarchy

### Two-Level Structure

```
Theme (Major Topic)
├── Sub-topic 1 [time: Xmin, confidence: Y%]
├── Sub-topic 2 [time: Xmin, confidence: Y%]
└── Sub-topic 3 [time: Xmin, confidence: Y%]
```

### Detection

1. **Boundary signals** (regex): "let's move on to", "next question", "okay so", "anyway"
2. **Theme extraction** (LLM per chunk): Main theme + sub-topics + key terms
3. **Cross-chunk merging**: Fuzzy match themes → cluster → canonical name

---

## Deviation Classification

### Three Types

| Type | Definition | Detection |
|------|------------|-----------|
| **Unrelated Jump** | Topic A → B with no logical connection | Cosine similarity < 0.3 |
| **Rabbit Hole** | Deep exploration that doesn't return | Duration > threshold + no return |
| **Depth Spiral** | Over-indexing on one sub-topic | Time > 2x average |

### Severity

- 🔴 **High**: Unrelated jump + never returned, or spiral > 3x average
- 🟠 **Medium**: Rabbit hole > 10min, or spiral 2-3x average
- 🟡 **Low**: Brief tangent < 5min with clean return
- ⚪ **Intentional**: Explicit marker like "let's take a quick detour"

### Return Attribution

Track who brought the conversation back:
```json
{
  "returned_by": "Amy",
  "return_phrase": "Anyway, going back to Kafka...",
  "return_timestamp": "1:23:45"
}
```

---

## Filler Word Analysis

### Categories

| Category | Patterns | Signal |
|----------|----------|--------|
| Classic fillers | um, uh, er, ah, hmm | Processing |
| Hedge words | like, you know, basically, essentially, kind of, sort of | Uncertainty |
| Stalling phrases | "so basically what I'm trying to say", "let me think" | Formulating |
| Confidence killers | "I'm not sure but", "I think maybe" | Low confidence |

### Per-Topic Confidence Score

```
filler_density = total_fillers / total_words
confidence = max(0, 100 - (density * 1000) - weighted_penalty)
```

### Output

- Topic confidence heatmap (green = confident, red = uncertain)
- Per-speaker filler profile
- Correlation: which topics had most uncertainty

---

## Visualizations

### 1. Mermaid Sankey (Topic Flow)

```mermaid
sankey-beta
Intro,Writes at Scale,5
Writes at Scale,Hot Keys,15
Hot Keys,Counter Sharding,10
Counter Sharding,Tangent: Lunch,2
Tangent: Lunch,Counter Sharding,2
```

### 2. Mermaid Timeline (Chronological)

```mermaid
timeline
    title Topic Flow
    section Hour 1
        0:00-0:15 : Hot Keys 🟢
        0:15-0:25 : Counter Sharding 🟢
        0:25-0:27 : ⚠️ Tangent
```

---

## Output Format

```markdown
## Topic Flow Analysis

**File:** {filename}
**Duration:** Xh Ym
**Participants:** {list}
**Main Theme:** {detected or provided}

---

### 1. Topic Hierarchy
{tree with time and confidence}

### 2. Topic Flow (Sankey)
{mermaid diagram}

### 3. Timeline View
{mermaid diagram}

### 4. Deviation Report
{table with type, from→to, duration, returned_by}

### 5. Filler Word Analysis
{topic confidence heatmap}
{per-speaker profile}

### 6. Insights & Recommendations
{LLM-generated actionable insights}

---

### JSON Export
{structured data}
```

---

## File Structure

```
transcription-analyzer/
├── SKILL.md                           # Add TopicFlowAnalysis mode
├── prompts/
│   ├── topic_flow_orchestrator.md     # NEW: Main orchestration
│   ├── topic_extractor.md             # NEW: Hierarchical topic detection
│   ├── deviation_classifier.md        # NEW: Tangent detection
│   ├── filler_word_analyzer.md        # NEW: Confidence signals
│   └── topic_flow_synthesizer.md      # NEW: Merge + visualize
└── examples/
    └── sample_topic_flow_output.md    # NEW: Example output
```

---

## Test Data

- `/Users/vishnu/Documents/obsibrain/obsibrain-vault/5-notes/Database party with Amy and Victor - transcription.md`
- 1072 lines, ~3 hours
- Speakers: Vishnu, Amy (marked as "Them"), Victor

---

## Implementation Order

1. Add TopicFlowAnalysis to SKILL.md session types
2. Create topic_flow_orchestrator.md prompt
3. Create topic_extractor.md prompt
4. Create deviation_classifier.md prompt
5. Create filler_word_analyzer.md prompt
6. Create topic_flow_synthesizer.md prompt
7. Add chunking logic to orchestrator
8. Test with Database Party transcript
9. Generate sample output
