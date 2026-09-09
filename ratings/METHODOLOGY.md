# Hutash Score — Rating Methodology

Version: 1.0

## Scale
0–10, one decimal place.

## Formula (v1.0 — no user ratings yet)

  Score = (Benchmark × 0.70) + (Popularity × 0.30)

### Benchmark (70% weight)
Published benchmark score, normalized to 0–10 via min-max
scaling within a realistic floor/ceiling for the modality
(not simple division):

| Modality    | Metric     | Worst | Best | Formula                          |
|-------------|------------|-------|------|-----------------------------------|
| TTS         | UTMOS      | 1.0   | 5.0  | (UTMOS - 1.0) / 4.0 × 10        |
| STT         | WER%       | 50    | 2    | (50 - WER) / 48 × 10            |
| Translation | BLEU       | 5     | 50   | (BLEU - 5) / 45 × 10            |
| LLM         | MMLU%      | 25    | 90   | (MMLU - 25) / 65 × 10           |
| Music       | MOS        | 1.0   | 5.0  | (MOS - 1.0) / 4.0 × 10          |
| Voice Clone | Sim (0-1)  | 0.3   | 0.95 | (Sim - 0.3) / 0.65 × 10         |

Bounds are set at realistic floor/ceiling for the modality,
not based on current best model. Scores remain stable when
new models are added.

If no published benchmark exists, an engineering estimate
is used and noted in the score entry.

### Popularity (30% weight)
Based on HuggingFace downloads + likes, normalized to 0–10
relative to the most popular model in the same modality.

## Future (v1.1)
User ratings will be added:
  User ratings:  50%
  Benchmark:     35%
  Popularity:    15%

## Source
Each score in ratings.json includes a "source" field linking
to the benchmark paper or dataset used.
