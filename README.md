# Transcription Analyzer

**Comprehensive mock interview transcript analysis with confidence-scored, evidence-backed insights across 10 categories.**

Built with an anti-hallucination protocol to ensure every insight is backed by evidence from your transcript.

## Features

- **10 Analytics Categories**: Scorecard, Time Breakdown, Communication, Mistakes, Positives, Gaps, Behavioral, Factual Claims, Action Items, Interviewer Quality
- **Anti-Hallucination Protocol**: Every metric includes confidence scoring (HIGH/MEDIUM/LOW/NOT_FOUND) and evidence citation
- **Works Anywhere**: Use with ChatGPT, Claude, Gemini, or any LLM
- **Structured Output**: Markdown report with consistent formatting

## Quick Start

### Option 1: Copy-Paste (Works with ANY LLM)

1. Open [prompts/analyzer.md](prompts/analyzer.md)
2. Copy the entire content
3. Paste your transcript at the end (where it says `[PASTE YOUR TRANSCRIPT BELOW]`)
4. Send to ChatGPT, Claude.ai, Gemini, or any LLM

### Option 2: Claude Code Skill

If you use [Claude Code](https://claude.ai/code), install as a skill:

```bash
# Copy the adapter to your Claude Code skills directory
cp -r adapters/claude-code ~/.claude/skills/transcription-analyzer
```

Then invoke with:
```
/transcription-analyzer
```
Or say: "analyze my mock interview transcript"

## How It Works

### The 10-Category Framework

| Category | What It Analyzes |
|----------|------------------|
| **1. Scorecard** | Overall performance (1-10), level assessment, readiness % |
| **2. Time Breakdown** | Phase durations, pacing feedback |
| **3. Communication** | Talk ratio, filler words, clarifying questions |
| **4. Mistakes** | Errors categorized by severity (CRITICAL → LOW) |
| **5. Positives** | What went well, explicit praise received |
| **6. Knowledge Gaps** | Missing knowledge areas prioritized (P0/P1/P2) |
| **7. Behavioral** | Staff+ signals: leadership, trade-offs, depth |
| **8. Factual Claims** | Technical accuracy verification |
| **9. Action Items** | Recommendations and next steps |
| **10. Interviewer Quality** | Feedback actionability and teaching moments |

### Anti-Hallucination Protocol

Every insight includes:

- **Confidence Level**: HIGH (90%+), MEDIUM (60-89%), LOW (30-59%), NOT_FOUND (0%)
- **Evidence Type**: `[EXPLICIT]` (direct quote) or `[INFERRED]` (concluded from patterns)
- **Source Citation**: Line numbers or direct quotes

**Example output:**
```markdown
| Metric | Score | Confidence | Evidence |
|--------|-------|------------|----------|
| Overall | 7/10 | HIGH 95% | "I'd say this is solid E6 level" (line 194) |
| Database Choice | Issue | HIGH 88% | [EXPLICIT] "you initially jumped to PostgreSQL" (line 186) |
| Pacing | Good | MEDIUM 65% | [INFERRED] Covered all topics within time |
```

### Why This Matters

LLMs tend to:
- Make up plausible-sounding details
- State uncertain things with false confidence
- Fill gaps with generic advice

This protocol forces the LLM to:
- Cite evidence for every claim
- Acknowledge when information is missing
- Distinguish facts from inferences

## File Structure

```
transcription-analyzer/
├── README.md                    # This file
├── LICENSE                      # MIT License
├── prompts/
│   ├── analyzer.md              # Main analysis prompt (portable)
│   └── confidence-scoring.md    # Confidence methodology reference
├── examples/
│   └── sample_transcript.md     # Example transcript for testing
└── adapters/
    └── claude-code/
        └── SKILL.md             # Claude Code skill integration
```

## Sample Output

When analyzing the included sample transcript (a URL shortener system design interview):

```markdown
## Mock Interview Analysis

**Interview Type:** System Design [Confidence: HIGH 95%]
**Overall Confidence:** MEDIUM 72%

### 1. Scorecard
| Metric | Score | Confidence | Evidence |
|--------|-------|------------|----------|
| Overall | 7/10 | HIGH 90% | "solid E6 level performance" (line 194) |
| Level | E6 | HIGH 92% | [EXPLICIT] Direct statement |
| Readiness | 78% | MEDIUM 70% | Based on 1 HIGH mistake, 2 P1 gaps |

### 4. Things That Went Well
1. **Back-of-envelope calculations** [HIGH 95%]
   - Evidence: "your back-of-envelope calculations were excellent" (line 184)

2. **Self-correction ability** [HIGH 90%]
   - Evidence: "I liked how you corrected yourself on the PostgreSQL point" (line 185)

### 5. Mistakes Identified
#### 🟠 HIGH
1. **Conflated consistent hashing with database partitioning** [HIGH 88%]
   - Category: Fundamentals
   - Evidence: "consistent hashing for the database, that's typically for caches" (line 191)

### 6. Knowledge Gaps
| Gap | Priority | Confidence | Evidence |
|-----|----------|------------|----------|
| Database partitioning vs consistent hashing | P1 | HIGH | Explicit feedback (line 191) |
| Cache-aside vs write-through patterns | P2 | HIGH | Recommended study area (line 197) |
```

## Contributing

Contributions welcome! Areas for improvement:

- [ ] Additional LLM adapters (Cursor, Aider, etc.)
- [ ] Support for coding interview transcripts
- [ ] Behavioral interview specific prompts
- [ ] Automated transcript preprocessing

## License

MIT License - see [LICENSE](LICENSE)

## Acknowledgments

Built with the philosophy that **LLM insights should be verifiable, not just plausible**.

Inspired by best practices in structured output generation and confidence calibration.
