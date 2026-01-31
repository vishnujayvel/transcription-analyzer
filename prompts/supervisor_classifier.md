# Supervisor Classifier

## Purpose

Classify transcript session type BEFORE any detailed analysis. The supervisor scans the transcript for signal patterns and routes to the appropriate specialized analyzer.

---

## Session Types (Discriminated Union)

| Type | Description | Routes To |
|------|-------------|-----------|
| `MockInterview.SystemDesign` | System design practice interview | mock_interview_analyzer.md |
| `MockInterview.Coding` | Coding/algorithm interview | mock_interview_analyzer.md |
| `MockInterview.Behavioral` | Behavioral/leadership interview | mock_interview_analyzer.md |
| `CoachingSession` | Mentoring/advice session | coaching_analyzer.md |
| `GenericMeeting` | Default for any conversation | meeting_analyzer.md |

---

## Signal Patterns

### MockInterview.SystemDesign Signals
**Weight: 1.0 (strong indicators)**
- "design a system"
- "scalability"
- "database" (in design context)
- "load balancer"
- "API design"
- "high availability"
- "microservices"
- "CAP theorem"
- "partitioning"
- "replication"

### MockInterview.Coding Signals
**Weight: 1.0 (strong indicators)**
- "write a function"
- "time complexity"
- "space complexity"
- "algorithm"
- "test cases"
- "edge cases"
- "optimal solution"
- "brute force"
- "data structure"
- "Big O"

### MockInterview.Behavioral Signals
**Weight: 1.0 (strong indicators)**
- "tell me about a time"
- "STAR"
- "leadership"
- "conflict"
- "situation"
- "difficult situation"
- "disagree with"
- "mentor"
- "action"
- "result"

### CoachingSession Signals
**Weight: 1.0 (strong indicators)**
- "advice"
- "tips"
- "here's what I'd recommend"
- "you should try"
- "feedback on your"
- "let me coach you"
- "I suggest"
- "my recommendation"
- "here's a framework"
- "practice this"

### GenericMeeting Signals
**Weight: 0.5 (weak, used as fallback)**
- Default when no strong signals from other types
- Generic conversation patterns
- No clear interview or coaching structure

---

## Classification Algorithm

### Step 1: Scan Transcript
For each signal pattern defined above:
1. Search transcript (case-insensitive)
2. Record each match with:
   - Line number where found
   - Exact phrase matched
   - Signal weight (1.0 for strong, 0.5 for weak)

### Step 2: Calculate Scores
For each session type:
```
raw_score = sum(weight for each matched signal)
```

### Step 3: Normalize to Percentages
```
total_signals = sum(all raw_scores)
if total_signals > 0:
    score_percentage = (raw_score / total_signals) * 100
else:
    score_percentage = 0
```

### Step 4: Determine Confidence Level

| Highest Score | Confidence | Action |
|---------------|------------|--------|
| >= 70% | HIGH | Route directly to analyzer |
| 50-69% | MEDIUM | Suggest user confirmation |
| < 50% | LOW | Require user confirmation |
| Equal top scores | AMBIGUOUS | Ask user to choose |
| No signals | DEFAULT | Use GenericMeeting |

### Step 5: Handle Mock Interview Subtypes
If classified as MockInterview:
- Further classify subtype (SystemDesign, Coding, Behavioral)
- If subtype unclear, use "Unknown" subtype

---

## Output Contract

```json
{
  "sessionType": {
    "type": "MockInterview" | "CoachingSession" | "GenericMeeting",
    "subtype": "SystemDesign" | "Coding" | "Behavioral" | "Unknown" | null
  },
  "confidence": "HIGH" | "MEDIUM" | "LOW",
  "confidenceScore": 0-100,
  "evidence": [
    {
      "signalType": "MockInterview.SystemDesign",
      "phrase": "design a system",
      "lineNumber": 42,
      "weight": 1.0
    }
  ],
  "requiresUserConfirmation": true | false,
  "allScores": {
    "MockInterview.SystemDesign": 0-100,
    "MockInterview.Coding": 0-100,
    "MockInterview.Behavioral": 0-100,
    "CoachingSession": 0-100,
    "GenericMeeting": 0-100
  }
}
```

---

## User Confirmation Flow

When `requiresUserConfirmation` is true, present options to user:

```
**Session Type Classification**

I detected signals for multiple session types. Please confirm which best describes this transcript:

1. Mock Interview - System Design
2. Mock Interview - Coding
3. Mock Interview - Behavioral
4. Coaching/Mentoring Session
5. General Meeting/Conversation

Detected signals:
- "design a system" (line 12) → System Design
- "here's my advice" (line 45) → Coaching
```

---

## Integration with SKILL.md

After classification, the supervisor passes to SKILL.md:
1. `sessionType` - For routing to correct analyzer
2. `confidence` - For display in output badge
3. `confidenceScore` - For numerical display
4. `evidence` - For transparency in output

The child analyzer receives the session type and adjusts its analysis accordingly.
