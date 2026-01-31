# Transcription Topic Extractor

## Purpose

Extract two-level topic hierarchy (themes → sub-topics) from a transcript chunk with time tracking. This enables cross-chunk coherence tracking and hierarchical topic mapping for transcription analysis.

## Input

- **Transcript chunk**: 15-20 minutes of content
- **Chunk ID**: Format like "0:00-0:20" (start-end timestamps)
- **Participant list**: Array of speaker names

## Extraction Rules

### Theme Detection

- A theme is a major topic discussed for **3+ minutes**
- Look for explicit topic introductions:
  - "let's talk about"
  - "the next question is"
  - "moving on to"
  - "before we discuss"
- Look for semantic clusters of related discussion
- Natural transitions and speaker pauses often mark theme boundaries

### Sub-topic Detection

- Sub-topics are specific concepts within a theme
- Each sub-topic should have distinct key terms
- Track time spent (infer from timestamps in transcript)
- Sub-topics typically last 1-8 minutes within a theme
- List 2-5 sub-topics per theme for granularity

### Entry/Exit Topics

- **CRITICAL**: Record what topic the chunk starts with (`entry_topic`)
- **CRITICAL**: Record what topic the chunk ends with (`exit_topic`)
- These are used for cross-chunk stitching and conversation flow analysis
- Entry/exit topics must match existing theme names or sub-topic names
- If chunk starts mid-topic, set entry_topic to that topic name

## Output Schema (JSON)

```json
{
  "chunk_id": "0:00-0:20",
  "themes": [
    {
      "name": "Counter Sharding",
      "start_time": "0:02",
      "end_time": "0:15",
      "duration_minutes": 13,
      "subtopics": [
        {
          "name": "Hot key problem",
          "duration_minutes": 5,
          "key_terms": ["row contention", "single row"]
        },
        {
          "name": "Shard strategies",
          "duration_minutes": 8,
          "key_terms": ["range", "hash", "random"]
        }
      ],
      "speaker_breakdown": {
        "vishnu": 60,
        "amy": 30,
        "victor": 10
      },
      "confidence": 0.85
    }
  ],
  "entry_topic": "Introduction",
  "exit_topic": "Counter Sharding",
  "total_themes": 2,
  "overall_confidence": 0.82
}
```

### Schema Notes

- `chunk_id`: Unique identifier for this transcript segment
- `themes`: Array of major topics discussed in the chunk
  - `name`: Theme title (should be descriptive and consistent)
  - `start_time`: When theme begins (HH:MM or MM:SS format)
  - `end_time`: When theme ends
  - `duration_minutes`: Calculated duration for this theme
  - `subtopics`: Array of 2-5 sub-topics under this theme
    - `name`: Specific concept name
    - `duration_minutes`: Time spent on this sub-topic
    - `key_terms`: 2-4 distinctive terms/phrases for this sub-topic
  - `speaker_breakdown`: Percentage of speaking time per participant (should sum to 100)
  - `confidence`: 0.0-1.0 confidence score for this theme's accuracy
- `entry_topic`: Name of topic when chunk starts
- `exit_topic`: Name of topic when chunk ends
- `total_themes`: Count of themes extracted
- `overall_confidence`: 0.0-1.0 confidence for entire extraction

## Anti-Hallucination Rules

- **Only extract topics explicitly discussed** in the transcript
- **Cite evidence**: For each theme/sub-topic, be able to reference timestamp ranges and key phrases
- **If uncertain**: Mark confidence < 0.6 and note reasoning
- **Avoid inference**: Don't infer topics not directly discussed
- **Consistency check**: Entry/exit topics must exist in the themes list or be accurately described as transitions
- **No phantom speakers**: Only include speakers from the provided participant list in speaker_breakdown

## Quality Checklist

Before returning output:
- [ ] All timestamps are valid and in chronological order
- [ ] Speaker breakdown percentages sum to 100%
- [ ] entry_topic and exit_topic are present and justified
- [ ] Confidence scores are realistic (not all 1.0 or all 0.9)
- [ ] Each theme is 3+ minutes
- [ ] Each sub-topic has 2-4 key terms
- [ ] No topics mentioned are hallucinated
- [ ] Time calculations are accurate

Keep the extraction focused and actionable - this will be run by chunk agents in parallel.
