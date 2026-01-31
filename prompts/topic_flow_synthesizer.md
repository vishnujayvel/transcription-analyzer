# Topic Flow Synthesizer Prompt

Merge chunk analysis results and generate visualizations for Phase 3.

## Inputs
- Array of ChunkAgentOutput from all chunk agents
- Participant list
- Main theme (detected or provided)

## Synthesis Steps

### 1. Merge Topic Trees
- Combine themes from all chunks
- Handle 2-minute overlap: If same topic appears in overlap region of adjacent chunks, dedupe
- Use entry/exit topics to stitch: chunk N exit_topic should match chunk N+1 entry_topic

### 2. Fuzzy Match Theme Names
- Calculate keyword overlap between theme names across chunks
- If overlap > 60%: Consider same theme, pick canonical name (most frequent or first occurrence)
- Example: "Kafka Patterns" and "Kafka EOS" both have "Kafka" → check sub-topics to decide if same theme

### 3. Cross-Chunk Pattern Detection
Detect:
- Revisit counts: "You returned to X topic 3 times"
- Recurring tangents: Same deviation type happening repeatedly
- Confidence dips: Topics that consistently have high filler density
- Return heroes: Speaker who most often brings conversation back on track

### 4. Generate Mermaid Sankey
```mermaid
sankey-beta
Intro,Writes at Scale,5
Writes at Scale,Hot Keys,15
Hot Keys,Counter Sharding,10
Counter Sharding,TANGENT: Builder Trap,3
TANGENT: Builder Trap,Counter Sharding,3
Writes at Scale,Kafka Patterns,20
```
- Edge width = time spent (minutes)
- Prefix tangents with "TANGENT:" for visual distinction

### 5. Generate Mermaid Timeline
```mermaid
timeline
    title Topic Flow - Database Party
    section Hour 1
        0:00-0:15 : Hot Keys 🟢
        0:15-0:25 : Counter Sharding 🟢
        0:25-0:28 : ⚠️ Builder Trap tangent
    section Hour 2
        0:30-0:50 : Kafka Patterns 🟡
        0:50-1:02 : Exactly-Once 🔴
```
- Color emoji by confidence: 🟢 strong, 🟡 moderate, 🔴 review
- Mark deviations with ⚠️

### 6. Generate Insights
Categories:
- **Focus Insights**: Main themes, time distribution, depth vs breadth balance
- **Dynamics Insights**: Who drove tangents, who refocused, group dynamics
- **Confidence Insights**: Knowledge gaps (high filler topics), strong areas
- **Recommendations**: Topics to review, patterns to practice

## Output Schema (JSON)
```json
{
  "merged_hierarchy": [...],
  "deviation_summary": {...},
  "filler_summary": {...},
  "visualizations": {
    "sankey": "sankey-beta\n...",
    "timeline": "timeline\n..."
  },
  "insights": {
    "focus": [...],
    "dynamics": [...],
    "confidence": [...],
    "recommendations": [...]
  },
  "cross_chunk_patterns": {
    "revisit_counts": {"Kafka": 3, "Sharding": 2},
    "primary_deviator": "Vishnu",
    "primary_returner": "Amy",
    "confidence_dips": ["Kafka EOS", "Isolation Levels"]
  }
}
```
