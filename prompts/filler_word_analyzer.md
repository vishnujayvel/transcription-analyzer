# Filler Word Analyzer Prompt

This prompt analyzes filler words as confidence signals within a transcript chunk.

## Four Filler Categories

### 1. Classic Fillers
Patterns: um, uh, er, ah, hmm
Signal: Processing/thinking pause

### 2. Hedge Words
Patterns: like, you know, basically, essentially, kind of, sort of, I guess, maybe
Signal: Uncertainty about statement

### 3. Stalling Phrases
Patterns: "so basically what I'm trying to say", "let me think", "how do I put this", "what's the word"
Signal: Formulating complex thought

### 4. Confidence Killers
Patterns: "I'm not sure but", "I think maybe", "don't quote me on this", "if I recall correctly"
Signal: Explicit low confidence

## Detection Rules
- Case-insensitive matching
- Count per speaker per topic
- Calculate filler density: total_fillers / total_words * 100

## Topic Confidence Score
```
filler_density = total_fillers / total_words
base_confidence = max(0, 100 - (filler_density * 1000))

# Weight by category (confidence killers hurt more)
penalty = (
  confidence_killers * 5 +
  hedge_words * 2 +
  stalling_phrases * 1.5 +
  classic_fillers * 1
)

topic_confidence = max(0, base_confidence - penalty)
```

## Output Schema (JSON)
```json
{
  "chunk_id": "0:18-0:38",
  "filler_counts": {
    "vishnu": {
      "classic": 12,
      "hedge": 18,
      "stalling": 3,
      "confidence_killer": 2,
      "total": 35,
      "word_count": 1250,
      "density_percent": 2.8
    },
    "amy": {
      "classic": 4,
      "hedge": 8,
      "stalling": 1,
      "confidence_killer": 0,
      "total": 13,
      "word_count": 890,
      "density_percent": 1.5
    }
  },
  "filler_by_topic": {
    "Kafka EOS": {
      "total_fillers": 22,
      "word_count": 450,
      "density_percent": 4.9,
      "confidence_score": 48,
      "signal": "review"
    },
    "Counter Sharding": {
      "total_fillers": 8,
      "word_count": 380,
      "density_percent": 2.1,
      "confidence_score": 79,
      "signal": "moderate"
    }
  },
  "chunk_summary": {
    "total_fillers": 48,
    "total_words": 2140,
    "overall_density_percent": 2.2,
    "most_uncertain_topic": "Kafka EOS",
    "most_confident_topic": "Counter Sharding"
  }
}
```

## Signal Thresholds
- strong: confidence >= 80
- moderate: confidence 60-79
- shaky: confidence 40-59
- review: confidence < 40
