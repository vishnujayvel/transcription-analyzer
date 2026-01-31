# Technical Design Document

## Overview

**Purpose**: The Transcription Analyzer delivers structured, confidence-scored feedback on conversation transcripts using a **Supervisor Agent architecture**. It classifies session types first, then routes to specialized analysis workflows for mock interviews, coaching sessions, or generic meetings.

**Users**: Anyone who has conversation transcripts and wants actionable insights - interview candidates analyzing mock interviews, mentees reviewing coaching sessions, or professionals summarizing meetings - without requiring external service dependencies.

**Impact**: Extends the proven mock interview review workflow from daily-copilot into a multi-purpose transcript analyzer that intelligently adapts its analysis based on session type.

### Goals
- Implement Supervisor Agent that classifies session type before detailed analysis
- Support strongly-typed session types: MockInterview.*, CoachingSession, GenericMeeting
- Provide specialized analysis workflows for each session type
- Maintain comprehensive confidence scoring and anti-hallucination protocols
- Support large transcripts (500+ lines) through intelligent subagent delegation
- Maintain zero external dependencies (no Mem0, Obsidian, or other services)

### Non-Goals
- Integration with memory services or note-taking applications
- Trend analysis across multiple sessions (requires persistence)
- Real-time transcript processing during conversations
- Automatic study plan or task creation in external systems

### Goals (v2.1 - TopicFlowAnalysis)
- Add TopicFlowAnalysis mode for long multi-person conversations
- Implement three-phase Map-Reduce architecture for token optimization
- Extract hierarchical topics (themes → sub-topics) with time tracking
- Detect three deviation types: unrelated jumps, rabbit holes, depth spirals
- Track who causes deviations and who brings conversation back on track
- Analyze filler words as confidence signals per topic
- Generate Mermaid visualizations (Sankey + Timeline)

---

## Architecture

### High-Level Architecture

```mermaid
graph TB
    User[User] --> |"invoke skill"| SKILL[SKILL.md Router]
    SKILL --> |"no args"| AUQ[AskUserQuestion]
    AUQ --> |"file path"| SKILL
    SKILL --> |"args provided"| Validate[File Validation]
    Validate --> |"valid"| Supervisor[Supervisor Agent]
    Validate --> |"invalid"| Error[Error Message]

    Supervisor --> |"classify"| TypeDetect[Session Type Detection]
    TypeDetect --> |"MockInterview.*"| MockRouter[Mock Interview Router]
    TypeDetect --> |"CoachingSession"| CoachAnalyzer[Coaching Analyzer]
    TypeDetect --> |"GenericMeeting"| MeetingAnalyzer[Meeting Analyzer]
    TypeDetect --> |"LOW confidence"| UserConfirm[Ask User to Confirm Type]
    UserConfirm --> MockRouter
    UserConfirm --> CoachAnalyzer
    UserConfirm --> MeetingAnalyzer

    MockRouter --> |"SystemDesign"| SDAnalyzer[System Design Analyzer]
    MockRouter --> |"Coding"| CodeAnalyzer[Coding Analyzer]
    MockRouter --> |"Behavioral"| BehavAnalyzer[Behavioral Analyzer]

    SDAnalyzer --> TenCategory[10-Category Analysis]
    CodeAnalyzer --> TenCategory
    BehavAnalyzer --> TenCategory

    TenCategory --> Formatter[Output Formatter]
    CoachAnalyzer --> Formatter
    MeetingAnalyzer --> Formatter

    Formatter --> MDReport[Markdown Report]
    Formatter --> JSONSummary[JSON Summary]
```

**Key Design Decisions**:

1. **Supervisor Agent Pattern**
   - **Decision**: Implement a Supervisor Agent that classifies session type before routing to specialized analyzers
   - **Context**: KDR's mentoring session revealed that mock interview analysis doesn't fit coaching sessions
   - **Alternatives**: Single analyzer with conditional logic; User-specified type upfront
   - **Selected Approach**: Supervisor detects session type from content signals, routes to appropriate workflow
   - **Rationale**: Reduces user friction (no upfront selection), enables specialized analysis per type
   - **Trade-offs**: Additional classification step adds latency; potential misclassification requires user confirmation

2. **Strongly-Typed Session Types**
   - **Decision**: Define explicit session types as a discriminated union rather than free-form strings
   - **Context**: Different session types need fundamentally different analysis categories
   - **Alternatives**: Single analysis framework with optional sections; Free-form type strings
   - **Selected Approach**: Typed enum with MockInterview.*, CoachingSession, GenericMeeting
   - **Rationale**: Type safety enables correct workflow routing; clear contracts per type
   - **Trade-offs**: Less flexible for unknown session types; requires GenericMeeting fallback

3. **Workflow Specialization**
   - **Decision**: Each session type has its own analysis workflow with distinct output categories
   - **Context**: Mock interviews need mistakes/gaps; coaching needs tips/scripts; meetings need decisions/action items
   - **Alternatives**: Universal category framework; Configurable category selection
   - **Selected Approach**: Dedicated analyzer prompts per session type with type-specific categories
   - **Rationale**: Better signal extraction; more actionable outputs; clearer structure
   - **Trade-offs**: More prompts to maintain; some category overlap across types

### Technology Alignment

This skill aligns with the existing Claude Code skill architecture:
- Uses standard SKILL.md as entry point
- Leverages built-in tools: Read, AskUserQuestion, Task
- No MCP server dependencies
- Follows prompt-centric pattern from daily-copilot
- Subagent delegation via Task tool for large transcripts

---

## System Flows

### Supervisor Agent Classification Flow

```mermaid
sequenceDiagram
    participant U as User
    participant S as SKILL.md
    participant Sup as Supervisor Agent
    participant R as Read Tool

    U->>S: /transcription-analyzer [path]
    S->>R: Read transcript file
    R-->>S: Transcript content
    S->>Sup: Classify session type

    Sup->>Sup: Scan for MockInterview signals
    Sup->>Sup: Scan for CoachingSession signals
    Sup->>Sup: Scan for GenericMeeting signals
    Sup->>Sup: Calculate confidence scores

    alt HIGH confidence classification
        Sup-->>S: SessionType + confidence
        S->>S: Route to appropriate analyzer
    else LOW confidence
        Sup-->>S: Uncertain classification
        S->>U: AskUserQuestion to confirm type
        U-->>S: Confirmed type
        S->>S: Route to confirmed analyzer
    end
```

### Session Type Detection Flow

```mermaid
flowchart TD
    Start[Read Transcript] --> Scan[Scan All Signal Types]

    Scan --> MockSD{MockInterview.SD Signals?}
    Scan --> MockCode{MockInterview.Coding Signals?}
    Scan --> MockBehav{MockInterview.Behavioral Signals?}
    Scan --> Coach{CoachingSession Signals?}

    MockSD -->|"design a system", "scalability", "load balancer"| ScoreSD[Score: MockInterview.SystemDesign]
    MockCode -->|"write a function", "time complexity", "algorithm"| ScoreCode[Score: MockInterview.Coding]
    MockBehav -->|"tell me about a time", "STAR", "leadership"| ScoreBehav[Score: MockInterview.Behavioral]
    Coach -->|"advice", "tips", "recommend", "coach"| ScoreCoach[Score: CoachingSession]

    ScoreSD --> Compare[Compare Confidence Scores]
    ScoreCode --> Compare
    ScoreBehav --> Compare
    ScoreCoach --> Compare

    Compare --> Winner{Highest Score > 70%?}
    Winner -->|Yes| Route[Route to Winner Workflow]
    Winner -->|No| AskUser[Ask User to Confirm]
    AskUser --> Route

    Compare --> NoSignals{No Strong Signals?}
    NoSignals -->|Yes| Generic[Default: GenericMeeting]
    Generic --> Route
```

### Mock Interview Analysis Flow

```mermaid
sequenceDiagram
    participant S as SKILL.md
    participant T as Task Tool
    participant A as Mock Interview Analyzer

    S->>S: Determine line count

    alt Lines > 500
        S->>T: Spawn Explore subagent
        T->>A: mock_interview_analyzer.md + transcript + subtype
        A->>A: Detect interview start
        A->>A: Extract 10 categories
        A-->>T: JSON analysis results
        T-->>S: Subagent results
    else Lines <= 500
        S->>S: Direct 10-category extraction
    end

    S->>S: Apply confidence scoring
    S->>S: Format markdown report
    S->>S: Generate JSON summary
```

### Coaching Session Analysis Flow

```mermaid
sequenceDiagram
    participant S as SKILL.md
    participant T as Task Tool
    participant C as Coaching Analyzer

    S->>T: Spawn coaching analyzer subagent
    T->>C: coaching_analyzer.md + transcript

    C->>C: Identify coach and mentee
    C->>C: Extract key advice/tips
    C->>C: Identify scripts/patterns
    C->>C: Extract action items
    C->>C: Note questions raised
    C->>C: Assess session quality

    C-->>T: CoachingAnalysisJSON
    T-->>S: Analysis results
    S->>S: Format coaching report
```

---

## Requirements Traceability

| Requirement | Summary | Components | Prompts |
|-------------|---------|------------|---------|
| R0 | Supervisor Agent Architecture | SKILL.md supervisor section | supervisor_classifier.md |
| R1 | Transcript input | SKILL.md input section | - |
| R2 | Interview start detection | mock_interview_analyzer.md | - |
| R3 | Mock interview type detection | supervisor_classifier.md | - |
| R4 | Anti-hallucination protocol | All prompts | confidence_scorer.md |
| R5 | 10-category framework | mock_interview_analyzer.md | - |
| R6 | Subagent delegation | SKILL.md delegation logic | All analyzer prompts |
| R7 | Output formats | SKILL.md output section | - |
| R8 | Portability constraints | All files (no external refs) | - |
| R9 | Diagram analysis | mock_interview_analyzer.md | - |
| R10 | ADHD-friendly design | SKILL.md output formatting | - |
| R11 | Coaching session analysis | coaching_analyzer.md | - |
| R12 | Generic meeting analysis | meeting_analyzer.md | - |

---

## Components and Interfaces

### Supervisor Layer

#### Supervisor Agent (supervisor_classifier.md)

**Responsibility & Boundaries**
- **Primary Responsibility**: Classify transcript session type and route to appropriate analyzer
- **Domain Boundary**: Session type classification only; no detailed analysis
- **Data Ownership**: Owns session type classification decision and confidence

**Dependencies**
- **Inbound**: SKILL.md after file validation
- **Outbound**: Routes to MockInterviewAnalyzer, CoachingAnalyzer, or MeetingAnalyzer
- **External**: None

**Contract Definition**

```typescript
// Session Type Discriminated Union
type SessionType =
  | { type: "MockInterview"; subtype: MockInterviewSubtype }
  | { type: "CoachingSession" }
  | { type: "GenericMeeting" };

type MockInterviewSubtype = "SystemDesign" | "Coding" | "Behavioral" | "Unknown";

interface ClassificationResult {
  sessionType: SessionType;
  confidence: ConfidenceLevel;
  confidenceScore: number;  // 0-100
  evidence: ClassificationEvidence[];
  requiresUserConfirmation: boolean;
}

interface ClassificationEvidence {
  signalType: string;       // e.g., "MockInterview.SystemDesign"
  phrase: string;           // Detected phrase
  lineNumber: number;
  weight: number;           // Signal strength 0-1
}

// Signal Detection Patterns
interface SignalPatterns {
  "MockInterview.SystemDesign": string[];  // ["design a system", "scalability", ...]
  "MockInterview.Coding": string[];        // ["write a function", "time complexity", ...]
  "MockInterview.Behavioral": string[];    // ["tell me about a time", "STAR", ...]
  "CoachingSession": string[];             // ["advice", "tips", "recommend", ...]
}
```

**Classification Algorithm**:
1. Scan transcript for all signal patterns
2. Calculate weighted score per session type
3. Normalize scores to percentages
4. If highest score >= 70%: classify with HIGH confidence
5. If highest score 50-69%: classify with MEDIUM confidence, suggest confirmation
6. If highest score < 50%: require user confirmation or default to GenericMeeting

---

### Skill Router

#### SKILL.md

**Responsibility & Boundaries**
- **Primary Responsibility**: Route user requests, coordinate supervisor and analyzers, format outputs
- **Domain Boundary**: Skill entry point and orchestration
- **Data Ownership**: None (stateless)

**Dependencies**
- **Inbound**: User invocation via `/transcription-analyzer`
- **Outbound**: Read tool, AskUserQuestion tool, Task tool, Supervisor Agent, Analyzers
- **External**: None

**Contract Definition**

```typescript
interface SkillInput {
  args?: string;  // Optional file path argument
}

interface SkillOutput {
  sessionType: SessionType;
  markdownReport: string;
  jsonSummary: AnalysisJSON | CoachingAnalysisJSON | MeetingAnalysisJSON;
}
```

**Workflow Sections**:
1. **Input Handling**: Parse args or prompt for file path
2. **File Validation**: Read file, check existence, count lines
3. **Session Classification**: Invoke Supervisor Agent
4. **User Confirmation** (if needed): Ask user to confirm ambiguous classification
5. **Analysis Routing**: Delegate to appropriate analyzer based on session type
6. **Output Formatting**: Generate type-specific markdown and JSON outputs

---

### Mock Interview Analysis Engine

#### prompts/mock_interview_analyzer.md

**Responsibility & Boundaries**
- **Primary Responsibility**: Extract all 10 analytics categories from mock interview transcript
- **Domain Boundary**: Mock interview analysis with anti-hallucination enforcement
- **Data Ownership**: Owns the 10-category extraction schema for mock interviews

**Dependencies**
- **Inbound**: SKILL.md via Task tool delegation
- **Outbound**: None (returns JSON)
- **External**: None

**Contract Definition**

```typescript
interface MockInterviewInput {
  transcript: string;
  subtype: MockInterviewSubtype;
  startLine?: number;
}

interface MockInterviewAnalysisJSON {
  metadata: {
    sessionType: "MockInterview";
    subtype: MockInterviewSubtype;
    interviewStartLine: number | null;
    totalLines: number;
    analysisTimestamp: string;
  };
  scorecard: ScorecardCategory;
  timeBreakdown: TimeCategory;
  communicationSignals: CommunicationCategory;
  mistakes: MistakeCategory[];
  positives: PositiveCategory[];
  knowledgeGaps: GapCategory[];
  behavioralAssessment: BehavioralCategory;
  factualClaims: ClaimCategory[];
  interviewerQuality: InterviewerCategory;
  actionItems: ActionItem[];
  confidenceSummary: ConfidenceSummary;
}
```

---

### Coaching Session Analysis Engine

#### prompts/coaching_analyzer.md

**Responsibility & Boundaries**
- **Primary Responsibility**: Extract actionable advice, scripts, and patterns from coaching sessions
- **Domain Boundary**: Coaching/mentoring session analysis
- **Data Ownership**: Owns the 6-category coaching extraction schema

**Dependencies**
- **Inbound**: SKILL.md via Task tool delegation
- **Outbound**: None (returns JSON)
- **External**: None

**Contract Definition**

```typescript
interface CoachingInput {
  transcript: string;
}

interface CoachingAnalysisJSON {
  metadata: {
    sessionType: "CoachingSession";
    totalLines: number;
    analysisTimestamp: string;
  };
  sessionContext: {
    coach: string | null;
    mentee: string | null;
    topics: string[];
    confidence: ConfidenceLevel;
  };
  keyAdvice: AdviceItem[];
  scriptsAndPatterns: ScriptItem[];
  actionItems: CoachingActionItem[];
  questionsRaised: QuestionItem[];
  sessionQuality: {
    actionabilityScore: number;  // 1-5
    examplesCount: number;
    confidence: ConfidenceLevel;
  };
  confidenceSummary: ConfidenceSummary;
}

interface AdviceItem {
  advice: string;
  domain: "communication" | "technical" | "career" | "behavioral" | "other";
  quote: string;
  lineNumber: number;
  confidence: ConfidenceLevel;
}

interface ScriptItem {
  name: string;           // e.g., "3-Minute Check-In"
  script: string;         // The actual script text
  context: string;        // When to use it
  lineNumber: number;
  confidence: ConfidenceLevel;
}

interface CoachingActionItem {
  action: string;
  type: "explicit" | "implicit";
  urgency: "high" | "medium" | "low" | "unknown";
  confidence: ConfidenceLevel;
  evidence: Evidence[];
}

interface QuestionItem {
  question: string;
  status: "answered" | "unanswered" | "needs-exploration";
  confidence: ConfidenceLevel;
}
```

---

### Generic Meeting Analysis Engine

#### prompts/meeting_analyzer.md

**Responsibility & Boundaries**
- **Primary Responsibility**: Extract summary, decisions, and action items from any meeting
- **Domain Boundary**: General meeting/conversation analysis
- **Data Ownership**: Owns the 6-category meeting extraction schema

**Dependencies**
- **Inbound**: SKILL.md via Task tool delegation
- **Outbound**: None (returns JSON)
- **External**: None

**Contract Definition**

```typescript
interface MeetingInput {
  transcript: string;
}

interface MeetingAnalysisJSON {
  metadata: {
    sessionType: "GenericMeeting";
    totalLines: number;
    analysisTimestamp: string;
  };
  meetingContext: {
    participants: string[];
    purpose: string;
    confidence: ConfidenceLevel;
  };
  summary: {
    executiveSummary: string;  // 3-5 sentences
    keyPoints: string[];
    confidence: ConfidenceLevel;
  };
  decisions: DecisionItem[];
  actionItems: MeetingActionItem[];
  openQuestions: OpenQuestionItem[];
  keyQuotes: QuoteItem[];
  confidenceSummary: ConfidenceSummary;
}

interface DecisionItem {
  decision: string;
  owner: string | null;
  confidence: ConfidenceLevel;
  evidence: Evidence[];
}

interface MeetingActionItem {
  action: string;
  owner: string | null;
  deadline: string | null;
  confidence: ConfidenceLevel;
  evidence: Evidence[];
}

interface OpenQuestionItem {
  question: string;
  context: string;
  needsFollowUp: boolean;
  confidence: ConfidenceLevel;
}

interface QuoteItem {
  quote: string;
  speaker: string | null;
  lineNumber: number;
  significance: string;
}
```

---

### Confidence Scoring Engine

#### prompts/confidence_scorer.md

**Responsibility & Boundaries**
- **Primary Responsibility**: Define and enforce confidence scoring methodology across all analyzers
- **Domain Boundary**: Anti-hallucination protocol implementation

**Confidence Level Definitions**:

| Level | Score Range | Criteria | Evidence Required |
|-------|-------------|----------|-------------------|
| HIGH | 90-100% | Direct quote, explicit statement | Line number + exact quote |
| MEDIUM | 60-89% | Contextual inference, multiple signals | Description of supporting signals |
| LOW | 30-59% | Single weak signal, ambiguous | Note ambiguity source |
| NOT_FOUND | 0% | No evidence in transcript | "Not found in transcript" |

**Common Interface**:

```typescript
interface ConfidenceLevel = "HIGH" | "MEDIUM" | "LOW" | "NOT_FOUND";

interface Evidence {
  type: "EXPLICIT" | "INFERRED";
  content: string;
  lineNumber?: number;
}

interface ConfidenceSummary {
  overall: ConfidenceLevel;
  overallScore: number;
  byCategory: Record<string, ConfidenceLevel>;
  dataQualityNotes: string[];
}
```

---

## Data Models

### Session Type Union

```typescript
// Discriminated Union for Session Types
type SessionType =
  | MockInterviewSession
  | CoachingSession
  | GenericMeetingSession;

interface MockInterviewSession {
  type: "MockInterview";
  subtype: "SystemDesign" | "Coding" | "Behavioral" | "Unknown";
}

interface CoachingSession {
  type: "CoachingSession";
}

interface GenericMeetingSession {
  type: "GenericMeeting";
}

// Type guard functions
function isMockInterview(session: SessionType): session is MockInterviewSession {
  return session.type === "MockInterview";
}

function isCoachingSession(session: SessionType): session is CoachingSession {
  return session.type === "CoachingSession";
}

function isGenericMeeting(session: SessionType): session is GenericMeetingSession {
  return session.type === "GenericMeeting";
}
```

### Analysis Result Union

```typescript
// Union type for all analysis results
type AnalysisResult =
  | MockInterviewAnalysisJSON
  | CoachingAnalysisJSON
  | MeetingAnalysisJSON;

// Type guard for result types
function getAnalysisType(result: AnalysisResult): SessionType {
  return { type: result.metadata.sessionType } as SessionType;
}
```

### Mock Interview Categories (preserved from v1)

```typescript
interface ScorecardCategory extends CategoryBase {
  overall: { score: number; confidence: ConfidenceLevel; evidence: Evidence[]; };
  level: { assessment: "E5" | "E6" | "E7" | "Staff+" | "Unknown"; confidence: ConfidenceLevel; evidence: Evidence[]; };
  dimensions: {
    communication: DimensionScore;
    technicalDepth: DimensionScore;
    structure: DimensionScore;
    leadership: DimensionScore;
  };
  readiness: { percentage: number; confidence: ConfidenceLevel; formula: string; };
}

interface MistakeCategory {
  title: string;
  description: string;
  severity: "CRITICAL" | "HIGH" | "MEDIUM" | "LOW";
  category: "Fundamentals" | "API Design" | "Patterns" | "Domain Knowledge" | "Communication";
  confidence: ConfidenceLevel;
  evidence: Evidence[];
}

interface GapCategory {
  area: string;
  category: "Fundamentals" | "API Design" | "Patterns" | "Domain";
  priority: "P0" | "P1" | "P2";
  confidence: ConfidenceLevel;
  evidence: Evidence[];
}
```

---

## Error Handling

### Error Categories and Responses

**Classification Errors (Supervisor Agent)**:
- No clear signals detected → Default to GenericMeeting with LOW confidence warning
- Conflicting signals (equal scores) → Ask user to choose between top candidates
- Very short transcript (<20 lines) → Warn about limited analysis, proceed with GenericMeeting

**File Errors (User Errors)**:
- File not found → Clear message with attempted path, suggest checking path
- File not readable → Permission error message
- Empty file → "Transcript file is empty" with guidance

**Analysis Errors (System Errors)**:
- Subagent timeout → Fall back to direct analysis with truncation warning
- Invalid subagent JSON → Attempt repair, fail gracefully with partial results
- Context overflow → Truncate with warning, analyze available portion

**Data Quality Errors (Business Logic)**:
- No interview start detected (MockInterview) → Analyze from beginning, flag LOW confidence
- Category extraction failed → Output NOT_FOUND for that category, continue others

### Error Output Format

```markdown
**Analysis Warning**

[Error type]: [Description]
- Impact: [What couldn't be analyzed]
- Fallback: [What was done instead]
- Recommendation: [How to improve next time]
```

---

## Testing Strategy

### Unit Tests
- Supervisor signal detection accuracy per session type
- Session type classification with mixed signals
- Confidence score calculation boundary conditions
- Type guard function correctness

### Integration Tests
- Full analysis of sample System Design mock interview
- Full analysis of sample Coding mock interview
- Full analysis of sample Behavioral mock interview
- Full analysis of sample coaching session (like KDR transcript)
- Full analysis of sample generic meeting
- Subagent delegation for 600+ line transcripts
- User confirmation flow for ambiguous classification

### E2E Tests
- `/transcription-analyzer` with no args → AskUserQuestion flow
- `/transcription-analyzer path/to/mock.md` → Auto-detect MockInterview, full analysis
- `/transcription-analyzer path/to/coaching.md` → Auto-detect CoachingSession, coaching analysis
- `/transcription-analyzer path/to/meeting.md` → GenericMeeting fallback, meeting analysis
- Analysis of ambiguous transcript → User confirmation flow

---

## File Structure

```
transcription-analyzer/
├── SKILL.md                              # Main skill router (entry point)
├── prompts/
│   ├── supervisor_classifier.md          # Session type classification
│   ├── mock_interview_analyzer.md        # 10-category mock interview analysis
│   ├── coaching_analyzer.md              # 6-category coaching session analysis
│   ├── meeting_analyzer.md               # 6-category generic meeting analysis
│   └── confidence_scorer.md              # Confidence methodology reference
├── examples/
│   ├── sample_mock_interview.md          # Test mock interview transcript
│   ├── sample_coaching_session.md        # Test coaching transcript
│   └── sample_meeting.md                 # Test generic meeting transcript
└── .kiro/
    └── specs/
        └── transcription-analyzer/
            ├── requirements.md
            ├── design.md
            ├── tasks.md
            └── spec.json
```

---

## Topic Flow Analysis Architecture (v2.1)

### Three-Phase Map-Reduce Architecture

```mermaid
graph TB
    subgraph Phase1["Phase 1: Skeleton Extraction"]
        Input[Transcript] --> Extract[Extract Timestamps/Speakers]
        Extract --> Keywords[Detect Topic Keywords]
        Keywords --> Boundaries[Identify Break Points]
    end

    subgraph Phase2["Phase 2: Chunked Analysis (Parallel)"]
        Boundaries --> Split[Split into 15-20min chunks]
        Split --> Chunk1[Chunk Agent 1]
        Split --> Chunk2[Chunk Agent 2]
        Split --> ChunkN[Chunk Agent N]
        Chunk1 --> Topics1[Topics + Deviations + Fillers]
        Chunk2 --> Topics2[Topics + Deviations + Fillers]
        ChunkN --> TopicsN[Topics + Deviations + Fillers]
    end

    subgraph Phase3["Phase 3: Synthesis"]
        Topics1 --> Merge[Merge Topic Trees]
        Topics2 --> Merge
        TopicsN --> Merge
        Merge --> Dedupe[Dedupe Overlaps]
        Dedupe --> CrossChunk[Cross-Chunk Patterns]
        CrossChunk --> Sankey[Generate Sankey]
        CrossChunk --> Timeline[Generate Timeline]
        CrossChunk --> Insights[Generate Insights]
    end
```

### Topic Flow Analyzer Component

#### prompts/topic_flow_orchestrator.md

**Responsibility**: Orchestrate three-phase analysis for topic flow

**Contract**:

```typescript
interface TopicFlowInput {
  transcript: string;
  participants?: string[];  // Optional, will detect if not provided
  mainTopic?: string;       // Optional, will infer from content
}

interface TopicFlowAnalysisJSON {
  metadata: {
    sessionType: "TopicFlowAnalysis";
    duration: string;
    participants: string[];
    mainTheme: string;
    totalLines: number;
    analysisTimestamp: string;
  };
  topicHierarchy: ThemeNode[];
  deviations: DeviationEvent[];
  fillerAnalysis: FillerAnalysis;
  visualizations: {
    sankey: string;     // Mermaid sankey code
    timeline: string;   // Mermaid timeline code
  };
  insights: string[];
  confidenceSummary: ConfidenceSummary;
}

interface ThemeNode {
  theme: string;
  totalTime: string;
  confidence: number;
  subtopics: SubtopicNode[];
  revisitCount: number;
  fillerDensity: number;
}

interface SubtopicNode {
  name: string;
  time: string;
  chunks: string[];
  keyTerms: string[];
  confidence: number;
}

interface DeviationEvent {
  id: string;
  type: "unrelated_jump" | "rabbit_hole" | "depth_spiral";
  severity: "high" | "medium" | "low" | "intentional";
  fromTopic: string;
  toTopic: string;
  timestamp: string;
  duration: string;
  returned: boolean;
  returnedBy?: string;
  returnTimestamp?: string;
  returnPhrase?: string;
  speakersInvolved: string[];
}

interface FillerAnalysis {
  byTopic: TopicConfidence[];
  bySpeaker: SpeakerFillerProfile[];
  insights: string[];
}

interface TopicConfidence {
  topic: string;
  fillerDensity: number;
  confidence: number;
  signal: "strong" | "moderate" | "shaky" | "review";
}

interface SpeakerFillerProfile {
  speaker: string;
  totalWords: number;
  totalFillers: number;
  density: string;
  primaryFiller: string;
  mostUncertainTopic: string;
  mostConfidentTopic: string;
  pattern?: string;
}
```

### Filler Word Patterns

```typescript
const FILLER_PATTERNS = {
  classic: ["um", "uh", "er", "ah", "hmm"],
  hedge: ["like", "you know", "basically", "essentially", "kind of", "sort of", "i guess", "maybe"],
  stalling: ["so basically what i'm trying to say", "let me think", "how do i put this", "what's the word"],
  confidence_killer: ["i'm not sure but", "i think maybe", "don't quote me", "if i recall"]
};
```

### Deviation Detection Logic

```typescript
// Unrelated Jump: Cosine similarity < 0.3
function detectUnrelatedJump(segmentA: Segment, segmentB: Segment): boolean {
  const similarity = cosineSimilarity(
    embed(segmentA.theme + segmentA.keyTerms),
    embed(segmentB.theme + segmentB.keyTerms)
  );
  return similarity < 0.3;
}

// Rabbit Hole: Doesn't return to original topic
function detectRabbitHole(topicStack: Topic[], currentTime: Time): DeviationEvent[] {
  return topicStack
    .filter(t => timeSince(t.leftAt) > 15 && !t.returned)
    .map(t => ({ type: "rabbit_hole", ... }));
}

// Depth Spiral: Time > 2x average on one subtopic
function detectDepthSpiral(theme: Theme): DeviationEvent[] {
  const avgTime = theme.totalTime / theme.subtopics.length;
  return theme.subtopics
    .filter(s => s.time > avgTime * 2)
    .map(s => ({ type: "depth_spiral", ... }));
}
```

---

## Portability Constraints

The following MUST NOT appear in any skill file:

| Prohibited | Reason |
|------------|--------|
| `mcp__mem0__*` | Memory service dependency |
| `mcp__obsidian__*` | Note-taking service dependency |
| `/Users/vishnu/` | Personal path hardcoding |
| `Mock Interviews/` | Obsidian-specific folder |
| `study plan` | Integration with other skills |
| `gotcha` | Integration with gotcha tracking |

**Allowed Tools Only**:
- `Read` - File reading
- `AskUserQuestion` - User interaction
- `Task` - Subagent delegation (with `subagent_type: "Explore"`)

---

## Output Format Specification

### Session Type Badge

All outputs include a session type badge:

```markdown
**Session Type:** MockInterview.SystemDesign [Confidence: HIGH 92%]
**Session Type:** CoachingSession [Confidence: MEDIUM 68%]
**Session Type:** GenericMeeting [Confidence: LOW - Default]
```

### Mock Interview Report Structure

```markdown
## Mock Interview Analysis

**File:** [filename]
**Session Type:** MockInterview.[subtype] [Confidence: X]
**Date Analyzed:** [timestamp]

---

### 1. Scorecard
### 2. Time Breakdown
### 3. Communication Signals
### 4. Things That Went Well
### 5. Mistakes Identified
### 6. Knowledge Gaps
### 7. Behavioral Assessment
### 8. Factual Claims
### 9. Action Items
### 10. Interviewer Quality

---
### Confidence Summary
```

### Topic Flow Analysis Report Structure

```markdown
## Topic Flow Analysis

**File:** [filename]
**Session Type:** TopicFlowAnalysis [Confidence: X]
**Duration:** Xh Ym
**Participants:** [list]
**Main Theme:** [detected or provided]

---

### 1. Topic Hierarchy
[Hierarchical topic tree with time spent and confidence per topic]

### 2. Topic Flow (Sankey Diagram)
[Mermaid sankey showing topic transitions with time-spent as edge width]

### 3. Timeline View
[Mermaid timeline showing chronological progression with deviation markers]

### 4. Deviation Report
| # | Type | From → To | Duration | Severity | Returned By |
[Table of all detected deviations]

**Summary:** X deviations, Y min off-topic, primary deviator: [name], primary returner: [name]

### 5. Filler Word Analysis
**Per-Topic Confidence:**
[Heatmap showing confidence levels based on filler density]

**Per-Speaker Profile:**
[Filler stats per speaker with most/least confident topics]

### 6. Insights & Recommendations
[LLM-generated actionable insights]

---
### JSON Export
[Full structured data]
```

---

### Coaching Session Report Structure

```markdown
## Coaching Session Analysis

**File:** [filename]
**Session Type:** CoachingSession [Confidence: X]
**Coach:** [name] | **Mentee:** [name]
**Topics:** [topic1], [topic2]

---

### 1. Key Advice & Tips
[Categorized by domain with direct quotes]

### 2. Scripts & Patterns
[Reusable scripts with context for when to use]

### 3. Action Items
[Explicit and implicit tasks with urgency]

### 4. Questions Raised
[Topics needing further exploration]

### 5. Session Quality
[Actionability score, examples count]

---
### Confidence Summary
```

### Generic Meeting Report Structure

```markdown
## Meeting Analysis

**File:** [filename]
**Session Type:** GenericMeeting [Confidence: X]
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
