# Transcription Analyzer

Open-source Claude Code skill for analyzing mock interview transcripts with comprehensive, confidence-scored analytics across 10 categories.

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

---

## Step 1: Input Handling

### If ARGUMENTS provided:
Use the provided file path directly.

### If NO ARGUMENTS:
Use AskUserQuestion to request the transcript file path:

```
question: "What is the path to your mock interview transcript file?"
header: "File Path"
options:
  - label: "Browse for file"
    description: "I'll provide the full file path"
```

---

## Step 2: File Validation

1. Use the Read tool to load the transcript file
2. Validate the file exists and contains content
3. Count total lines for delegation decision

### Error Handling:

**If file not found:**
```markdown
❌ **File Not Found**

Could not find transcript at: `[attempted_path]`

Please check:
- The file path is correct
- The file extension is .md or .txt
- The file has read permissions
```

**If file is empty:**
```markdown
❌ **Empty File**

The transcript file appears to be empty: `[path]`

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

Ask user if they have an architecture diagram:
```
question: "Do you have an architecture diagram from this mock interview?"
header: "Diagram"
options:
  - label: "Yes, I'll share it"
    description: "I have an Excalidraw, image, or diagram file"
  - label: "No diagram available"
    description: "Skip diagram analysis"
```

**IF diagram provided:**
Analyze:
- Components identified (services, databases, caches, queues)
- Data flow clarity (request paths, async flows)
- Missing components vs. verbal description
- Naming quality
- Diagram Quality Score (1-10)

**IF no diagram:**
```markdown
### Diagram Analysis [Confidence: NOT_FOUND]
- No diagram provided for analysis
- Recommendation: Save diagrams for future review
```

---

## Step 6: Delegation Decision

**Check transcript line count:**

### IF transcript > 500 lines:
Delegate to Explore subagent using the Task tool:

```
Use Task tool with:
- subagent_type: "Explore"
- prompt: [Load prompts/analyzer_subagent.md content + transcript]
- description: "Analyze mock interview transcript"
```

### IF transcript ≤ 500 lines:
Perform direct analysis using the 10-category framework below.

---

## Step 7: 10-Category Analysis

For each category, extract insights with confidence scoring and evidence citation.

### Category 1: Scorecard
- **Overall performance** (1-10 scale) - Look for explicit feedback
- **Level assessment** (E5/E6/E7/Staff+) - Look for "this is E6 level" or infer from complexity
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
- Severity: CRITICAL (interview-ending), HIGH (major), MEDIUM (notable), LOW (minor)
- Category: Fundamentals, API Design, Patterns, Domain Knowledge, Communication
- Direct evidence with line number

### Category 5: Things That Went Well
- Explicit praise ("good", "nice", "I like that")
- Demonstrated strengths
- Approaches that worked

### Category 6: Knowledge Gaps
For EACH gap:
- Area/topic
- Category: Fundamentals, API Design, Patterns, Domain
- Priority: P0 (must fix), P1 (important), P2 (nice to have)

### Category 7: Behavioral Assessment (Staff+ Signals)
- Leadership presence (drove conversation?)
- Trade-off discussions (made and defended decisions?)
- Depth areas (which topics went deep?)
- Handling pushback (response to challenges?)

### Category 8: Factual Claims
For EACH technical claim:
- The claim
- Classification: Correct, Wrong, Needs Verification
- Correction if wrong (from transcript evidence)

### Category 9: Action Items
- Explicit recommendations from interviewer
- Resources recommended (books, sites, problems)

### Category 10: Interviewer Quality
- Feedback actionability (1-5 scale)
- Specific examples given (count)
- Teaching moments (where interviewer explained concepts)

---

## Step 8: Output Formatting

### Markdown Report Structure

**IMPORTANT: Show positives BEFORE mistakes (ADHD-friendly ordering)**

```markdown
## Mock Interview Analysis

**File:** [filename]
**Interview Type:** [type] [Confidence: X]
**Interview Start:** Line [N] (trigger: "[phrase]")
**Date Analyzed:** [timestamp]
**Overall Confidence:** [weighted average]

---

### 1. Scorecard

| Metric | Score | Confidence | Evidence |
|--------|-------|------------|----------|
| **Overall** | X/10 | HIGH 92% | "[quote]" (line N) |
| **Level Assessment** | E6 | MEDIUM 70% | [INFERRED] from discussion depth |
| **Readiness** | X% | MEDIUM | Based on gaps and mistakes |

**Dimensional Scores:**
| Dimension | Score | Confidence |
|-----------|-------|------------|
| Communication | X/10 | HIGH |
| Technical Depth | X/10 | MEDIUM |
| Structure | X/10 | HIGH |
| Leadership | X/10 | LOW |

---

### 2. Time Breakdown

| Phase | Duration | Target | Status | Confidence |
|-------|----------|--------|--------|------------|
| Requirements | X min | 5-7 min | ✅ On track | MEDIUM |
| High-Level Design | X min | 10-15 min | ⚠️ Over | MEDIUM |
| Deep Dives | X min | 15-20 min | ✅ On track | MEDIUM |

---

### 3. Communication Signals

| Signal | Value | Confidence | Evidence |
|--------|-------|------------|----------|
| Talk Ratio | Candidate X% / Interviewer Y% | MEDIUM | [INFERRED] |
| Filler Words | X frequency | MEDIUM | Counted instances |
| Clarifying Questions | N asked | HIGH | Direct count |
| Course Corrections | N made | HIGH | Explicit pivots |

---

### 4. Things That Went Well ⭐

1. **[Strength]** [Confidence: HIGH 90%]
   - Evidence: "[Positive feedback]" (line N)
   - Category: Keep doing

2. **[Positive Signal]** [Confidence: MEDIUM 70%]
   - Evidence: Interviewer said "good" after explanation
   - Category: Strength to leverage

---

### 5. Mistakes Identified

#### 🔴 CRITICAL
*None identified* OR list...

#### 🟠 HIGH

1. **[Mistake Title]** [Confidence: HIGH 88%]
   - **Category:** [type]
   - **Evidence:** "[quote]" (line N)

#### 🟡 MEDIUM

1. **[Mistake Title]** [Confidence: MEDIUM 65%]
   - **Category:** [type]
   - **Evidence:** [INFERRED] from follow-up question

#### ⚪ LOW
*None identified* OR list...

---

### 6. Knowledge Gaps

| Gap | Category | Priority | Confidence | Evidence |
|-----|----------|----------|------------|----------|
| [Gap 1] | Fundamentals | P0 | HIGH | "[quote]" |
| [Gap 2] | Patterns | P1 | MEDIUM | [INFERRED] |

**Priority Legend:**
- **P0:** Must fix before real interview
- **P1:** Important to address
- **P2:** Nice to have

---

### 7. Behavioral Assessment (Staff+ Signals)

| Signal | Assessment | Confidence | Evidence |
|--------|------------|------------|----------|
| Leadership Presence | Yes/No/Partial | X | [evidence] |
| Decisive Trade-offs | N made | X | [examples] |
| Depth in 3+ Areas | Yes/No | X | [areas listed] |
| Handling Pushback | Good/Needs Work | X | [example] |

---

### 8. Factual Accuracy Check

| Claim | Classification | Confidence | Correction |
|-------|---------------|------------|------------|
| "[Technical claim 1]" | ✅ Correct | HIGH | - |
| "[Technical claim 2]" | ❌ Wrong | HIGH | [Correct info] |
| "[Technical claim 3]" | ❓ Needs Verification | MEDIUM | - |

---

### 9. Action Items

- [ ] **P0:** [Action from P0 gaps]
- [ ] **P0:** [Action from CRITICAL/HIGH mistakes]
- [ ] [Explicit recommendation from interviewer]
- [ ] Review [resource] recommended by interviewer

---

### 10. Interviewer Quality

| Metric | Value | Confidence |
|--------|-------|------------|
| Feedback Actionability | X/5 | HIGH |
| Specific Examples Given | N | HIGH |
| Teaching Moments | N | MEDIUM |

---

### Confidence Summary

| Category | Confidence |
|----------|------------|
| Scorecard | X |
| Time Breakdown | X |
| Communication | X |
| Positives | X |
| Mistakes | X |
| Knowledge Gaps | X |
| Behavioral | X |
| Factual Accuracy | X |
| Action Items | X |
| Interviewer Quality | X |
| **Overall** | **X** |

**Data Quality Notes:**
- [Any caveats about transcript quality]
- [Any sections with NOT_FOUND results]
```

---

## Step 9: JSON Summary

After the markdown report, output a JSON summary:

```json
{
  "metadata": {
    "file": "[filename]",
    "interviewType": "[type]",
    "interviewStartLine": N,
    "totalLines": N,
    "analysisTimestamp": "[ISO timestamp]"
  },
  "scorecard": {
    "overall": {"score": X, "confidence": "HIGH|MEDIUM|LOW", "evidence": "..."},
    "level": {"assessment": "E5|E6|E7|Staff+|Unknown", "confidence": "...", "evidence": "..."},
    "dimensions": {...},
    "readiness": {"percentage": X, "confidence": "...", "formula": "..."}
  },
  "timeBreakdown": {...},
  "communicationSignals": {...},
  "positives": [...],
  "mistakes": [...],
  "knowledgeGaps": [...],
  "behavioralAssessment": {...},
  "factualClaims": [...],
  "interviewerQuality": {...},
  "actionItems": [...],
  "confidenceSummary": {
    "overall": "HIGH|MEDIUM|LOW",
    "overallScore": X,
    "byCategory": {...},
    "dataQualityNotes": [...]
  }
}
```

---

## Portability Notes

This skill is designed to be **portable and dependency-free**:
- Works with any transcript file path
- No MCP server dependencies
- No cross-session memory or note-taking service dependencies
- Uses only: Read, AskUserQuestion, Task tools

**Allowed Tools:**
- `Read` - File reading
- `AskUserQuestion` - User interaction
- `Task` - Subagent delegation (Explore type only)
