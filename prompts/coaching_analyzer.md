# Coaching Session Analyzer

Extract actionable advice, scripts, and patterns from coaching/mentoring sessions with confidence-scored, evidence-backed insights.

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

### Category 1: Session Context

**Extract:**
- **Coach/Mentor identification**: Look for who is giving advice, using phrases like "I'd recommend", "you should", "here's what I suggest"
- **Mentee identification**: Look for who is receiving advice, asking questions, seeking feedback
- **Session topic(s)**: What subjects are being discussed

**Signals for coach identification:**
- Uses imperative voice ("do this", "try that")
- Shares personal experience as examples
- Asks probing questions to understand mentee's situation
- Uses phrases: "in my experience", "I've seen", "what works is"

**Signals for mentee identification:**
- Asks questions seeking advice
- Describes their situation or challenge
- Uses phrases: "I'm struggling with", "what should I", "how do I"

### Category 2: Key Advice/Tips

**Extract ALL explicit recommendations from the coach:**

For EACH piece of advice:
- **Advice**: The recommendation (paraphrased if needed)
- **Domain**: communication, technical, career, behavioral, other
- **Quote**: Direct quote from transcript
- **Line number**: Where in transcript
- **Confidence**: Based on how explicit the advice is

**Domain categories:**
- **communication**: How to communicate, present, write, speak
- **technical**: Technical skills, tools, processes
- **career**: Career growth, job search, promotions
- **behavioral**: How to behave, act, respond in situations
- **other**: Anything not fitting above

### Category 3: Scripts/Patterns

**Extract reusable phrases or frameworks taught:**

For EACH script/pattern:
- **Name**: Short descriptive name (e.g., "3-Minute Check-In Script")
- **Script**: The actual quotable text - format as ready-to-use
- **Context**: When to use this script/pattern
- **Line number**: Where in transcript
- **Confidence**: How clearly the script was articulated

**Look for:**
- "Here's what you say..."
- "Try this approach..."
- "The framework is..."
- "When X happens, do Y..."
- Step-by-step instructions
- Templates or formats shared

### Category 4: Action Items

**Extract tasks the mentee should do:**

For EACH action item:
- **Action**: What needs to be done
- **Type**: explicit (directly stated) or implicit (inferred from advice)
- **Urgency**: 🔴 high (do immediately), 🟡 medium (do soon), ⚪ low (when possible), unknown
- **Evidence**: Quote or context
- **Confidence**: How clearly this was stated as an action

**Signals for explicit action items:**
- "You should...", "I want you to...", "Your homework is..."
- "Before next time, do..."
- "Action item: ..."

**Signals for implicit action items:**
- Advice that implies practice needed
- Skills mentioned as needing improvement
- Resources mentioned to explore

### Category 5: Questions Raised

**Track topics that need further exploration:**

For EACH question:
- **Question/Topic**: What needs more exploration
- **Status**: answered (coach provided answer), unanswered (left open), needs-exploration (partially addressed)
- **Context**: Why this question matters
- **Confidence**: How clearly identified

**Look for:**
- Questions the mentee asked
- Topics the coach said "we should discuss more"
- Areas marked for follow-up
- Tangents that were noted but not explored

### Category 6: Session Quality

**Assess the coaching session quality:**

- **Actionability Score** (1-5):
  - 5: Every piece of advice is immediately actionable with clear steps
  - 4: Most advice is actionable with some general guidance
  - 3: Mix of actionable and abstract advice
  - 2: Mostly abstract principles without clear actions
  - 1: Vague or unhelpful advice

- **Examples Count**: How many concrete examples were provided
- **Confidence**: Overall assessment confidence

---

## JSON Output Schema

```json
{
  "metadata": {
    "sessionType": "CoachingSession",
    "totalLines": number,
    "analysisTimestamp": "ISO8601"
  },
  "sessionContext": {
    "coach": "name or description" | null,
    "mentee": "name or description" | null,
    "topics": ["topic1", "topic2"],
    "confidence": "HIGH"|"MEDIUM"|"LOW"|"NOT_FOUND"
  },
  "keyAdvice": [
    {
      "advice": "string",
      "domain": "communication"|"technical"|"career"|"behavioral"|"other",
      "quote": "direct quote",
      "lineNumber": number,
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "scriptsAndPatterns": [
    {
      "name": "Script Name",
      "script": "The actual quotable text...",
      "context": "When to use this",
      "lineNumber": number,
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "actionItems": [
    {
      "action": "string",
      "type": "explicit"|"implicit",
      "urgency": "high"|"medium"|"low"|"unknown",
      "evidence": { "type": "EXPLICIT"|"INFERRED", "content": "quote", "lineNumber": number },
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "questionsRaised": [
    {
      "question": "string",
      "status": "answered"|"unanswered"|"needs-exploration",
      "context": "string",
      "confidence": "HIGH"|"MEDIUM"|"LOW"
    }
  ],
  "sessionQuality": {
    "actionabilityScore": 1-5,
    "examplesCount": number,
    "confidence": "HIGH"|"MEDIUM"|"LOW"
  },
  "confidenceSummary": {
    "overall": "HIGH"|"MEDIUM"|"LOW",
    "overallScore": 0-100,
    "byCategory": {
      "sessionContext": "...",
      "keyAdvice": "...",
      "scriptsAndPatterns": "...",
      "actionItems": "...",
      "questionsRaised": "...",
      "sessionQuality": "..."
    },
    "dataQualityNotes": []
  }
}
```

---

## Markdown Output Format

```markdown
## Coaching Session Analysis

**File:** [filename]
**Session Type:** CoachingSession [Confidence: X%]
**Coach:** [name/description] | **Mentee:** [name/description]
**Topics:** [topic1], [topic2], [topic3]

---

### 1. Key Advice & Tips

#### 💬 Communication
| Advice | Quote | Confidence |
|--------|-------|------------|
| [advice] | "[quote]" (line X) | HIGH |

#### 🔧 Technical
| Advice | Quote | Confidence |
|--------|-------|------------|
| [advice] | "[quote]" (line X) | MEDIUM |

#### 📈 Career
...

#### 🎭 Behavioral
...

#### 📝 Other
...

---

### 2. Scripts & Patterns

#### [Script Name]
**Context:** [When to use this]
**Confidence:** [HIGH/MEDIUM/LOW]

```
[The actual script/pattern text, formatted as ready-to-use]
```

(Line [X])

---

### 3. Action Items

#### 🔴 High Urgency
- [ ] [Action item] - [EXPLICIT/INFERRED] (line X)

#### 🟡 Medium Urgency
- [ ] [Action item] - [EXPLICIT/INFERRED] (line X)

#### ⚪ Low Urgency
- [ ] [Action item] - [EXPLICIT/INFERRED] (line X)

---

### 4. Questions Raised

| Question/Topic | Status | Context |
|----------------|--------|---------|
| [question] | ✅ Answered / ❓ Unanswered / 🔍 Needs Exploration | [context] |

---

### 5. Session Quality

| Metric | Score | Notes |
|--------|-------|-------|
| Actionability | X/5 | [assessment] |
| Examples Given | X | [list if notable] |

---

### Confidence Summary

| Category | Confidence |
|----------|------------|
| Overall | X% (HIGH/MEDIUM/LOW) |
| Session Context | ... |
| Key Advice | ... |
| Scripts/Patterns | ... |
| Action Items | ... |
| Questions Raised | ... |
| Session Quality | ... |

**Data Quality Notes:**
- [Any caveats about the analysis]
```
