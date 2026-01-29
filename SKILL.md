---
name: transcription-analyzer
description: >
  Analyzes mock interview transcripts using multi-agent architecture with 4 parallel
  analyst agents (Strengths, Mistakes, Behavioral, Factual) to produce confidence-scored
  insights across 10 categories. Features anti-hallucination protocol requiring evidence
  citation for every claim. Use when reviewing mock interviews, wanting interview feedback
  analysis, or saying "analyze my transcript" or "mock review".
license: MIT
metadata:
  author: vishnu-jayavel
  version: "1.0"
  categories: interview-prep, analysis, multi-agent
---

# Transcription Analyzer

Analyze mock interview transcripts with comprehensive, confidence-scored analytics across 10 categories using multi-agent architecture.

## Triggers
- "analyze my transcript"
- "transcription-analyzer"
- "mock review"
- "review my transcript"

---

## Anti-Hallucination Protocol (MANDATORY)

**Every metric and insight MUST include confidence scoring and evidence citation.**

### Confidence Levels

| Level | Score | Criteria |
|-------|-------|----------|
| **HIGH** | 90%+ | Direct quote from transcript, explicit statement |
| **MEDIUM** | 60-89% | Inferred from context, multiple supporting signals |
| **LOW** | 30-59% | Single weak signal, ambiguous evidence |
| **NOT_FOUND** | 0% | No evidence in transcript - explicitly state this |

### Rules (Non-Negotiable)
1. **Never fabricate** - If not in transcript, output "Not found in transcript"
2. **Cite evidence** - Every claim needs line number or direct quote
3. **Distinguish inference from fact** - Mark clearly: `[INFERRED]` vs `[EXPLICIT]`
4. **Aggregate confidence** - Overall score = weighted average of components

See [references/confidence-scoring.md](references/confidence-scoring.md) for detailed methodology.

---

## Step 1: Input Handling

### If ARGUMENTS provided:
Use the provided file path directly.

### If NO ARGUMENTS:
Ask user for the transcript file path.

---

## Step 2: File Validation

1. Load the transcript file
2. Validate the file exists and contains content
3. Count total lines for delegation decision

### Error Handling:

**If file not found:**
```
Could not find transcript at: [attempted_path]
Please check the file path is correct.
```

**If file is empty:**
```
The transcript file appears to be empty.
Please provide a transcript with interview content.
```

---

## Step 3: Interview Start Detection

Scan the transcript for trigger phrases that indicate when the actual interview begins (skip small talk):

| Trigger Phrase | Context |
|----------------|---------|
| "go design" | System design prompt |
| "let's get started" | Formal interview start |
| "the problem is" | Coding problem introduction |
| "design a system" | System design prompt |
| "let's dive into" | Technical start |
| "first question" | Interview structure cue |
| "walk me through" | Technical prompt |

**Record:**
- Line number where interview starts
- If no trigger found: analyze from beginning, flag LOW confidence on timing

---

## Step 4: Interview Type Detection

Classify the interview type based on content signals:

### System Design Signals
- "design a system", "scalability", "database schema"
- "high availability", "load balancer", "microservices"
- "CAP theorem", "partitioning", "replication"

### Coding Signals
- "write a function", "time complexity", "space complexity"
- "test cases", "edge cases", "optimal solution"
- "brute force", "algorithm", "data structure"

### Behavioral Signals
- "tell me about a time", "leadership", "conflict"
- "difficult situation", "disagree with", "mentor"
- "STAR format", "situation", "action", "result"

**Output:**
- Interview type with confidence level and evidence
- If unclear: "Unknown" with NOT_FOUND confidence

---

## Step 5: Optional Diagram Analysis (System Design Only)

**IF interview type is "System Design":**

Ask user if they have an architecture diagram to analyze alongside the transcript.

**IF diagram provided:**
Analyze:
- Components identified (services, databases, caches, queues)
- Data flow clarity (request paths, async flows)
- Missing components vs. verbal description
- Naming quality
- Diagram Quality Score (1-10)

**IF no diagram:**
Note that no diagram was provided and recommend saving diagrams for future review.

---

## Step 6: Multi-Perspective Agent Analysis

**IMPORTANT: Use parallel agents for comprehensive, bias-reduced analysis.**

Launch **4 parallel agents** to analyze from different perspectives. This prevents single-viewpoint blind spots.

### Agent 1: Strengths Analyst
Find everything positive:
- Explicit praise from interviewer ("good", "nice", "I like that")
- Demonstrated competencies
- Strong moments and recoveries
- Communication wins

Output: `{"positives": [{"title": "", "evidence": "", "confidence": "", "category": ""}]}`

### Agent 2: Mistakes Analyst
Find errors and problems:
- Technical errors corrected by interviewer
- Conceptual misunderstandings
- Communication issues (filler words, long pauses)
- Missed opportunities

Severity levels: CRITICAL (interview-ending), HIGH, MEDIUM, LOW

Output: `{"mistakes": [{"title": "", "severity": "", "evidence": "", "confidence": "", "category": ""}]}`

### Agent 3: Behavioral Analyst
Assess Staff+ signals:
- Leadership presence (drove vs followed conversation)
- Trade-off articulation (made decisions, defended them)
- Depth of technical discussion
- Response to pushback/challenges
- Communication maturity

Output: `{"behavioral": {"leadership": {...}, "tradeoffs": {...}, "depth_areas": [], "pushback_handling": {...}}}`

### Agent 4: Factual Verifier
Check technical accuracy:
- CORRECT: Technically accurate
- WRONG: Incorrect (cite the correction from transcript)
- NEEDS_VERIFICATION: Cannot determine from transcript alone

Only mark WRONG if interviewer explicitly corrected it.

Output: `{"claims": [{"claim": "", "classification": "", "correction": "", "confidence": ""}]}`

### Synthesis

After all 4 agents return, cross-validate:
- If Strengths Agent found a positive but Mistakes Agent found related error → note the recovery
- If Behavioral Agent found leadership but Factual Agent found errors → assess net impact
- Resolve conflicts by citing evidence from both perspectives

---

## Step 7: 10-Category Analysis

For each category, extract insights with confidence scoring and evidence citation.

### Category 1: Scorecard
- **Overall performance** (1-10 scale) - Look for explicit feedback
- **Level assessment** (Junior/Mid/Senior/Staff+) - Look for explicit statements or infer
- **Dimensions**: Communication, Technical Depth, Structure, Leadership
- **Readiness %** = `100 - (P0_gaps × 15) - (P1_gaps × 5) - (CRITICAL_mistakes × 20) - (HIGH_mistakes × 10) - (MEDIUM_mistakes × 3)`

### Category 2: Time Breakdown
- Total interview duration
- Phase timings: Requirements, High-Level Design, Deep Dives, Q&A
- Time-related feedback from interviewer

### Category 3: Communication Signals
- Talk ratio (candidate vs interviewer)
- Long pauses, filler words (um, uh, like, you know, basically)
- Clarifying questions asked
- Course corrections after feedback

### Category 4: Mistakes Identified
For EACH mistake:
- Title and description
- Severity: CRITICAL, HIGH, MEDIUM, LOW
- Category: Fundamentals, API Design, Patterns, Domain Knowledge, Communication
- Direct evidence with line number

### Category 5: Things That Went Well
- Explicit praise
- Demonstrated strengths
- Approaches that worked

### Category 6: Knowledge Gaps
For EACH gap:
- Area/topic
- Category: Fundamentals, API Design, Patterns, Domain
- Priority: P0 (must fix), P1 (important), P2 (nice to have)

### Category 7: Behavioral Assessment (Staff+ Signals)
- Leadership presence
- Trade-off discussions
- Depth areas
- Handling pushback

### Category 8: Factual Claims
For EACH technical claim:
- The claim
- Classification: Correct, Wrong, Needs Verification
- Correction if wrong

### Category 9: Action Items
- Explicit recommendations from interviewer
- Resources recommended

### Category 10: Interviewer Quality
- Feedback actionability (1-5 scale)
- Specific examples given (count)
- Teaching moments

---

## Step 8: Output Formatting

**IMPORTANT: Show positives BEFORE mistakes (motivation-friendly ordering)**

Structure the report as:
1. Metadata (file, type, confidence)
2. Scorecard
3. Time Breakdown
4. Communication Signals
5. Things That Went Well (before mistakes!)
6. Mistakes Identified
7. Knowledge Gaps
8. Behavioral Assessment
9. Factual Accuracy Check
10. Action Items
11. Interviewer Quality
12. Confidence Summary

Include tables with evidence citations and confidence levels for each item.

---

## Step 9: JSON Summary

After the markdown report, output a structured JSON summary with all categories for programmatic consumption.

---

## Assets

- [assets/sample_transcript.md](assets/sample_transcript.md) - Example transcript for testing
- [assets/sample_output.md](assets/sample_output.md) - Example analysis output

## References

- [references/analyzer-prompt.md](references/analyzer-prompt.md) - Portable prompt for any LLM
- [references/confidence-scoring.md](references/confidence-scoring.md) - Confidence methodology
