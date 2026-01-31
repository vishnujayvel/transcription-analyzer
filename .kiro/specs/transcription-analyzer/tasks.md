# Implementation Plan

## Overview

This plan implements the Transcription Analyzer v2.0 skill with **Supervisor Agent architecture** - a portable, dependency-free Claude Code skill for analyzing conversation transcripts. The supervisor first classifies session type (MockInterview.*, CoachingSession, GenericMeeting), then routes to specialized analysis workflows.

**Total Tasks:** 9 major tasks with 22 sub-tasks
**Version:** 2.0 (Supervisor Agent Architecture)

---

- [x] 1. Restructure skill directory for supervisor architecture
- [x] 1.1 Migrate existing structure to new prompts-based layout
  - Rename `references/` directory to `prompts/`
  - Move `analyzer-prompt.md` to `prompts/mock_interview_analyzer.md`
  - Move `confidence-scoring.md` to `prompts/confidence_scorer.md`
  - Update SKILL.md references to point to new locations
  - _Requirements: NFR-3, NFR-4_

- [x] 1.2 Create new analyzer prompt files for each session type
  - Create placeholder `prompts/supervisor_classifier.md`
  - Create placeholder `prompts/coaching_analyzer.md`
  - Create placeholder `prompts/meeting_analyzer.md`
  - Create `examples/` directory for test transcripts
  - _Requirements: R0, R11, R12, NFR-3_

---

- [x] 2. Implement Supervisor Agent session classification
- [x] 2.1 Build session type signal detection in supervisor_classifier.md
  - Define signal patterns for MockInterview.SystemDesign: "design a system", "scalability", "database", "load balancer", "API design"
  - Define signal patterns for MockInterview.Coding: "write a function", "time complexity", "algorithm", "test cases", "edge cases"
  - Define signal patterns for MockInterview.Behavioral: "tell me about a time", "STAR", "leadership", "conflict", "situation"
  - Define signal patterns for CoachingSession: "advice", "tips", "here's what I'd recommend", "you should try", "feedback on your", "let me coach you"
  - Specify GenericMeeting as default fallback when no strong signals
  - _Requirements: R0.2_

- [x] 2.2 Implement confidence-based classification algorithm
  - Scan transcript for all defined signal patterns
  - Count signal occurrences and calculate weighted scores per session type
  - Normalize scores to percentages (0-100)
  - HIGH confidence if highest score >= 70%
  - MEDIUM confidence if highest score 50-69% (suggest user confirmation)
  - Require user confirmation if highest score < 50% or equal top scores
  - Default to GenericMeeting if no confirmation and low confidence
  - _Requirements: R0.1, R0.3, R0.5_

- [x] 2.3 Define classification output contract
  - Output session type with discriminated union format: MockInterview.{subtype}, CoachingSession, or GenericMeeting
  - Include confidence level (HIGH/MEDIUM/LOW) and confidence score (0-100)
  - Include evidence array with detected phrases, line numbers, and signal weights
  - Include requiresUserConfirmation boolean flag
  - _Requirements: R0.3, R0.6_

---

- [x] 3. Update SKILL.md router with supervisor orchestration
- [x] 3.1 Refactor input handling to integrate supervisor
  - Preserve existing argument parsing and file path prompt flow
  - After file validation, invoke supervisor classification before any analysis
  - Pass transcript content to supervisor for session type detection
  - _Requirements: R1.1, R1.2, R1.3, R1.4, R1.5, R1.6_

- [x] 3.2 Implement session type routing logic
  - If supervisor classifies as MockInterview.*, route to mock interview analysis workflow
  - If supervisor classifies as CoachingSession, route to coaching analysis workflow
  - If supervisor classifies as GenericMeeting, route to meeting analysis workflow
  - If requiresUserConfirmation is true, use AskUserQuestion to let user choose type
  - Pass classified type and confidence to child analyzer
  - _Requirements: R0.4, R0.5, R0.6_

- [x] 3.3 Add session type badge to all outputs
  - Display session type badge at top of every report
  - Format: "**Session Type:** MockInterview.SystemDesign [Confidence: HIGH 92%]"
  - Format: "**Session Type:** CoachingSession [Confidence: MEDIUM 68%]"
  - Format: "**Session Type:** GenericMeeting [Confidence: LOW - Default]"
  - _Requirements: R0.3, R7.1_

---

- [x] 4. Enhance mock interview analyzer with subtype awareness
- [x] 4.1 Refactor mock_interview_analyzer.md for subtype context
  - Accept MockInterviewSubtype parameter (SystemDesign, Coding, Behavioral, Unknown)
  - Adjust analysis emphasis based on subtype
  - SystemDesign: emphasize architecture decisions, scalability trade-offs, component design
  - Coding: emphasize algorithm choice, complexity analysis, edge case handling
  - Behavioral: emphasize STAR format, leadership signals, communication quality
  - Preserve all 10-category extraction for all subtypes
  - _Requirements: R3.1, R3.2, R3.3, R3.4, R3.5, R3.6_

- [x] 4.2 Preserve existing interview start detection
  - Keep trigger phrase detection: "go design", "let's get started", "the problem is", etc.
  - Record interview start line number in output
  - Flag LOW confidence on timing if no trigger found
  - _Requirements: R2.1, R2.2, R2.3, R2.4_

- [x] 4.3 Maintain anti-hallucination protocol in mock analyzer
  - Enforce confidence levels: HIGH (90%+), MEDIUM (60-89%), LOW (30-59%), NOT_FOUND (0%)
  - Require evidence citation (line number or quote) for every insight
  - Tag [EXPLICIT] vs [INFERRED] for all claims
  - Calculate aggregate confidence using weighted average
  - _Requirements: R4.1-R4.8_

---

- [x] 5. Implement coaching session analyzer
- [x] 5.1 Build coaching_analyzer.md with 6-category extraction
  - Category 1 - Session Context: identify coach/mentor and mentee names, extract session topics discussed
  - Category 2 - Key Advice/Tips: list all explicit recommendations, categorize by domain (communication, technical, career, behavioral, other), include direct quotes with line numbers
  - Category 3 - Scripts/Patterns: extract reusable phrases or frameworks taught, format as quotable ready-to-use text, note context for when to use each script
  - Category 4 - Action Items: identify explicit tasks assigned by coach, identify implicit tasks (practice, review), prioritize by urgency
  - Category 5 - Questions Raised: list topics needing further exploration, note unanswered questions
  - Category 6 - Session Quality: rate actionability (1-5 scale), count concrete examples provided
  - _Requirements: R11.1-R11.15_

- [x] 5.2 Apply anti-hallucination protocol to coaching analyzer
  - Require confidence level for every insight
  - Require evidence (line number, quote) for all advice and scripts
  - Handle NOT_FOUND cases explicitly
  - Include confidence summary at end
  - _Requirements: R4.1-R4.8_

- [x] 5.3 Define coaching analysis JSON output schema
  - Include metadata with sessionType="CoachingSession", totalLines, timestamp
  - Include sessionContext with coach, mentee, topics, confidence
  - Include arrays for keyAdvice, scriptsAndPatterns, actionItems, questionsRaised
  - Include sessionQuality with actionabilityScore, examplesCount, confidence
  - Include confidenceSummary
  - _Requirements: R7.5-R7.8, R11.1-R11.15_

---

- [x] 6. Implement generic meeting analyzer
- [x] 6.1 Build meeting_analyzer.md with 6-category extraction
  - Category 1 - Meeting Context: identify participants (if detectable), identify meeting purpose/topic
  - Category 2 - Summary: provide 3-5 sentence executive summary, highlight most important points discussed
  - Category 3 - Decisions Made: list all explicit decisions reached, note owner for each decision
  - Category 4 - Action Items: list all tasks assigned, note owner and deadline if mentioned
  - Category 5 - Open Questions: list unresolved topics, note topics needing follow-up
  - Category 6 - Key Quotes: identify memorable/important statements, include speaker attribution and line number
  - _Requirements: R12.1-R12.12_

- [x] 6.2 Apply anti-hallucination protocol to meeting analyzer
  - Require confidence level for every insight
  - Require evidence for decisions, action items, and quotes
  - Handle NOT_FOUND cases explicitly
  - Include confidence summary
  - _Requirements: R4.1-R4.8_

- [x] 6.3 Define meeting analysis JSON output schema
  - Include metadata with sessionType="GenericMeeting", totalLines, timestamp
  - Include meetingContext with participants, purpose, confidence
  - Include summary with executiveSummary, keyPoints, confidence
  - Include arrays for decisions, actionItems, openQuestions, keyQuotes
  - Include confidenceSummary
  - _Requirements: R7.5-R7.8, R12.1-R12.12_

---

- [x] 7. Update output formatting for all session types
- [x] 7.1 Create coaching session markdown report template
  - Header with file name, session type badge, coach/mentee, topics
  - Section 1: Key Advice & Tips (categorized by domain with quotes)
  - Section 2: Scripts & Patterns (quotable text with usage context)
  - Section 3: Action Items (explicit/implicit with urgency)
  - Section 4: Questions Raised (topics needing exploration)
  - Section 5: Session Quality (actionability score, examples count)
  - Confidence summary at end
  - Use emoji indicators and ADHD-friendly formatting
  - _Requirements: R7.1-R7.4, R10.1-R10.5, R11.1-R11.15_

- [x] 7.2 Create generic meeting markdown report template
  - Header with file name, session type badge, participants, purpose
  - Section 1: Executive Summary (3-5 sentences)
  - Section 2: Key Decisions (with owners)
  - Section 3: Action Items (with owners and deadlines)
  - Section 4: Open Questions (topics needing follow-up)
  - Section 5: Key Quotes (with speaker attribution)
  - Confidence summary at end
  - Use emoji indicators and ADHD-friendly formatting
  - _Requirements: R7.1-R7.4, R10.1-R10.5, R12.1-R12.12_

- [x] 7.3 Ensure mock interview report preserves existing formatting
  - Maintain positives BEFORE mistakes ordering
  - Preserve all 10 categories with confidence indicators
  - Preserve emoji severity indicators
  - Add session type badge header
  - _Requirements: R5.1-R5.29, R7.1-R7.4, R10.1-R10.5_

---

- [x] 8. Integrate subagent delegation for all session types
- [x] 8.1 Update delegation logic for session-type-aware routing
  - Check line count threshold (>500 lines requires delegation)
  - Select appropriate analyzer prompt based on session type
  - MockInterview.* → mock_interview_analyzer.md
  - CoachingSession → coaching_analyzer.md
  - GenericMeeting → meeting_analyzer.md
  - Pass session type and confidence to subagent
  - _Requirements: R6.1, R6.2, R0.4_

- [x] 8.2 Implement subagent response validation per session type
  - Validate JSON structure matches expected schema for each session type
  - Handle malformed responses with repair attempt
  - Fall back to direct analysis with truncation warning if subagent fails
  - Preserve anti-hallucination requirements in all subagent prompts
  - _Requirements: R6.3, R6.4, R6.5_

---

- [x] 9. Create test transcripts and verify portability
- [x] 9.1 Create sample coaching session transcript
  - Create `examples/sample_coaching_session.md` with realistic mentoring content
  - Include coaching signal phrases: "advice", "tips", "I'd recommend", etc.
  - Include identifiable coach and mentee speakers
  - Include explicit advice, scripts/frameworks, and action items
  - Keep under 500 lines for direct testing
  - _Requirements: R11.1-R11.15, NFR-1_

- [x] 9.2 Create sample generic meeting transcript
  - Create `examples/sample_meeting.md` with realistic meeting content
  - Include multiple participants
  - Include explicit decisions and action items
  - Include open questions and memorable quotes
  - Keep under 500 lines for direct testing
  - _Requirements: R12.1-R12.12, NFR-1_

- [x] 9.3 Verify portability constraints across all files
  - Audit all created/modified files for prohibited references
  - Ensure no Mem0, Obsidian, or personal path references exist
  - Confirm only allowed tools used: Read, AskUserQuestion, Task
  - Verify skill works with just transcript file path
  - Test with each session type to confirm routing works
  - _Requirements: R8.1-R8.6, NFR-4_

---

---

## v2.1 Tasks: Topic Flow Analysis Mode

- [x] 10. Add TopicFlowAnalysis to session type routing
- [x] 10.1 Update SKILL.md with TopicFlowAnalysis triggers and routing
  - Add trigger phrases: "analyze topic flow", "topic flow analysis", "how did we deviate?", "show me our tangents"
  - Add TopicFlowAnalysis to session type detection signals
  - Route to topic_flow_orchestrator.md when detected
  - _Requirements: R13.1, R13.2, R13.3, R13.4_

- [ ] 10.2 Update supervisor_classifier.md with TopicFlowAnalysis signals
  - Add signal patterns for long discussions: multiple speakers, extended duration markers
  - TopicFlowAnalysis should be explicitly requested (not auto-detected)
  - _Requirements: R13.1_

---

- [x] 11. Implement Phase 1: Skeleton Extraction
- [x] 11.1 Create prompts/topic_flow_orchestrator.md
  - Define three-phase orchestration flow
  - Phase 1: Lightweight extraction of timestamps, speakers, keywords
  - Implement speaker detection: inline patterns → header metadata → ask user
  - Detect topic boundary signals: "let's move on", "next question", "anyway", "going back to"
  - _Requirements: R13.2, R13.3, R13.4, R13.5, R13.6, R13.10, R13.11_

- [ ] 11.2 Implement chunking logic
  - Split transcript into 15-20 minute chunks
  - Add 2-minute overlap between chunks
  - Calculate chunk boundaries based on timestamps
  - _Requirements: R13.7, R13.8_

---

- [x] 12. Implement Phase 2: Parallel Chunk Analysis
- [x] 12.1 Create prompts/topic_extractor.md
  - Extract two-level hierarchy: themes → sub-topics
  - Output per chunk: theme name, sub-topics, key terms, time spent, confidence
  - Include entry/exit topics for stitching
  - _Requirements: R13.10, R13.12_

- [x] 12.2 Create prompts/deviation_classifier.md
  - Detect unrelated jumps (semantic similarity < 0.3)
  - Detect rabbit holes (exploration without return)
  - Detect depth spirals (time > 2x average)
  - Track return events with speaker attribution
  - Assign severity: high, medium, low, intentional
  - _Requirements: R13.14, R13.15, R13.16, R13.17, R13.18, R13.19_

- [x] 12.3 Create prompts/filler_word_analyzer.md
  - Detect four filler categories: classic, hedge, stalling, confidence killers
  - Calculate filler density per speaker per topic
  - Compute topic confidence scores from filler density
  - _Requirements: R13.20, R13.21, R13.22, R13.23_

- [ ] 12.4 Implement parallel agent spawning
  - Spawn chunk agents via Task tool in single message (true parallelism)
  - Use Haiku model for chunk agents (speed + cost optimization)
  - Collect results from all agents
  - _Requirements: R13.8_

---

- [x] 13. Implement Phase 3: Synthesis
- [x] 13.1 Create prompts/topic_flow_synthesizer.md
  - Merge chunk results into global topic tree
  - Deduplicate overlapping sections (2-min overlap handling)
  - Fuzzy match themes across chunks, pick canonical names
  - Cross-chunk pattern detection: "returned to X multiple times"
  - _Requirements: R13.9, R13.13_

- [ ] 13.2 Implement Mermaid Sankey generation
  - Generate sankey-beta diagram from topic flow
  - Edge width = time spent on topic
  - Color tangent nodes differently
  - _Requirements: R13.24, R13.26_

- [ ] 13.3 Implement Mermaid Timeline generation
  - Generate timeline diagram with chronological flow
  - Mark deviation events with warning indicators
  - Color by confidence level (emoji indicators)
  - _Requirements: R13.25, R13.27_

- [ ] 13.4 Implement insight generation
  - Knowledge gap signals (high filler topics)
  - Positive signals (confident topics)
  - Speaker patterns
  - Learning signals (filler decrease over time)
  - _Requirements: R13.30_

---

- [ ] 14. Create output formatting and JSON export
- [ ] 14.1 Create TopicFlowAnalysis markdown report template
  - Section 1: Topic Hierarchy (tree with time and confidence)
  - Section 2: Topic Flow Sankey
  - Section 3: Timeline View
  - Section 4: Deviation Report (table)
  - Section 5: Filler Word Analysis (heatmap + profiles)
  - Section 6: Insights & Recommendations
  - _Requirements: R13.28, R13.29_

- [ ] 14.2 Define JSON export schema
  - Full structured data for programmatic consumption
  - Match TypeScript interfaces from design doc
  - _Requirements: R13.31_

---

- [ ] 15. Test with real transcript
- [ ] 15.1 Test with Database Party transcript
  - File: /Users/vishnu/Documents/obsibrain/obsibrain-vault/5-notes/Database party with Amy and Victor - transcription.md
  - Verify speaker detection (Vishnu, Amy/Them, Victor)
  - Verify topic extraction accuracy
  - Verify deviation detection
  - Verify filler word analysis
  - Verify visualization generation
  - _Requirements: All R13.*_

- [ ] 15.2 Generate sample output
  - Create examples/sample_topic_flow_output.md
  - Document expected structure for future reference
  - _Requirements: NFR-3_

---

## Requirements Coverage Matrix

| Requirement | Task(s) |
|-------------|---------|
| R0: Supervisor Agent Architecture | 2.1, 2.2, 2.3, 3.1, 3.2, 3.3 |
| R1: Transcript Input | 3.1 |
| R2: Interview Start Detection | 4.2 |
| R3: Mock Interview Type Detection | 2.1, 4.1 |
| R4: Anti-Hallucination Protocol | 4.3, 5.2, 6.2 |
| R5: 10-Category Framework | 4.1, 7.3 |
| R6: Subagent Delegation | 8.1, 8.2 |
| R7: Output Formats | 3.3, 5.3, 6.3, 7.1, 7.2, 7.3 |
| R8: Portability Constraints | 9.3 |
| R9: Diagram Analysis | (preserved from v1, no changes) |
| R10: ADHD-Friendly Design | 7.1, 7.2, 7.3 |
| R11: Coaching Session Analysis | 5.1, 5.2, 5.3, 7.1, 9.1 |
| R12: Generic Meeting Analysis | 6.1, 6.2, 6.3, 7.2, 9.2 |
| NFR-1: Performance | 8.1, 9.1, 9.2 |
| NFR-2: Reliability | 8.2 |
| NFR-3: Maintainability | 1.1, 1.2 |
| NFR-4: Compatibility | 1.1, 9.3 |
| R13: Topic Flow Analysis | 10.1, 10.2, 11.1, 11.2, 12.1, 12.2, 12.3, 12.4, 13.1, 13.2, 13.3, 13.4, 14.1, 14.2, 15.1, 15.2 |

---

## Implementation Order

Execute tasks in numerical order. Each task builds on previous outputs:

1. **Restructure** (Tasks 1.x): Migrate to prompts-based directory layout
2. **Supervisor** (Tasks 2.x): Build session type classification
3. **Router** (Tasks 3.x): Update SKILL.md with supervisor orchestration
4. **Mock Interview** (Tasks 4.x): Enhance with subtype awareness
5. **Coaching** (Tasks 5.x): New coaching session analyzer
6. **Meeting** (Tasks 6.x): New generic meeting analyzer
7. **Output** (Tasks 7.x): Format reports for all session types
8. **Delegation** (Tasks 8.x): Integrate subagent routing
9. **Testing** (Tasks 9.x): Create test transcripts and verify
