# Requirements Document

## Introduction

The **Transcription Analyzer** is an open-source, standalone Claude Code skill that analyzes conversation transcripts using a **Supervisor Agent architecture**. It classifies session types first, then routes to specialized analysis workflows for mock interviews, coaching sessions, or generic meetings.

**Business Value:** Enables users to get structured, actionable feedback from any recorded conversation - mock interviews, mentoring sessions, coaching calls, or meetings - without requiring external services. The anti-hallucination protocol ensures all insights are evidence-based and trustworthy.

**Source:** Adapted from `.claude/skills/daily-copilot/study/mock_interview_review.md` (765 lines)

**Key Tenets:**
- `supervisor-agent-architecture` - Classify session type before analysis
- `strongly-typed-sessions` - Distinct workflows for each session type
- `pure-cli-portable` - No external service dependencies
- `anti-hallucination-protocol` - Confidence scoring on ALL metrics
- `no-external-dependencies` - Works with just a transcript file path

**Session Types Supported:**
| Type | Description | Analysis Focus |
|------|-------------|----------------|
| `MockInterview.SystemDesign` | System design practice | Technical depth, architecture, trade-offs |
| `MockInterview.Coding` | Coding interview practice | Algorithm, complexity, edge cases |
| `MockInterview.Behavioral` | Behavioral interview practice | STAR format, leadership signals, communication |
| `CoachingSession` | Mentoring/advice session | Key tips, action items, scripts/patterns |
| `GenericMeeting` | Any other conversation | Summary, decisions, action items |
| `TopicFlowAnalysis` | Long multi-person discussions | Topic hierarchy, deviations, filler words, visualizations |

---

## Requirements

### Requirement 0: Supervisor Agent Architecture

**Objective:** As a user, I want the analyzer to first classify what type of session the transcript represents, so that it can apply the appropriate analysis workflow.

#### Acceptance Criteria

1. WHEN a transcript is loaded THEN the Supervisor Agent SHALL classify the session type BEFORE any detailed analysis.

2. WHEN classifying session type THEN the Supervisor Agent SHALL detect signals for each type:
   - **MockInterview.SystemDesign**: "design a system", "scalability", "database", "load balancer", "API design"
   - **MockInterview.Coding**: "write a function", "time complexity", "algorithm", "test cases", "edge cases"
   - **MockInterview.Behavioral**: "tell me about a time", "STAR", "leadership", "conflict", "situation"
   - **CoachingSession**: "advice", "tips", "here's what I'd recommend", "you should try", "feedback on your", "let me coach you"
   - **GenericMeeting**: Default if no strong signals for other types

3. WHEN session type is classified THEN the Supervisor Agent SHALL output the type with confidence level and evidence.

4. WHEN session type is determined THEN the Supervisor Agent SHALL route to the appropriate analysis workflow:
   - MockInterview.* → 10-Category Mock Interview Analysis
   - CoachingSession → Coaching Session Analysis (see Requirement 11)
   - GenericMeeting → Generic Meeting Analysis (see Requirement 12)

5. IF session type cannot be determined with HIGH confidence THEN the Supervisor Agent SHALL ask the user to confirm the type.

6. WHEN routing to a workflow THEN the Supervisor Agent SHALL pass the classified type and confidence to the child analyzer.

---

### Requirement 1: Transcript Input

**Objective:** As a user, I want to provide a transcript file path for analysis, so that I can analyze any mock interview transcription regardless of where it's stored.

#### Acceptance Criteria

1. WHEN the skill is invoked without arguments THEN the Transcription Analyzer SHALL use AskUserQuestion to request the file path from the user.

2. WHEN the skill is invoked with a file path argument THEN the Transcription Analyzer SHALL use that path directly without prompting.

3. WHEN a file path is provided THEN the Transcription Analyzer SHALL validate that the file exists and is readable.

4. IF the file does not exist or is not readable THEN the Transcription Analyzer SHALL display a clear error message with the attempted path.

5. WHEN reading the transcript THEN the Transcription Analyzer SHALL support markdown (.md) and plain text (.txt) file formats.

6. WHEN the transcript is loaded THEN the Transcription Analyzer SHALL count the total lines to determine if subagent delegation is needed.

---

### Requirement 2: Interview Start Detection

**Objective:** As a user, I want the analyzer to automatically skip small talk and focus on the actual interview content, so that analysis is relevant and not polluted by casual conversation.

#### Acceptance Criteria

1. WHEN analyzing a transcript THEN the Transcription Analyzer SHALL detect interview start trigger phrases.

2. WHERE trigger phrases include "go design", "let's get started", "the problem is", "design a system", "let's dive into", "first question", or "walk me through" THEN the Transcription Analyzer SHALL mark the interview start point.

3. IF no trigger phrase is detected THEN the Transcription Analyzer SHALL analyze from the beginning with confidence "LOW" on time breakdown metrics.

4. WHEN the interview start is detected THEN the Transcription Analyzer SHALL note the line number in the analysis output.

---

### Requirement 3: Mock Interview Type Detection

**Objective:** As a user with a mock interview transcript, I want the analyzer to detect the specific interview type, so that the 10-category analysis is contextualized appropriately.

**Note:** This requirement applies AFTER the Supervisor Agent (Requirement 0) has classified the session as `MockInterview.*`.

#### Acceptance Criteria

1. WHEN the Supervisor Agent classifies as MockInterview THEN the Transcription Analyzer SHALL further classify the interview subtype.

2. WHERE signals include "design a system", "scalability", "database schema" THEN the Transcription Analyzer SHALL classify as "MockInterview.SystemDesign".

3. WHERE signals include "write a function", "time complexity", "test cases" THEN the Transcription Analyzer SHALL classify as "MockInterview.Coding".

4. WHERE signals include "tell me about a time", "leadership", "conflict" THEN the Transcription Analyzer SHALL classify as "MockInterview.Behavioral".

5. WHEN interview type is detected THEN the Transcription Analyzer SHALL include the type in the output with confidence level and evidence.

6. IF interview subtype cannot be determined THEN the Transcription Analyzer SHALL output "MockInterview.Unknown" and apply generic 10-category analysis.

---

### Requirement 4: Anti-Hallucination Protocol

**Objective:** As a user, I want every insight to include confidence scoring and evidence citations, so that I can trust the analysis and distinguish facts from inferences.

#### Acceptance Criteria

1. WHILE analyzing any metric THEN the Transcription Analyzer SHALL assign a confidence level to every insight.

2. WHERE confidence level is HIGH (90%+) THEN the Transcription Analyzer SHALL require direct quote or explicit statement from transcript.

3. WHERE confidence level is MEDIUM (60-89%) THEN the Transcription Analyzer SHALL require inference from context with multiple supporting signals.

4. WHERE confidence level is LOW (30-59%) THEN the Transcription Analyzer SHALL note single weak signal or ambiguous evidence.

5. WHERE confidence level is NOT_FOUND (0%) THEN the Transcription Analyzer SHALL explicitly state "Not found in transcript" without fabrication.

6. WHEN citing evidence THEN the Transcription Analyzer SHALL include line number or direct quote.

7. WHEN making inferences THEN the Transcription Analyzer SHALL mark clearly with `[INFERRED]` vs `[EXPLICIT]` tags.

8. WHEN calculating aggregate scores THEN the Transcription Analyzer SHALL use weighted average of component confidence scores.

---

### Requirement 5: 10-Category Analytics Framework

**Objective:** As a user, I want comprehensive analysis across 10 distinct categories, so that I get a complete picture of my mock interview performance.

#### Acceptance Criteria

##### Category 1: Scorecard
1. WHEN analyzing scorecard THEN the Transcription Analyzer SHALL extract overall performance impression (1-10 scale).
2. WHEN analyzing scorecard THEN the Transcription Analyzer SHALL identify level signals (E5/E6/E7/Staff+).
3. WHEN analyzing scorecard THEN the Transcription Analyzer SHALL assess dimensions: communication, technical depth, structure, leadership.
4. WHEN calculating readiness THEN the Transcription Analyzer SHALL apply formula: `100 - (P0_gaps × 15) - (P1_gaps × 5) - (CRITICAL_mistakes × 20) - (HIGH_mistakes × 10) - (MEDIUM_mistakes × 3)`.

##### Category 2: Time Breakdown
5. WHEN analyzing time THEN the Transcription Analyzer SHALL extract total interview duration if detectable.
6. WHEN analyzing time THEN the Transcription Analyzer SHALL identify phase timings: requirements gathering, high-level design, deep dives, Q&A.
7. WHEN analyzing time THEN the Transcription Analyzer SHALL note any time-related feedback from interviewer.

##### Category 3: Communication Signals
8. WHEN analyzing communication THEN the Transcription Analyzer SHALL estimate talk ratio between candidate and interviewer.
9. WHEN analyzing communication THEN the Transcription Analyzer SHALL detect long pauses, filler words (um, uh, like, you know, basically).
10. WHEN analyzing communication THEN the Transcription Analyzer SHALL count clarifying questions asked by candidate.
11. WHEN analyzing communication THEN the Transcription Analyzer SHALL identify course corrections after feedback.

##### Category 4: Mistakes Identified
12. WHEN identifying mistakes THEN the Transcription Analyzer SHALL categorize by severity: CRITICAL, HIGH, MEDIUM, LOW.
13. WHEN identifying mistakes THEN the Transcription Analyzer SHALL categorize by type: Fundamentals, API Design, Patterns, Domain Knowledge, Communication.
14. WHEN identifying mistakes THEN the Transcription Analyzer SHALL include direct evidence (interviewer correction, explicit feedback).

##### Category 5: Things That Went Well
15. WHEN identifying positives THEN the Transcription Analyzer SHALL extract explicit praise from interviewer.
16. WHEN identifying positives THEN the Transcription Analyzer SHALL note demonstrated strengths and effective approaches.

##### Category 6: Knowledge Gaps
17. WHEN identifying gaps THEN the Transcription Analyzer SHALL categorize by area: Fundamentals, API Design, Patterns, Domain.
18. WHEN identifying gaps THEN the Transcription Analyzer SHALL assign priority: P0 (must fix), P1 (important), P2 (nice to have).

##### Category 7: Behavioral Assessment
19. WHEN assessing behavioral signals THEN the Transcription Analyzer SHALL evaluate leadership presence.
20. WHEN assessing behavioral signals THEN the Transcription Analyzer SHALL analyze trade-off discussions and decision defense.
21. WHEN assessing behavioral signals THEN the Transcription Analyzer SHALL assess handling of pushback.

##### Category 8: Factual Claims
22. WHEN evaluating claims THEN the Transcription Analyzer SHALL list technical claims made by candidate.
23. WHEN evaluating claims THEN the Transcription Analyzer SHALL classify as Correct, Wrong, or Needs Verification.
24. IF a claim is wrong THEN the Transcription Analyzer SHALL provide the correct information if evident from transcript.

##### Category 9: Action Items
25. WHEN extracting action items THEN the Transcription Analyzer SHALL identify explicit recommendations from interviewer.
26. WHEN extracting action items THEN the Transcription Analyzer SHALL note resources recommended (books, sites, problems).

##### Category 10: Interviewer Quality
27. WHEN assessing interviewer THEN the Transcription Analyzer SHALL rate feedback actionability (1-5 scale).
28. WHEN assessing interviewer THEN the Transcription Analyzer SHALL count specific examples given.
29. WHEN assessing interviewer THEN the Transcription Analyzer SHALL identify teaching moments.

---

### Requirement 6: Subagent Delegation

**Objective:** As a user analyzing large transcripts, I want efficient processing that doesn't overwhelm context limits, so that even long interviews can be analyzed completely.

#### Acceptance Criteria

1. WHEN transcript exceeds 500 lines THEN the Transcription Analyzer SHALL delegate analysis to an Explore subagent.

2. WHEN delegating to subagent THEN the Transcription Analyzer SHALL provide the complete 10-category extraction prompt.

3. WHEN subagent returns results THEN the Transcription Analyzer SHALL validate JSON structure before display.

4. IF subagent fails or returns invalid data THEN the Transcription Analyzer SHALL attempt direct analysis with truncation warning.

5. WHEN using subagent THEN the Transcription Analyzer SHALL preserve all anti-hallucination requirements in the subagent prompt.

---

### Requirement 7: Output Formats

**Objective:** As a user, I want analysis output in both human-readable and machine-readable formats, so that I can review results and potentially integrate with other tools.

#### Acceptance Criteria

##### Markdown Report
1. WHEN generating output THEN the Transcription Analyzer SHALL produce a structured markdown report with all 10 categories.
2. WHEN generating markdown THEN the Transcription Analyzer SHALL include confidence indicators for every metric.
3. WHEN generating markdown THEN the Transcription Analyzer SHALL use tables, headers, and emoji indicators for visual clarity.
4. WHEN generating markdown THEN the Transcription Analyzer SHALL include a confidence summary section at the end.

##### JSON Summary
5. WHEN generating output THEN the Transcription Analyzer SHALL produce a JSON summary alongside the markdown.
6. WHEN generating JSON THEN the Transcription Analyzer SHALL structure data with nested objects for each category.
7. WHEN generating JSON THEN the Transcription Analyzer SHALL include confidence and evidence fields for each metric.
8. IF data is NOT_FOUND for any category THEN the Transcription Analyzer SHALL output: `{"category": "X", "status": "NOT_FOUND", "reason": "No evidence in transcript"}`.

---

### Requirement 8: Portability Constraints

**Objective:** As an open-source user, I want the skill to work without external dependencies, so that anyone can use it without special setup.

#### Acceptance Criteria

1. WHILE implementing the skill THEN the Transcription Analyzer SHALL NOT reference Mem0 or any memory service.

2. WHILE implementing the skill THEN the Transcription Analyzer SHALL NOT reference Obsidian or any note-taking service.

3. WHILE implementing the skill THEN the Transcription Analyzer SHALL NOT include personal file paths or hardcoded directories.

4. WHILE implementing the skill THEN the Transcription Analyzer SHALL NOT create study plans or integrate with other skills.

5. WHILE implementing the skill THEN the Transcription Analyzer SHALL NOT create gotchas or write to external databases.

6. WHEN the skill is invoked THEN the Transcription Analyzer SHALL work with only: transcript file path, Claude Code, and the Task tool for subagent delegation.

---

### Requirement 9: Diagram Analysis (Optional)

**Objective:** As a user with system design mocks, I want the option to include architecture diagram analysis, so that visual artifacts can be evaluated alongside the transcript.

#### Acceptance Criteria

1. WHEN analyzing a System Design interview THEN the Transcription Analyzer SHALL ask if user has an architecture diagram to share.

2. IF user provides a diagram THEN the Transcription Analyzer SHALL analyze components, data flows, and gaps vs verbal description.

3. IF user declines or no diagram is available THEN the Transcription Analyzer SHALL output: `[Confidence: NOT_FOUND] No diagram provided for analysis`.

4. WHEN diagram is analyzed THEN the Transcription Analyzer SHALL include a Diagram Quality Score (1-10).

---

### Requirement 10: ADHD-Friendly Design

**Objective:** As a user with ADHD, I want the output to be structured for easy scanning and action-taking, so that I can quickly understand results without cognitive overload.

#### Acceptance Criteria

1. WHEN displaying results THEN the Transcription Analyzer SHALL show accomplishments (Things That Went Well) BEFORE mistakes.

2. WHEN displaying severity levels THEN the Transcription Analyzer SHALL use visual indicators (emoji) for quick scanning.

3. WHEN generating action items THEN the Transcription Analyzer SHALL prioritize P0 items first with clear formatting.

4. WHEN displaying confidence levels THEN the Transcription Analyzer SHALL use a simple legend and consistent formatting.

5. WHILE generating output THEN the Transcription Analyzer SHALL use single clear recommendations, not multiple competing options.

---

### Requirement 11: Coaching Session Analysis

**Objective:** As a user who had a coaching or mentoring session, I want to extract actionable advice and patterns, so that I can apply the mentor's guidance effectively.

#### Acceptance Criteria

##### Category 1: Session Context
1. WHEN analyzing a coaching session THEN the Transcription Analyzer SHALL identify the coach/mentor and mentee.
2. WHEN analyzing a coaching session THEN the Transcription Analyzer SHALL identify the session topic(s) discussed.

##### Category 2: Key Advice/Tips
3. WHEN extracting advice THEN the Transcription Analyzer SHALL list all explicit recommendations from the coach.
4. WHEN extracting advice THEN the Transcription Analyzer SHALL categorize tips by domain (communication, technical, career, etc.).
5. WHEN extracting advice THEN the Transcription Analyzer SHALL include direct quotes with line numbers as evidence.

##### Category 3: Scripts/Patterns
6. WHEN identifying scripts THEN the Transcription Analyzer SHALL extract any reusable phrases or frameworks taught.
7. WHEN identifying scripts THEN the Transcription Analyzer SHALL format scripts as quotable, ready-to-use text.
8. WHEN identifying scripts THEN the Transcription Analyzer SHALL note the context for when to use each script.

##### Category 4: Action Items
9. WHEN extracting action items THEN the Transcription Analyzer SHALL identify explicit tasks assigned by the coach.
10. WHEN extracting action items THEN the Transcription Analyzer SHALL identify implicit tasks (things to practice, review, etc.).
11. WHEN extracting action items THEN the Transcription Analyzer SHALL prioritize by urgency if context is available.

##### Category 5: Questions Raised
12. WHEN identifying questions THEN the Transcription Analyzer SHALL list topics that need further exploration.
13. WHEN identifying questions THEN the Transcription Analyzer SHALL note any unanswered questions from the session.

##### Category 6: Session Quality
14. WHEN assessing session THEN the Transcription Analyzer SHALL rate actionability of advice (1-5 scale).
15. WHEN assessing session THEN the Transcription Analyzer SHALL note how many concrete examples were provided.

---

### Requirement 12: Generic Meeting Analysis

**Objective:** As a user who had a meeting or conversation, I want to extract key information, so that I can remember and act on what was discussed.

#### Acceptance Criteria

##### Category 1: Meeting Context
1. WHEN analyzing a meeting THEN the Transcription Analyzer SHALL identify participants (if detectable).
2. WHEN analyzing a meeting THEN the Transcription Analyzer SHALL identify the meeting purpose/topic.

##### Category 2: Summary
3. WHEN summarizing THEN the Transcription Analyzer SHALL provide a 3-5 sentence executive summary.
4. WHEN summarizing THEN the Transcription Analyzer SHALL highlight the most important points discussed.

##### Category 3: Decisions Made
5. WHEN extracting decisions THEN the Transcription Analyzer SHALL list all explicit decisions reached.
6. WHEN extracting decisions THEN the Transcription Analyzer SHALL note who made or owns each decision.

##### Category 4: Action Items
7. WHEN extracting action items THEN the Transcription Analyzer SHALL list all tasks assigned.
8. WHEN extracting action items THEN the Transcription Analyzer SHALL note the owner and deadline (if mentioned).

##### Category 5: Open Questions
9. WHEN identifying questions THEN the Transcription Analyzer SHALL list unresolved topics.
10. WHEN identifying questions THEN the Transcription Analyzer SHALL note topics that need follow-up.

##### Category 6: Key Quotes
11. WHEN extracting quotes THEN the Transcription Analyzer SHALL identify memorable or important statements.
12. WHEN extracting quotes THEN the Transcription Analyzer SHALL include speaker attribution and line number.

---

### Requirement 13: Topic Flow Analysis Mode

**Objective:** As a user with a long multi-person conversation, I want to analyze topic flow, deviations, and conversational dynamics, so that I can understand how effectively we stayed on track and who contributed to focus vs. tangents.

#### Acceptance Criteria

##### Input Handling
1. WHEN TopicFlowAnalysis mode is invoked THEN the Transcription Analyzer SHALL accept transcript file path.
2. WHEN loading transcript THEN the Transcription Analyzer SHALL detect speakers using layered approach: inline patterns first, header metadata second, interactive prompt third.
3. WHEN speaker detection is ambiguous THEN the Transcription Analyzer SHALL ask user to confirm participants list.
4. WHEN participants are provided externally (e.g., "Participants: Vishnu, Amy, Victor") THEN the Transcription Analyzer SHALL trust that metadata even if transcript shows different labels.

##### Token Optimization (Map-Reduce Architecture)
5. WHEN analyzing transcripts over 500 lines THEN the Transcription Analyzer SHALL use three-phase Map-Reduce architecture.
6. WHEN executing Phase 1 (Skeleton) THEN the Transcription Analyzer SHALL extract timestamps, speaker changes, and topic keywords using lightweight processing.
7. WHEN executing Phase 2 (Chunked Analysis) THEN the Transcription Analyzer SHALL split transcript into 15-20 minute chunks with 2-minute overlap.
8. WHEN executing Phase 2 THEN the Transcription Analyzer SHALL spawn parallel chunk agents via Task tool for concurrent processing.
9. WHEN executing Phase 3 (Synthesis) THEN the Transcription Analyzer SHALL merge chunk results into global topic tree and deduplicate overlapping sections.

##### Topic Hierarchy Detection
10. WHEN extracting topics THEN the Transcription Analyzer SHALL produce two-level hierarchy: Themes (major topics) containing Sub-topics (specific concepts).
11. WHEN detecting topic boundaries THEN the Transcription Analyzer SHALL scan for signals: "let's move on to", "next question", "okay so", "anyway", "going back to".
12. WHEN extracting themes from chunks THEN the Transcription Analyzer SHALL output: theme name, sub-topics list, key terms, time spent, confidence score.
13. WHEN merging themes across chunks THEN the Transcription Analyzer SHALL use fuzzy matching to cluster similar theme names and pick canonical name.

##### Deviation Classification
14. WHEN classifying deviations THEN the Transcription Analyzer SHALL detect three types: Unrelated Jump, Rabbit Hole, Depth Spiral.
15. WHEN detecting Unrelated Jump THEN the Transcription Analyzer SHALL flag when cosine similarity between adjacent topics < 0.3.
16. WHEN detecting Rabbit Hole THEN the Transcription Analyzer SHALL flag when topic exploration exceeds threshold AND conversation doesn't return to original topic.
17. WHEN detecting Depth Spiral THEN the Transcription Analyzer SHALL flag when time spent on sub-topic exceeds 2x average while other sub-topics are skipped.
18. WHEN a deviation is detected THEN the Transcription Analyzer SHALL assign severity: 🔴 High, 🟠 Medium, 🟡 Low, ⚪ Intentional.
19. WHEN conversation returns to original topic THEN the Transcription Analyzer SHALL track who initiated the return ("returned_by") with timestamp and trigger phrase.

##### Filler Word Analysis
20. WHEN analyzing filler words THEN the Transcription Analyzer SHALL detect four categories: classic fillers (um, uh), hedge words (like, basically), stalling phrases, confidence killers.
21. WHEN calculating filler density THEN the Transcription Analyzer SHALL compute: total_fillers / total_words per speaker per topic.
22. WHEN correlating fillers with topics THEN the Transcription Analyzer SHALL produce topic confidence score: high filler density = low confidence signal.
23. WHEN outputting filler analysis THEN the Transcription Analyzer SHALL show per-speaker profile with primary filler type, most/least confident topics.

##### Visualization Generation
24. WHEN generating visualizations THEN the Transcription Analyzer SHALL produce Mermaid Sankey diagram showing topic flow with time-spent as edge width.
25. WHEN generating visualizations THEN the Transcription Analyzer SHALL produce Mermaid Timeline diagram showing chronological topic progression with deviation markers.
26. WHEN generating Sankey THEN the Transcription Analyzer SHALL color tangent nodes differently from main topic nodes.
27. WHEN generating Timeline THEN the Transcription Analyzer SHALL use emoji indicators for confidence level per topic segment.

##### Output Format
28. WHEN generating output THEN the Transcription Analyzer SHALL produce markdown report with sections: Topic Hierarchy, Sankey Diagram, Timeline View, Deviation Report, Filler Word Analysis, Insights.
29. WHEN generating deviation report THEN the Transcription Analyzer SHALL include table with: type, from→to topic, duration, severity, returned_by.
30. WHEN generating insights THEN the Transcription Analyzer SHALL identify: knowledge gap signals (high filler topics), positive signals (confident topics), speaker patterns.
31. WHEN generating output THEN the Transcription Analyzer SHALL produce JSON export with full structured data for programmatic consumption.

---

## Non-Functional Requirements

### NFR-1: Performance
- Transcripts up to 2000 lines should complete analysis within reasonable time
- Subagent delegation should be transparent to user

### NFR-2: Reliability
- Invalid file paths should fail gracefully with helpful error messages
- Partial transcripts should still produce partial analysis

### NFR-3: Maintainability
- Skill should be self-contained in a single directory
- Prompts should be modular and editable

### NFR-4: Compatibility
- Should work with any Claude Code installation
- No MCP server dependencies required
