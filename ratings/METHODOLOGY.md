# Hutash Score — Rating Methodology

Version: 1.0

## Scale
0–10, one decimal place.

## Formula (v1.0 — no user ratings yet)

  Score = (Benchmark × 0.70) + (Popularity × 0.30)

### Benchmark (70% weight)
Published benchmark score, normalized to 0–10:

| Modality    | Metric               | Normalization       |
|-------------|----------------------|----------------------|
| TTS         | UTMOS (0–5)          | × 2                 |
| STT         | WER% (0–100)         | (100 − WER) / 10    |
| Translation | BLEU (0–100)         | / 10                |
| LLM         | MMLU% (0–100)        | / 10                |
| Music       | MOS (0–5)            | × 2                 |
| Voice Clone | Speaker Sim (0–1)    | × 10                |

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
