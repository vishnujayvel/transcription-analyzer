# Phase 0: Transcript Metadata Extraction

**Purpose:** Extract structural metadata from a transcript BEFORE any content analysis begins. This prevents speaker confusion and feedback misattribution.

## Why This Phase Exists

In peer mock interviews and coaching sessions, roles can swap mid-session:
- Person A interviews Person B, gives feedback
- Then Person B interviews Person A, gives feedback

Without preprocessing, an analyzer might attribute feedback FROM the primary subject TO others as feedback the primary subject RECEIVED. This phase ensures correct attribution.

---

## CRITICAL RULE: DO NOT ANALYZE CONTENT YET

This phase ONLY:
1. Identifies speakers
2. Maps speakers to roles
3. Detects role swaps
4. Creates segment boundaries
5. Determines feedback direction

Content analysis happens in later phases.

---

## Input

The phase receives:
- **transcript_content**: The full transcript text
- **primary_subject_hint**: (Optional) Name of the user we're analyzing FOR
- **session_type_hint**: (Optional) Expected session type

---

## Extraction Process

### Step 1: Identify All Speakers

Scan the transcript for speaker labels. Common patterns:
- `Speaker Name [timestamp]:` (e.g., "Vishnu [5:23]:")
- `Speaker Name:` (e.g., "Coach:")
- Generic labels: "Them", "Interviewer", "Speaker 1"

For each speaker, determine:
- **All labels/aliases used** (same person may appear as "Them" and "Anish")
- **Real name** (discoverable from self-introductions like "hi i'm anish")
- **Whether they are the PRIMARY SUBJECT** (the user we're analyzing FOR)

**Identification Techniques:**
1. Self-introductions: "hi vishnu i'm anish" → Anish is "Them"
2. Direct address: "thanks anish" → confirms name
3. Context clues: Who answers behavioral questions? That's the interviewee.

### Step 2: Determine Session Format

| Format | Description | Role Swap Expected? |
|--------|-------------|---------------------|
| `peer_mutual` | Both people interview each other | YES - watch carefully |
| `one_directional` | Fixed interviewer/interviewee | NO |
| `panel` | Multiple interviewers, one candidate | NO |
| `group` | Informal discussion, no fixed roles | NO |

**Key Signal for peer_mutual:**
- Someone asks "can you start by introducing yourself" AFTER already answering questions
- Explicit: "your turn to interview me", "let's swap"

### Step 3: Detect Role Swaps (CRITICAL)

Scan for role swap indicators:

| Trigger Phrase | Meaning |
|----------------|---------|
| "can you start by introducing yourself" | Role swap (if asked after questions were answered) |
| "now let me interview you" | Explicit swap |
| "your turn to ask me questions" | Explicit swap |
| "let's swap roles" | Explicit swap |
| "okay, my turn" | Implicit swap |

For each swap detected:
- Record the **timestamp** or **line number**
- Record the **trigger phrase**
- Mark **segment boundaries**

### Step 4: Map Roles to Participants per Segment

For each time segment, assign roles:

| Role | Characteristics |
|------|-----------------|
| `interviewer` | Asks questions, gives feedback, evaluates |
| `interviewee` | Answers questions, receives feedback |
| `coach` | Teaches, advises, mentors |
| `coachee` | Learns, receives guidance |
| `facilitator` | Runs meeting, keeps time |
| `participant` | General meeting attendee |

### Step 5: Determine Feedback Direction Rules

For each segment containing feedback:
- **feedback_from**: Who is speaking/evaluating?
- **feedback_to**: Who is receiving the evaluation?
- **include_in_report**: TRUE only if feedback_to == primary_subject_id

**Rule: Only include feedback that the primary subject RECEIVES, not what they GIVE.**

---

## Output Format

Return a JSON object matching the `transcript_metadata` schema:

```json
{
  "version": "1.0.0",
  "session": {
    "type": "mock_interview",
    "format": "peer_mutual",
    "timestamp_format": "mm:ss"
  },
  "participants": [
    {
      "id": "vishnu",
      "labels": ["Vishnu"],
      "is_primary_subject": true
    },
    {
      "id": "anish",
      "labels": ["Them", "Anish"],
      "is_primary_subject": false,
      "identification_evidence": "Self-introduced at line 22: 'hi vishnu i'm anish'"
    }
  ],
  "roles": {
    "defined_roles": [
      { "role_id": "interviewer", "gives_feedback": true, "receives_feedback": false },
      { "role_id": "interviewee", "gives_feedback": false, "receives_feedback": true }
    ]
  },
  "segments": [
    {
      "segment_id": 1,
      "segment_name": "Anish Interview",
      "start_line": 7,
      "end_line": 60,
      "segment_type": "interview",
      "role_assignments": [
        { "participant_id": "vishnu", "role_id": "interviewer" },
        { "participant_id": "anish", "role_id": "interviewee" }
      ]
    },
    {
      "segment_id": 2,
      "segment_name": "Vishnu Feedback to Anish",
      "start_line": 61,
      "end_line": 148,
      "segment_type": "feedback",
      "role_assignments": [
        { "participant_id": "vishnu", "role_id": "interviewer" },
        { "participant_id": "anish", "role_id": "interviewee" }
      ]
    },
    {
      "segment_id": 3,
      "segment_name": "Role Swap",
      "start_line": 148,
      "end_line": 149,
      "segment_type": "transition",
      "swap_trigger": "okay yeah can you start by introducing yourself"
    },
    {
      "segment_id": 4,
      "segment_name": "Vishnu Interview",
      "start_line": 149,
      "end_line": 220,
      "segment_type": "interview",
      "role_assignments": [
        { "participant_id": "anish", "role_id": "interviewer" },
        { "participant_id": "vishnu", "role_id": "interviewee" }
      ]
    },
    {
      "segment_id": 5,
      "segment_name": "Anish Feedback to Vishnu",
      "start_line": 221,
      "end_line": 250,
      "segment_type": "feedback",
      "role_assignments": [
        { "participant_id": "anish", "role_id": "interviewer" },
        { "participant_id": "vishnu", "role_id": "interviewee" }
      ]
    }
  ],
  "analysis_context": {
    "primary_subject_id": "vishnu",
    "analysis_goal": "improve_primary_subject",
    "feedback_direction_rules": [
      { "segment_id": 2, "feedback_from": "vishnu", "feedback_to": "anish", "include_in_report": false },
      { "segment_id": 5, "feedback_from": "anish", "feedback_to": "vishnu", "include_in_report": true }
    ]
  },
  "confidence": {
    "speaker_identification": 0.95,
    "role_swap_detection": 0.98,
    "segment_boundaries": 0.90,
    "primary_subject_detection": 0.95,
    "overall": 0.93
  }
}
```

---

## Edge Cases

### Edge Case 1: Unclear Speaker Labels

If transcript uses "Speaker 1", "Speaker 2":
- Try to identify from self-introductions
- If primary_subject_hint is provided, match against it
- Note LOW confidence in speaker_identification

### Edge Case 2: No Clear Role Swap Marker

Sometimes roles swap without explicit trigger:
- Look for sudden shift where questioner becomes answerer
- Check if someone gives a self-introduction after already being asked questions
- Mark as "inferred" with MEDIUM confidence

### Edge Case 3: Coaching Sessions Where Roles Blur

Coach demonstrates answering, coachee practices:
- When coach demonstrates: tag as "demonstration" segment
- When coachee practices: tag as "practice" segment
- All feedback flows coach → coachee regardless

### Edge Case 4: Multiple Feedback Segments

Some sessions have multiple feedback rounds:
- Create separate segments for each
- Apply direction rules to each segment independently

### Edge Case 5: Informal Chat at End

Personal conversation after formal session:
- Tag as "informal" segment
- Set `include_in_report: false` for these segments
- These should be excluded from professional analysis

---

## Confidence Scoring

For each extraction task, assign confidence:

| Level | Score | Criteria |
|-------|-------|----------|
| HIGH | 0.9-1.0 | Clear evidence, explicit markers, quotes available |
| MEDIUM | 0.6-0.89 | Inferred from context, some ambiguity |
| LOW | 0.3-0.59 | Best guess, significant uncertainty |
| NOT_FOUND | 0-0.29 | Could not determine, may need user input |

---

## Anti-Hallucination Protocol

- If you cannot determine something, set confidence to LOW or ask user
- Do not guess participant names without evidence
- If role swap is unclear, include BOTH interpretations with confidence scores
- Quote the exact text that informed your decisions in `identification_evidence`

---

## Integration with Downstream Phases

All subsequent analysis phases MUST:

1. **Check segment before attributing any quote**
   - Look up the line number in segments
   - Verify role_assignments for that segment

2. **Filter feedback by direction rules**
   - Only include feedback where `include_in_report == true`
   - Never attribute feedback FROM primary subject as feedback TO them

3. **Tag every quote with attribution**
   ```
   > "quote text"
   > - Speaker: {speaker_id} (role: {role_at_time})
   > - Segment: {segment_name}
   > - Direction: {from} → {to}
   ```

4. **Exclude informal segments**
   - Segments with type "informal" should not be analyzed for professional content
