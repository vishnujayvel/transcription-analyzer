# Mock Interview Analyzer

A structured prompt for analyzing mock interview transcripts across 10 comprehensive categories with confidence-scored, evidence-backed insights.

## Input Parameters

This analyzer receives from the supervisor:
- **transcript**: The full transcript content
- **subtype**: MockInterviewSubtype (SystemDesign, Coding, Behavioral, Unknown)
- **confidence**: Classification confidence from supervisor

---

## Interview Subtype Context

Adjust analysis emphasis based on subtype:

### SystemDesign
**Focus areas:**
- Architecture decisions and justifications
- Scalability trade-offs (consistency vs availability, etc.)
- Component design and interactions
- Data modeling and storage choices
- System boundaries and API contracts

**Weighted categories:** Technical Depth (1.5x), Structure (1.3x)

### Coding
**Focus areas:**
- Algorithm choice and complexity analysis
- Edge case handling
- Code quality signals (naming, structure)
- Optimization discussions
- Testing approach

**Weighted categories:** Technical Depth (1.5x), Factual Claims (1.3x)

### Behavioral
**Focus areas:**
- STAR format adherence
- Leadership and impact signals
- Communication clarity
- Storytelling effectiveness
- Conflict resolution approach

**Weighted categories:** Communication (1.5x), Behavioral Assessment (1.5x)

### Unknown
**Focus areas:** Apply balanced weighting across all categories

---

## Interview Start Detection

Scan for trigger phrases indicating interview start (skip small talk):

| Trigger Phrase | Context |
|----------------|---------|
| "go design" | System design prompt |
| "let's get started" | Formal start |
| "the problem is" | Coding problem |
| "design a system" | System design |
| "let's dive into" | Technical start |
| "first question" | Structure cue |
| "walk me through" | Technical prompt |

**Record:**
- Line number where interview starts
- If no trigger found: analyze from beginning, flag LOW confidence on timing

---

## Anti-Hallucination Protocol (MANDATORY)

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

## 10-Category Analysis Framework

### Category 1: Scorecard
Extract:
- **Overall performance** (1-10 scale) - Look for explicit feedback
- **Level assessment** (E5/E6/E7/Staff+) - Look for explicit statements or infer from complexity
- **Dimensions**: Communication, Technical Depth, Structure, Leadership (rate each 1-10)
- **Readiness %** = `100 - (P0_gaps × 15) - (P1_gaps × 5) - (CRITICAL_mistakes × 20) - (HIGH_mistakes × 10) - (MEDIUM_mistakes × 3)`

### Category 2: Time Breakdown
Extract:
- Total interview duration (if detectable)
- Phase timings: Requirements/Clarification, High-Level Design, Deep Dives, Q&A
- Time-related feedback (rushed, slow, good pacing)

### Category 3: Communication Signals
Extract:
- Talk ratio estimate (candidate vs interviewer %)
- Long pauses or hesitations noted
- Filler words frequency (um, uh, like, you know, basically)
- Clarifying questions asked (count)
- Course corrections after feedback (count)

### Category 4: Things That Went Well
**IMPORTANT: Output this BEFORE mistakes (ADHD-friendly ordering)**

Extract:
- Explicit praise ("good", "nice", "I like that", "exactly")
- Demonstrated strengths
- Approaches that worked
- For each: quote the evidence

### Category 5: Mistakes Identified
For EACH mistake:
- **Title**: Short description
- **Severity**: 🔴 CRITICAL (interview-ending), 🟠 HIGH (major), 🟡 MEDIUM (notable), ⚪ LOW (minor)
- **Category**: Fundamentals, API Design, Patterns, Domain Knowledge, Communication
- **Evidence**: Direct quote with line number

### Category 6: Knowledge Gaps
For EACH gap:
- **Area/Topic**: What was missing
- **Category**: Fundamentals, API Design, Patterns, Domain
- **Priority**: P0 (must fix), P1 (important), P2 (nice to have)
- **Evidence**: How gap was revealed

### Category 7: Behavioral Assessment (Staff+ Signals)
Extract:
- Leadership presence (drove the conversation vs followed prompts)
- Trade-off discussions (made and defended decisions?)
- Depth areas (which topics went deep?)
- Handling pushback (how did they respond to challenges?)

### Category 8: Factual Claims
For EACH technical claim made by the candidate:
- **The claim**: What they said
- **Classification**: ✅ Correct, ❌ Wrong, ❓ Needs Verification
- **Correction**: If wrong, what's the correct information (based on transcript evidence)

### Category 9: Action Items
Extract:
- Explicit recommendations from interviewer
- Resources recommended (books, sites, problems)
- Skills to improve

### Category 10: Interviewer Quality
Rate:
- Feedback actionability (1-5 scale)
- Specific examples given (count)
- Teaching moments (where interviewer explained concepts)

---

## JSON Output Schema

```json
{
  "metadata": {
    "sessionType": "MockInterview",
    "subtype": "SystemDesign" | "Coding" | "Behavioral" | "Unknown",
    "interviewStartLine": number | null,
    "totalLines": number,
    "analysisTimestamp": "ISO8601"
  },
  "scorecard": {
    "overall": { "score": 1-10, "confidence": "HIGH"|"MEDIUM"|"LOW"|"NOT_FOUND", "evidence": [] },
    "level": { "assessment": "E5"|"E6"|"E7"|"Staff+"|"Unknown", "confidence": "...", "evidence": [] },
    "dimensions": {
      "communication": { "score": 1-10, "confidence": "..." },
      "technicalDepth": { "score": 1-10, "confidence": "..." },
      "structure": { "score": 1-10, "confidence": "..." },
      "leadership": { "score": 1-10, "confidence": "..." }
    },
    "readiness": { "percentage": 0-100, "confidence": "...", "formula": "calculation details" }
  },
  "timeBreakdown": {
    "totalDuration": "string or null",
    "phases": [
      { "name": "Requirements", "duration": "...", "confidence": "..." }
    ],
    "feedback": []
  },
  "communicationSignals": {
    "talkRatio": { "candidate": 0-100, "interviewer": 0-100, "confidence": "..." },
    "fillerWords": { "count": number, "examples": [], "confidence": "..." },
    "clarifyingQuestions": { "count": number, "examples": [], "confidence": "..." },
    "courseCorrections": { "count": number, "examples": [], "confidence": "..." }
  },
  "positives": [
    {
      "title": "string",
      "description": "string",
      "evidence": { "type": "EXPLICIT"|"INFERRED", "content": "quote", "lineNumber": number },
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "mistakes": [
    {
      "title": "string",
      "description": "string",
      "severity": "CRITICAL"|"HIGH"|"MEDIUM"|"LOW",
      "category": "Fundamentals"|"API Design"|"Patterns"|"Domain Knowledge"|"Communication",
      "evidence": { "type": "EXPLICIT"|"INFERRED", "content": "quote", "lineNumber": number },
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "knowledgeGaps": [
    {
      "area": "string",
      "category": "Fundamentals"|"API Design"|"Patterns"|"Domain",
      "priority": "P0"|"P1"|"P2",
      "evidence": { "type": "...", "content": "...", "lineNumber": number },
      "confidence": "..."
    }
  ],
  "behavioralAssessment": {
    "leadership": { "score": 1-10, "evidence": [], "confidence": "..." },
    "tradeoffs": { "count": number, "examples": [], "confidence": "..." },
    "depthAreas": [],
    "pushbackHandling": { "assessment": "string", "evidence": [], "confidence": "..." }
  },
  "factualClaims": [
    {
      "claim": "string",
      "classification": "Correct"|"Wrong"|"NeedsVerification",
      "correction": "string or null",
      "confidence": "..."
    }
  ],
  "actionItems": [
    {
      "item": "string",
      "type": "explicit"|"implicit",
      "source": "interviewer"|"inferred",
      "priority": "high"|"medium"|"low"
    }
  ],
  "interviewerQuality": {
    "actionability": { "score": 1-5, "confidence": "..." },
    "specificExamples": { "count": number },
    "teachingMoments": []
  },
  "confidenceSummary": {
    "overall": "HIGH"|"MEDIUM"|"LOW",
    "overallScore": 0-100,
    "byCategory": {
      "scorecard": "...",
      "timeBreakdown": "...",
      "communicationSignals": "...",
      "positives": "...",
      "mistakes": "...",
      "knowledgeGaps": "...",
      "behavioralAssessment": "...",
      "factualClaims": "...",
      "actionItems": "...",
      "interviewerQuality": "..."
    },
    "dataQualityNotes": []
  }
}
```

---

## Markdown Output Format

```markdown
## Mock Interview Analysis

**File:** [filename]
**Session Type:** MockInterview.[subtype] [Confidence: X%]
**Interview Start:** Line [N] | [trigger phrase]
**Date Analyzed:** [timestamp]

---

### 1. Scorecard

| Metric | Score | Confidence | Evidence |
|--------|-------|------------|----------|
| Overall | X/10 | HIGH/MEDIUM/LOW | "quote" |
| Level | E6 | [INFERRED] | reason |
| Readiness | X% | MEDIUM | Based on gaps/mistakes |

**Dimensions:**
- Communication: X/10 [confidence]
- Technical Depth: X/10 [confidence]
- Structure: X/10 [confidence]
- Leadership: X/10 [confidence]

---

### 2. Time Breakdown

| Phase | Duration | Target | Status |
|-------|----------|--------|--------|
| Requirements | Xm | 5-10m | ✅/⚠️ |
| ... | ... | ... | ... |

---

### 3. Communication Signals

| Signal | Value | Confidence | Evidence |
|--------|-------|------------|----------|
| Talk Ratio | X% candidate | MEDIUM | [INFERRED] |
| Filler Words | X instances | HIGH | "um, uh, like" |
| ... | ... | ... | ... |

---

### 4. ⭐ Things That Went Well

| Positive | Evidence | Confidence |
|----------|----------|------------|
| [Title] | "quote" (line X) | HIGH |
| ... | ... | ... |

---

### 5. Mistakes Identified

#### 🔴 CRITICAL
| Mistake | Category | Evidence |
|---------|----------|----------|
| [Title] | [Category] | "quote" (line X) |

#### 🟠 HIGH
...

#### 🟡 MEDIUM
...

---

### 6. Knowledge Gaps

| Gap | Category | Priority | Evidence |
|-----|----------|----------|----------|
| [Area] | [Category] | P0/P1/P2 | "quote" |
| ... | ... | ... | ... |

---

### 7. Behavioral Assessment

| Signal | Assessment | Evidence | Confidence |
|--------|------------|----------|------------|
| Leadership | X/10 | "quote" | MEDIUM |
| Trade-offs | X made | examples | HIGH |
| ... | ... | ... | ... |

---

### 8. Factual Accuracy Check

| Claim | Classification | Correction |
|-------|----------------|------------|
| "claim" | ✅/❌/❓ | [if wrong] |
| ... | ... | ... |

---

### 9. Action Items

- [ ] 🔴 [High priority item] - [source]
- [ ] 🟡 [Medium priority item] - [source]
- [ ] ⚪ [Low priority item] - [source]

---

### 10. Interviewer Quality

| Metric | Score | Notes |
|--------|-------|-------|
| Actionability | X/5 | [notes] |
| Examples Given | X | [list] |
| Teaching Moments | X | [list] |

---

### Confidence Summary

| Category | Confidence |
|----------|------------|
| Overall | X% (HIGH/MEDIUM/LOW) |
| Scorecard | ... |
| Time Breakdown | ... |
| ... | ... |

**Data Quality Notes:**
- [Any caveats about the analysis]
```
