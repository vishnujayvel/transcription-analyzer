# Confidence Scoring Methodology

A rigorous protocol for preventing hallucination and ensuring evidence-backed analysis in LLM-generated insights.

## Why Confidence Scoring?

LLMs are prone to:
- **Hallucination**: Making up facts that sound plausible
- **False confidence**: Stating uncertain things with certainty
- **Filling gaps**: Inventing details when information is missing

This protocol forces explicit acknowledgment of uncertainty.

---

## Confidence Levels

| Level | Score Range | When to Use | Example |
|-------|-------------|-------------|---------|
| **HIGH** | 90-100% | Direct quote, explicit statement in source | Interviewer said "that's a 7 out of 10" |
| **MEDIUM** | 60-89% | Multiple supporting signals, reasonable inference | Several follow-up questions suggest confusion |
| **LOW** | 30-59% | Single weak signal, ambiguous evidence | Tone seemed uncertain (subjective) |
| **NOT_FOUND** | 0% | No evidence exists in the source | No time breakdown mentioned |

---

## Evidence Types

### [EXPLICIT] Evidence
Direct quotes or clear statements from the source material.

**Examples:**
- `[EXPLICIT] Interviewer: "You need to think about edge cases more"`
- `[EXPLICIT] "The overall score is 7/10"`

### [INFERRED] Evidence
Conclusions drawn from patterns, context, or indirect signals.

**Examples:**
- `[INFERRED] Multiple clarifying questions suggest requirement gathering was weak`
- `[INFERRED] Long pause before answering indicates uncertainty`

---

## Rules for Application

### Rule 1: No Fabrication
If information doesn't exist in the source, output:
```
**[Category]** [Confidence: NOT_FOUND]
- No evidence found in transcript
```

Do NOT:
- Make up plausible-sounding details
- Assume based on typical interview patterns
- Fill gaps with generic advice

### Rule 2: Cite Everything
Every claim needs a reference:

```markdown
| Claim | Confidence | Evidence |
|-------|------------|----------|
| Good API design | HIGH 92% | "I really liked how you structured the endpoints" (line 45) |
| Rushed through requirements | MEDIUM 65% | [INFERRED] Only 2 minutes spent on clarification |
| Weak on consistency | LOW 40% | Single follow-up question about CAP |
```

### Rule 3: Distinguish Inference from Fact
Always mark:
- `[EXPLICIT]` - Directly stated
- `[INFERRED]` - Concluded from evidence

### Rule 4: Aggregate Properly
Overall confidence = weighted average:

```
Category weights:
- Scorecard: 20%
- Mistakes: 20%
- Positives: 15%
- Gaps: 15%
- Time: 10%
- Communication: 10%
- Behavioral: 5%
- Factual: 5%

Overall = Σ(category_confidence × weight)
```

---

## Common Pitfalls

### Pitfall 1: Inflating Confidence
**Wrong:** "The candidate clearly struggled with system design" [HIGH]
**Right:** "The candidate received critical feedback on database choice" [HIGH] + "Follow-up questions suggest gaps in scaling knowledge" [MEDIUM, INFERRED]

### Pitfall 2: Ignoring Absence
**Wrong:** Skipping a category because no data
**Right:** "Time Breakdown [NOT_FOUND] - Transcript does not include timestamps or duration references"

### Pitfall 3: False Precision
**Wrong:** "Confidence: 73.4%"
**Right:** "Confidence: MEDIUM ~70%"

### Pitfall 4: Generic Statements
**Wrong:** "Communication could be improved"
**Right:** "Counted 15 filler words ('um', 'like') in first 10 minutes" [HIGH] + "Interviewer noted 'try to be more concise'" [EXPLICIT]

---

## Confidence Summary Template

Include at the end of every analysis:

```markdown
### Confidence Summary

| Category | Confidence | Notes |
|----------|------------|-------|
| Scorecard | HIGH 88% | Explicit score given |
| Time Breakdown | LOW 35% | No timestamps, estimated |
| Communication | MEDIUM 65% | Some explicit feedback |
| Positives | HIGH 90% | Multiple explicit praises |
| Mistakes | HIGH 85% | Direct corrections quoted |
| Knowledge Gaps | MEDIUM 70% | Inferred from questions |
| Behavioral | LOW 45% | Limited Staff+ discussion |
| Factual | MEDIUM 60% | Some claims unverifiable |
| Action Items | HIGH 92% | Explicit recommendations |
| Interviewer Quality | MEDIUM 55% | Subjective assessment |
| **Overall** | **MEDIUM 68%** | Weighted average |

**Data Quality Notes:**
- Transcript missing timestamps
- Some audio unclear sections noted
- Interviewer feedback was detailed
```

---

## Quick Reference

```
HIGH (90%+)     = "The transcript explicitly says..."
MEDIUM (60-89%) = "Based on multiple signals, it appears..."
LOW (30-59%)    = "There's weak evidence suggesting..."
NOT_FOUND (0%)  = "No evidence in transcript for..."

[EXPLICIT] = Direct quote or clear statement
[INFERRED] = Conclusion from patterns/context
```
