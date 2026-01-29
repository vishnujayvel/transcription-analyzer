# Mock Interview Transcript Analyzer

A structured prompt for analyzing mock interview transcripts across 10 comprehensive categories with confidence-scored, evidence-backed insights.

## How to Use

**With any LLM (ChatGPT, Claude, Gemini, etc.):**

1. Copy this entire prompt
2. Paste your transcript at the end
3. Send to the LLM

---

## Instructions

You are analyzing a mock interview transcript. Follow these rules strictly:

### Anti-Hallucination Protocol (MANDATORY)

**Every metric and insight MUST include confidence scoring and evidence citation.**

| Confidence Level | Score | Criteria |
|-----------------|-------|----------|
| **HIGH** | 90%+ | Direct quote from transcript, explicit statement |
| **MEDIUM** | 60-89% | Inferred from context, multiple supporting signals |
| **LOW** | 30-59% | Single weak signal, ambiguous evidence |
| **NOT_FOUND** | 0% | No evidence in transcript - explicitly state this |

**Rules:**
1. **Never fabricate** - If not in transcript, output "Not found in transcript"
2. **Cite evidence** - Every claim needs a direct quote or line reference
3. **Distinguish inference from fact** - Mark clearly: `[INFERRED]` vs `[EXPLICIT]`
4. **Aggregate confidence** - Overall score = weighted average of components

---

## Analysis Categories

### 1. Scorecard
Extract:
- **Overall performance** (1-10 scale) - Look for explicit feedback
- **Level assessment** (Junior/Mid/Senior/Staff+) - Look for explicit statements or infer from complexity
- **Dimensions**: Communication, Technical Depth, Structure, Leadership (rate each 1-10)
- **Readiness %** = `100 - (Critical_gaps × 15) - (Major_gaps × 5) - (Critical_mistakes × 20) - (Major_mistakes × 10)`

### 2. Time Breakdown
Extract:
- Total interview duration
- Phase timings: Requirements/Clarification, High-Level Design, Deep Dives, Q&A
- Time-related feedback (rushed, slow, good pacing)

### 3. Communication Signals
Extract:
- Talk ratio estimate (candidate vs interviewer %)
- Long pauses or hesitations noted
- Filler words frequency (um, uh, like, you know, basically)
- Clarifying questions asked (count)
- Course corrections after feedback (count)

### 4. Mistakes Identified
For EACH mistake, extract:
- **Title**: Short description
- **Severity**: CRITICAL (interview-ending), HIGH (major), MEDIUM (notable), LOW (minor)
- **Category**: Fundamentals, API Design, Patterns, Domain Knowledge, Communication
- **Evidence**: Direct quote with context

### 5. Things That Went Well
Extract:
- Explicit praise ("good", "nice", "I like that", "exactly")
- Demonstrated strengths
- Approaches that worked
- For each: quote the evidence

### 6. Knowledge Gaps
For EACH gap:
- **Area/Topic**: What was missing
- **Category**: Fundamentals, API Design, Patterns, Domain
- **Priority**: P0 (must fix before real interview), P1 (important), P2 (nice to have)

### 7. Behavioral Assessment (Staff+ Signals)
Extract:
- Leadership presence (drove the conversation vs followed prompts)
- Trade-off discussions (made and defended decisions?)
- Depth areas (which topics went deep?)
- Handling pushback (how did they respond to challenges?)

### 8. Factual Claims
For EACH technical claim made by the candidate:
- **The claim**: What they said
- **Classification**: Correct, Wrong, or Needs Verification
- **Correction**: If wrong, what's the correct information (based on transcript evidence)

### 9. Action Items
Extract:
- Explicit recommendations from interviewer
- Resources recommended (books, sites, problems)
- Skills to improve

### 10. Interviewer Quality
Rate:
- Feedback actionability (1-5 scale)
- Specific examples given (count)
- Teaching moments (where interviewer explained concepts)

---

## Output Format

Structure your response as follows:

```markdown
## Mock Interview Analysis

**Interview Type:** [System Design / Coding / Behavioral / Mixed] [Confidence: X]
**Overall Confidence:** [weighted average across categories]

---

### 1. Scorecard

| Metric | Score | Confidence | Evidence |
|--------|-------|------------|----------|
| Overall | X/10 | X | "quote" |
| Level | X | X | [INFERRED/EXPLICIT] reason |
| Readiness | X% | X | Based on gaps/mistakes |

**Dimensions:** Communication X/10, Technical X/10, Structure X/10, Leadership X/10

---

### 2. Time Breakdown
[Table with phases, durations, targets, status]

---

### 3. Communication Signals
[Table with signal, value, confidence, evidence]

---

### 4. Things That Went Well
[List positives FIRST - this ordering matters for motivation]

---

### 5. Mistakes Identified
[Grouped by severity: CRITICAL, HIGH, MEDIUM, LOW]

---

### 6. Knowledge Gaps
[Table with gap, category, priority, evidence]

---

### 7. Behavioral Assessment
[Table with Staff+ signals and evidence]

---

### 8. Factual Accuracy Check
[Table with claims, classification, corrections]

---

### 9. Action Items
[Prioritized checklist]

---

### 10. Interviewer Quality
[Ratings with evidence]

---

### Confidence Summary
[Table showing confidence per category + overall]
```

---

## Transcript to Analyze

[PASTE YOUR TRANSCRIPT BELOW THIS LINE]
