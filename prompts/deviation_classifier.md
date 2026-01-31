# Deviation Classifier Prompt

This prompt detects three types of conversational deviations within a transcript chunk.

## Three Deviation Types

### 1. Unrelated Jump
- Definition: Topic A → Topic B with no logical connection
- Detection: New topic has <20% keyword overlap with previous topic
- Example: "Kafka partitioning" → "What's for lunch?"

### 2. Rabbit Hole
- Definition: Deep exploration that doesn't return to the main thread
- Detection: Topic exploration > 10 minutes AND no return signal detected
- Return signals: "anyway", "going back to", "as I was saying"

### 3. Depth Spiral
- Definition: Over-indexing on one sub-topic at expense of breadth
- Detection: Time on sub-topic > 2x average of sibling sub-topics
- Example: Spending 20 min on "exactly-once semantics" when other Kafka topics got 5 min each

## Severity Classification
- 🔴 High: Unrelated jump + never returned, OR depth spiral > 3x average
- 🟠 Medium: Rabbit hole > 10min, OR depth spiral 2-3x average
- 🟡 Low: Brief tangent < 5min with clean return
- ⚪ Intentional: Explicit marker like "let's take a quick detour"

## Return Attribution
When conversation returns to original topic, track:
- Who initiated the return (speaker name)
- Return phrase used
- Timestamp of return

## Output Schema (JSON)
```json
{
  "chunk_id": "0:18-0:38",
  "deviations": [
    {
      "id": "dev_001",
      "type": "rabbit_hole",
      "severity": "medium",
      "from_topic": "Kafka EOS",
      "to_topic": "Builder's trap discussion",
      "timestamp": "0:25:30",
      "duration_minutes": 8,
      "returned": true,
      "returned_by": "Amy",
      "return_timestamp": "0:33:30",
      "return_phrase": "going back to Kafka...",
      "speakers_involved": ["Vishnu", "Amy"],
      "confidence": 0.78
    }
  ],
  "deviation_summary": {
    "total": 2,
    "by_type": {"unrelated_jump": 0, "rabbit_hole": 1, "depth_spiral": 1},
    "total_time_off_topic_minutes": 12
  }
}
```

## Keyword Overlap Calculation (for Unrelated Jump)
```
overlap = |keywords_A ∩ keywords_B| / |keywords_A ∪ keywords_B|
if overlap < 0.20: flag as unrelated_jump
```
