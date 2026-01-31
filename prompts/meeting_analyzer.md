# Generic Meeting Analyzer

Extract summary, decisions, and action items from any meeting or conversation with confidence-scored, evidence-backed insights.

## Input Parameters

This analyzer receives from the supervisor:
- **transcript**: The full transcript content
- **confidence**: Classification confidence from supervisor

---

## Anti-Hallucination Protocol (MANDATORY)

**Every insight MUST include confidence scoring and evidence citation.**

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

---

## 6-Category Extraction Framework

### Category 1: Meeting Context

**Extract:**
- **Participants**: Who is in the meeting (names if identifiable, descriptions otherwise)
- **Purpose/Topic**: What the meeting is about
- **Confidence**: How clearly this is established

**Signals for participant identification:**
- Explicit greetings ("Hi [name]", "Thanks [name]")
- Speaker labels in transcript
- References to each other
- Meeting opener introductions

**Signals for purpose identification:**
- Meeting title mentioned
- "We're here to discuss..."
- "The agenda is..."
- Context from early discussion

### Category 2: Summary

**Create:**
- **Executive Summary**: 3-5 sentences capturing the essence of the meeting
- **Key Points**: Bullet list of most important topics discussed

**Summarization guidelines:**
- Focus on outcomes, not process
- Highlight what changed or was decided
- Note significant disagreements or debates
- Capture the "so what" of the meeting

### Category 3: Decisions Made

**Extract ALL explicit decisions:**

For EACH decision:
- **Decision**: What was decided
- **Owner**: Who is responsible (if mentioned)
- **Context**: Why this decision was made
- **Evidence**: Direct quote showing the decision
- **Confidence**: How explicit the decision was

**Signals for decisions:**
- "We decided to..."
- "Let's go with..."
- "The decision is..."
- "We agreed that..."
- "I'll approve..."
- Consensus statements

### Category 4: Action Items

**Extract ALL tasks assigned:**

For EACH action item:
- **Action**: What needs to be done
- **Owner**: Who is responsible (null if not specified)
- **Deadline**: When it's due (null if not specified)
- **Evidence**: Quote showing the assignment
- **Confidence**: How clearly this was stated

**Signals for action items:**
- "[Name] will..."
- "Action item: ..."
- "Can you [do X]?"
- "We need to..."
- "Follow up on..."
- "By [date], we need..."

### Category 5: Open Questions

**Track unresolved topics:**

For EACH open question:
- **Question**: What remains unresolved
- **Context**: Why this matters
- **Needs Follow-up**: Whether this requires future discussion
- **Confidence**: How clearly identified

**Look for:**
- Questions without clear answers
- "We need to figure out..."
- "TBD"
- "Let's revisit..."
- Topics deferred to future meetings
- Parking lot items

### Category 6: Key Quotes

**Capture memorable or important statements:**

For EACH key quote:
- **Quote**: The exact words
- **Speaker**: Who said it (if identifiable)
- **Line Number**: Where in transcript
- **Significance**: Why this quote matters

**Look for:**
- Strong opinions or positions
- Strategic statements
- Commitments made
- Surprising or unexpected statements
- Quotable summaries of complex topics

---

## JSON Output Schema

```json
{
  "metadata": {
    "sessionType": "GenericMeeting",
    "totalLines": number,
    "analysisTimestamp": "ISO8601"
  },
  "meetingContext": {
    "participants": ["name1", "name2"],
    "purpose": "string",
    "confidence": "HIGH"|"MEDIUM"|"LOW"|"NOT_FOUND"
  },
  "summary": {
    "executiveSummary": "3-5 sentence summary",
    "keyPoints": ["point1", "point2", "point3"],
    "confidence": "HIGH"|"MEDIUM"|"LOW"
  },
  "decisions": [
    {
      "decision": "string",
      "owner": "string" | null,
      "context": "string",
      "evidence": { "type": "EXPLICIT"|"INFERRED", "content": "quote", "lineNumber": number },
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "actionItems": [
    {
      "action": "string",
      "owner": "string" | null,
      "deadline": "string" | null,
      "evidence": { "type": "EXPLICIT"|"INFERRED", "content": "quote", "lineNumber": number },
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "openQuestions": [
    {
      "question": "string",
      "context": "string",
      "needsFollowUp": true | false,
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "keyQuotes": [
    {
      "quote": "string",
      "speaker": "string" | null,
      "lineNumber": number,
      "significance": "string"
    }
  ],
  "confidenceSummary": {
    "overall": "HIGH"|"MEDIUM"|"LOW",
    "overallScore": 0-100,
    "byCategory": {
      "meetingContext": "...",
      "summary": "...",
      "decisions": "...",
      "actionItems": "...",
      "openQuestions": "...",
      "keyQuotes": "..."
    },
    "dataQualityNotes": []
  }
}
```

---

## Markdown Output Format

```markdown
## Meeting Analysis

**File:** [filename]
**Session Type:** GenericMeeting [Confidence: X%]
**Participants:** [name1], [name2], [name3]
**Purpose:** [detected purpose]

---

### 1. Executive Summary

[3-5 sentence summary capturing the essence of the meeting]

**Key Points:**
- [Key point 1]
- [Key point 2]
- [Key point 3]

---

### 2. Key Decisions

| Decision | Owner | Evidence | Confidence |
|----------|-------|----------|------------|
| [decision] | [owner or —] | "[quote]" (line X) | HIGH |
| [decision] | [owner or —] | "[quote]" (line X) | MEDIUM |

---

### 3. Action Items

| Action | Owner | Deadline | Evidence |
|--------|-------|----------|----------|
| [action] | [owner or TBD] | [date or —] | "[quote]" (line X) |
| [action] | [owner or TBD] | [date or —] | "[quote]" (line X) |

**Checklist format:**
- [ ] [Action] - Owner: [name], Due: [date]
- [ ] [Action] - Owner: [name], Due: [date]

---

### 4. Open Questions

| Question | Context | Follow-up Needed |
|----------|---------|------------------|
| [question] | [context] | ✅ Yes / ❌ No |
| [question] | [context] | ✅ Yes / ❌ No |

---

### 5. Key Quotes

> "[Quote]"
> — [Speaker], line [X]
> *Significance: [Why this matters]*

> "[Quote]"
> — [Speaker], line [X]
> *Significance: [Why this matters]*

---

### Confidence Summary

| Category | Confidence |
|----------|------------|
| Overall | X% (HIGH/MEDIUM/LOW) |
| Meeting Context | ... |
| Summary | ... |
| Decisions | ... |
| Action Items | ... |
| Open Questions | ... |
| Key Quotes | ... |

**Data Quality Notes:**
- [Any caveats about the analysis]
```
