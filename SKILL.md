---
name: transcription-analyzer
description: >
  Analyzes conversation transcripts using Supervisor Agent architecture. First classifies
  session type (MockInterview, CoachingSession, GenericMeeting), then routes to specialized
  analysis workflows. Features anti-hallucination protocol with confidence scoring and
  evidence citation for every claim. Use when reviewing mock interviews, coaching sessions,
  meetings, or saying "analyze my transcript".
license: MIT
metadata:
  author: vishnu-jayavel
  version: "2.0"
  categories: interview-prep, analysis, supervisor-agent
---

# Transcription Analyzer v2.0

Analyze conversation transcripts with intelligent session type detection and specialized analysis workflows.

## Triggers
- "analyze my transcript"
- "transcription-analyzer"
- "mock review"
- "review my transcript"
- "analyze this meeting"
- "coaching session review"
- "analyze topic flow"
- "topic flow analysis"
- "how did we deviate?"
- "show me our tangents"

---

## Session Types Supported

| Type | Description | Analysis Focus |
|------|-------------|----------------|
| `MockInterview.SystemDesign` | System design practice | Technical depth, architecture, trade-offs |
| `MockInterview.Coding` | Coding interview practice | Algorithm, complexity, edge cases |
| `MockInterview.Behavioral` | Behavioral interview practice | STAR format, leadership, communication |
| `CoachingSession` | Mentoring/advice session | Key tips, action items, scripts/patterns |
| `GenericMeeting` | Any other conversation | Summary, decisions, action items |
| `TopicFlowAnalysis` | Long multi-person discussions | Topic hierarchy, deviations, filler words, visualizations |

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

See [prompts/confidence_scorer.md](prompts/confidence_scorer.md) for detailed methodology.

---

## Phase 0: Transcript Metadata Extraction (NEW)

**CRITICAL: Run this phase BEFORE any content analysis.**

This phase prevents speaker confusion by:
1. Identifying all speakers and their aliases
2. Detecting role swaps (e.g., peer mock interviews)
3. Mapping who gives feedback to whom
4. Creating `feedback_direction_rules` to filter what's included in the report

See [prompts/phase0_metadata_extraction.md](prompts/phase0_metadata_extraction.md) for the full extraction prompt.
See [prompts/transcript_metadata_schema.json](prompts/transcript_metadata_schema.json) for the output schema.

### Phase 0 Quick Reference

**When to run:** Always, before Step 3 (Session Type Classification)

**Key output:** `transcript_metadata` JSON with:
- `participants[]` - All speakers with `is_primary_subject` flag
- `segments[]` - Time/line-based segments with role assignments
- `analysis_context.feedback_direction_rules[]` - Which feedback to include

**Role swap detection triggers:**
- "can you start by introducing yourself" (after questions were answered)
- "now let me interview you"
- "your turn to ask me questions"
- "let's swap roles"

**Example feedback_direction_rules:**
```json
{
  "feedback_direction_rules": [
    { "segment_id": 2, "feedback_from": "vishnu", "feedback_to": "anish", "include_in_report": false },
    { "segment_id": 5, "feedback_from": "anish", "feedback_to": "vishnu", "include_in_report": true }
  ]
}
```

### Downstream Agent Rules (MANDATORY)

All analysis phases MUST:
1. Check segment role_assignments before attributing ANY quote
2. Only include feedback where `include_in_report == true`
3. Never attribute feedback FROM primary subject as feedback TO them

---

## Step 1: Input Handling

### If ARGUMENTS provided:
Use the provided file path directly.

### If NO ARGUMENTS:
Use AskUserQuestion to request the file path:

```
What transcript file would you like me to analyze?

Please provide the full file path (e.g., /path/to/transcript.md)
```

---

## Step 2: File Validation

1. Use the Read tool to load the transcript file
2. Validate the file exists and contains content
3. Count total lines for delegation decision (>500 lines = use subagent)

### Error Handling:

**If file not found:**
```
Could not find transcript at: [attempted_path]
Please check the file path is correct.
```

**If file is empty:**
```
The transcript file appears to be empty.
Please provide a transcript with content to analyze.
```

---

## Step 3: Session Type Classification (SUPERVISOR AGENT)

**CRITICAL: Classify session type BEFORE any detailed analysis.**

Follow the classification algorithm in [prompts/supervisor_classifier.md](prompts/supervisor_classifier.md):

### 3.1 Scan for Signal Patterns

Scan the transcript for these signals (case-insensitive):

**MockInterview.SystemDesign signals:**
- "design a system", "scalability", "database", "load balancer", "API design"
- "high availability", "microservices", "CAP theorem", "partitioning"

**MockInterview.Coding signals:**
- "write a function", "time complexity", "algorithm", "test cases", "edge cases"
- "optimal solution", "brute force", "data structure", "Big O"

**MockInterview.Behavioral signals:**
- "tell me about a time", "STAR", "leadership", "conflict", "situation"
- "difficult situation", "disagree with", "mentor"

**CoachingSession signals:**
- "advice", "tips", "here's what I'd recommend", "you should try"
- "feedback on your", "let me coach you", "I suggest", "my recommendation"

### 3.2 Calculate Confidence Score

1. Count signal matches per session type
2. Normalize to percentage (0-100)
3. Determine confidence level:
   - >= 70%: HIGH confidence → Route directly
   - 50-69%: MEDIUM confidence → Suggest confirmation
   - < 50%: LOW confidence → Require confirmation

### 3.3 User Confirmation (if needed)

If confidence is LOW or multiple types have similar scores, use AskUserQuestion:

```
**Session Type Classification**

I detected signals for multiple session types. Please confirm which best describes this transcript:

Options:
- Mock Interview - System Design
- Mock Interview - Coding
- Mock Interview - Behavioral
- Coaching/Mentoring Session
- General Meeting/Conversation

Detected signals:
[List top signals found with line numbers]
```

### 3.4 Output Session Type Badge

Display at the top of every report:

```markdown
**Session Type:** MockInterview.SystemDesign [Confidence: HIGH 92%]
```
or
```markdown
**Session Type:** CoachingSession [Confidence: MEDIUM 68%]
```
or
```markdown
**Session Type:** GenericMeeting [Confidence: LOW - Default]
```

---

## Step 4: Route to Specialized Analyzer

Based on session type classification, route to the appropriate workflow:

### If MockInterview.*:
→ Continue to **Step 5: Mock Interview Analysis**
→ Use [prompts/mock_interview_analyzer.md](prompts/mock_interview_analyzer.md)

### If CoachingSession:
→ Jump to **Step 10: Coaching Session Analysis**
→ Use [prompts/coaching_analyzer.md](prompts/coaching_analyzer.md)

### If GenericMeeting:
→ Jump to **Step 11: Generic Meeting Analysis**
→ Use [prompts/meeting_analyzer.md](prompts/meeting_analyzer.md)

### If TopicFlowAnalysis:
→ Jump to **Step 12: Topic Flow Analysis**
→ Use [prompts/topic_flow_orchestrator.md](prompts/topic_flow_orchestrator.md)

---

## Step 5: Interview Start Detection (MockInterview only)

Scan the transcript for trigger phrases that indicate when the actual interview begins:

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

## Step 6: Mock Interview Subtype Context

Adjust analysis emphasis based on detected subtype:

### SystemDesign:
- Emphasize architecture decisions, scalability trade-offs, component design
- Focus on design patterns and system thinking

### Coding:
- Emphasize algorithm choice, complexity analysis, edge case handling
- Focus on code quality signals and optimization discussion

### Behavioral:
- Emphasize STAR format usage, leadership signals, communication quality
- Focus on storytelling and impact articulation

---

## Step 7: Optional Diagram Analysis (System Design Only)

**IF interview type is "MockInterview.SystemDesign":**

Ask user if they have an architecture diagram to analyze alongside the transcript.

**IF diagram provided:**
Analyze:
- Components identified (services, databases, caches, queues)
- Data flow clarity (request paths, async flows)
- Missing components vs. verbal description
- Naming quality
- Diagram Quality Score (1-10)

**IF no diagram:**
```
[Confidence: NOT_FOUND] No diagram provided for analysis.
Tip: Save diagrams from future interviews for more comprehensive review.
```

---

## Step 8: 10-Category Mock Interview Analysis

For transcripts over 500 lines, delegate to subagent using Task tool with `subagent_type: "Explore"`.

Extract insights with confidence scoring and evidence citation for each category:

### Category 1: Scorecard
- **Overall performance** (1-10 scale) - Look for explicit feedback
- **Level assessment** (E5/E6/E7/Staff+) - Look for explicit statements or infer
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

### Category 4: Things That Went Well
**IMPORTANT: Show positives BEFORE mistakes (ADHD-friendly ordering)**
- Explicit praise from interviewer
- Demonstrated strengths
- Approaches that worked

### Category 5: Mistakes Identified
For EACH mistake:
- Title and description
- Severity: 🔴 CRITICAL, 🟠 HIGH, 🟡 MEDIUM, ⚪ LOW
- Category: Fundamentals, API Design, Patterns, Domain Knowledge, Communication
- Direct evidence with line number

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
- Classification: ✅ Correct, ❌ Wrong, ❓ Needs Verification
- Correction if wrong

### Category 9: Action Items
- Explicit recommendations from interviewer
- Resources recommended (books, sites, problems)

### Category 10: Interviewer Quality
- Feedback actionability (1-5 scale)
- Specific examples given (count)
- Teaching moments

---

## Step 9: Mock Interview Output Formatting

Structure the report as:

```markdown
## Mock Interview Analysis

**File:** [filename]
**Session Type:** MockInterview.[subtype] [Confidence: X%]
**Date Analyzed:** [timestamp]

---

### 1. Scorecard
[Overall score, level assessment, dimensional scores, readiness %]

### 2. Time Breakdown
[Duration, phase timings]

### 3. Communication Signals
[Talk ratio, filler words, clarifying questions]

### 4. ⭐ Things That Went Well
[Positives with evidence - BEFORE mistakes]

### 5. Mistakes Identified
[Severity-coded mistakes with evidence]

### 6. Knowledge Gaps
[Priority-coded gaps]

### 7. Behavioral Assessment
[Staff+ signals]

### 8. Factual Claims
[Verification status]

### 9. Action Items
[Recommendations and resources]

### 10. Interviewer Quality
[Actionability assessment]

---

### Confidence Summary
[Overall confidence, by-category breakdown, data quality notes]
```

After the markdown report, output a JSON summary for programmatic consumption.

→ **END of MockInterview workflow**

---

## Step 10: Coaching Session Analysis

**ONLY execute this step if session type is CoachingSession**

Use [prompts/coaching_analyzer.md](prompts/coaching_analyzer.md) for detailed extraction.

### 6-Category Extraction:

#### Category 1: Session Context
- Identify coach/mentor and mentee names
- Extract session topic(s) discussed

#### Category 2: Key Advice/Tips
- List all explicit recommendations from coach
- Categorize by domain: communication, technical, career, behavioral, other
- Include direct quotes with line numbers

#### Category 3: Scripts/Patterns
- Extract reusable phrases or frameworks taught
- Format as quotable, ready-to-use text
- Note context for when to use each script

#### Category 4: Action Items
- Explicit tasks assigned by coach
- Implicit tasks (things to practice, review)
- Prioritize by urgency: 🔴 High, 🟡 Medium, ⚪ Low

#### Category 5: Questions Raised
- Topics needing further exploration
- Unanswered questions from session

#### Category 6: Session Quality
- Actionability score (1-5)
- Concrete examples count

### Coaching Output Format:

```markdown
## Coaching Session Analysis

**File:** [filename]
**Session Type:** CoachingSession [Confidence: X%]
**Coach:** [name] | **Mentee:** [name]
**Topics:** [topic1], [topic2]

---

### 1. Key Advice & Tips
[Categorized by domain with direct quotes]

### 2. Scripts & Patterns
[Quotable text with usage context]

### 3. Action Items
[Explicit and implicit tasks with urgency]

### 4. Questions Raised
[Topics needing exploration]

### 5. Session Quality
[Actionability score, examples count]

---

### Confidence Summary
```

→ **END of CoachingSession workflow**

---

## Step 11: Generic Meeting Analysis

**ONLY execute this step if session type is GenericMeeting**

Use [prompts/meeting_analyzer.md](prompts/meeting_analyzer.md) for detailed extraction.

### 6-Category Extraction:

#### Category 1: Meeting Context
- Identify participants (if detectable)
- Identify meeting purpose/topic

#### Category 2: Summary
- 3-5 sentence executive summary
- Most important points discussed

#### Category 3: Decisions Made
- All explicit decisions reached
- Owner for each decision (if mentioned)

#### Category 4: Action Items
- All tasks assigned
- Owner and deadline (if mentioned)

#### Category 5: Open Questions
- Unresolved topics
- Topics needing follow-up

#### Category 6: Key Quotes
- Memorable or important statements
- Speaker attribution and line number

### Meeting Output Format:

```markdown
## Meeting Analysis

**File:** [filename]
**Session Type:** GenericMeeting [Confidence: X%]
**Participants:** [list]
**Purpose:** [detected purpose]

---

### 1. Executive Summary
[3-5 sentence summary]

### 2. Key Decisions
[Decisions with owners]

### 3. Action Items
[Tasks with owners and deadlines]

### 4. Open Questions
[Unresolved topics needing follow-up]

### 5. Key Quotes
[Memorable statements with attribution]

---

### Confidence Summary
```

→ **END of GenericMeeting workflow**

---

## Step 12: Topic Flow Analysis

**ONLY execute this step if session type is TopicFlowAnalysis**

Use [prompts/topic_flow_orchestrator.md](prompts/topic_flow_orchestrator.md) for the full workflow.

### Three-Phase Map-Reduce Approach:

#### Phase 1: Map (Chunk Analysis)
- Split transcript into 300-line chunks with 50-line overlap
- For each chunk: extract topics, speakers, timestamps, deviations, filler words
- Output intermediate JSON per chunk

#### Phase 2: Reduce (Merge & Reconcile)
- Merge topic hierarchies across chunks
- Deduplicate and reconcile overlapping segments
- Build global topic tree with parent-child relationships

#### Phase 3: Synthesize (Visualizations & Insights)
- Generate Topic Hierarchy (tree structure)
- Generate Sankey diagram data (topic flow)
- Generate Timeline visualization data
- Identify deviation patterns and tangent analysis
- Aggregate filler word statistics by speaker
- Produce actionable insights

### Topic Flow Output Format:

```markdown
## Topic Flow Analysis

**File:** [filename]
**Session Type:** TopicFlowAnalysis [Confidence: X%]
**Duration:** [total time] | **Speakers:** [count]

---

### 1. Topic Hierarchy
[Tree structure of main topics → subtopics]

### 2. Flow Visualization (Sankey Data)
[JSON for Sankey diagram: topic transitions with weights]

### 3. Timeline
[Chronological topic progression with timestamps]

### 4. Deviations & Tangents
[Where conversation deviated from main topics, duration, return points]

### 5. Filler Word Analysis
[Per-speaker breakdown: um, uh, like, you know, basically, etc.]

### 6. Insights
[Patterns, recommendations, conversation quality metrics]

---

### Confidence Summary
```

→ **END of TopicFlowAnalysis workflow**

---

## Subagent Delegation (Large Transcripts)

For transcripts exceeding 500 lines, use the Task tool to delegate analysis:

```
Task tool with subagent_type: "Explore"
Prompt: [Include appropriate analyzer prompt + transcript content]
```

Validate JSON response structure before displaying results.

If subagent fails:
1. Attempt direct analysis with truncation warning
2. Analyze available portion
3. Note incomplete analysis in output

---

## Portability Constraints

This skill MUST remain portable and dependency-free:

**PROHIBITED references:**
- `mcp__mem0__*` - No memory service
- `mcp__obsidian__*` - No note-taking service
- `/Users/vishnu/` - No personal paths
- `study plan` - No integration with other skills
- `gotcha` - No gotcha tracking

**ALLOWED tools only:**
- `Read` - File reading
- `AskUserQuestion` - User interaction
- `Task` - Subagent delegation (with `subagent_type: "Explore"`)

---

## Examples

- [examples/sample_transcript.md](examples/sample_transcript.md) - Example mock interview transcript
- [examples/sample_output.md](examples/sample_output.md) - Example analysis output

## Prompts

- [prompts/phase0_metadata_extraction.md](prompts/phase0_metadata_extraction.md) - **Phase 0:** Speaker/role extraction (run first!)
- [prompts/transcript_metadata_schema.json](prompts/transcript_metadata_schema.json) - JSON schema for metadata output
- [prompts/supervisor_classifier.md](prompts/supervisor_classifier.md) - Session type classification
- [prompts/mock_interview_analyzer.md](prompts/mock_interview_analyzer.md) - Mock interview analysis
- [prompts/coaching_analyzer.md](prompts/coaching_analyzer.md) - Coaching session analysis
- [prompts/meeting_analyzer.md](prompts/meeting_analyzer.md) - Generic meeting analysis
- [prompts/topic_flow_orchestrator.md](prompts/topic_flow_orchestrator.md) - Topic flow analysis (Map-Reduce)
- [prompts/confidence_scorer.md](prompts/confidence_scorer.md) - Confidence methodology
