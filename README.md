# Learn2Land

**Which skills does the market actually want for the role I'm targeting — and which of those can I prove?**

Learn2Land reads real job postings, extracts the skills they ask for, and compares them to your evidence (shipped / built / studied). It tells you which gap to close next and shows the exact posting sentences behind every number.

![ci](https://github.com/YOUR_USER/learn2land/actions/workflows/ci.yml/badge.svg)

## Results

Skill extraction, micro-averaged over (posting, skill) pairs.

| Method | Golden set | Postings | Precision | Recall | F1 |
|---|---|---|---|---|---|
| Taxonomy rules | synthetic fixtures | 6 | 0.985 | 1.000 | 0.992 |
| Taxonomy rules | real, hand-labelled | 50 | — | — | — |
| Gemini (taxonomy-constrained) | real, hand-labelled | 50 | — | — | — |
| Hybrid (rules ∪ Gemini) | real, hand-labelled | 50 | — | — | — |

The fixture row only proves the harness works — the fixtures were written by the same person as the rules. The real rows are the ones that count and are filled in from `evals/golden/software.real.json`.

### Findings so far

- **Sentence-initial capitals defeat case-sensitive aliases.** "React to feedback quickly" is counted as React. Case-sensitivity fixes "the rest of the team" (REST) and "the go-to person" (Go), but not a word that starts a sentence. Candidate fix: ignore a case-sensitive match at sentence start unless another tech term is in the same sentence. Not applied yet — waiting for real-posting numbers to see if it matters.
- **Nested aliases need an explicit rule.** "GitHub Actions" used to count as Git. Overlaps now resolve longest-first, and the taxonomy's `implies` field says when a specific skill entails a broader one (Amazon Bedrock → AWS).

## How it works

```
postings (JSON) ─► extract skills ─► demand per skill ─► compare with profile ─► gaps + citations
                   ├ taxonomy rules  (share of postings)   (tier-weighted)
                   └ Gemini (ids only, logged)
```

- **Market coverage** — demand-weighted share of skills you can claim at all.
- **Evidence coverage** — the same, weighted by proof: shipped 1.0, built 0.7, studied 0.3.
- **Gap ranking** — how many points of evidence coverage one project with that skill would add.

See [docs/architecture.md](docs/architecture.md) for the deployed design (Cloud Run + Cloud SQL/pgvector, `australia-southeast1`).

## Try it

```bash
uv sync
cp profile.example.yaml profile.yaml   # fill in your evidence
uv run l2l demand evals/golden/software.fixtures.json
uv run l2l gap profile.yaml evals/golden/software.fixtures.json --role engineer
uv run l2l explain evals/golden/software.fixtures.json docker
uv run l2l eval evals/golden/software.fixtures.json --show-errors
```

With real postings in `data/postings/`:

```bash
uv run l2l gap profile.yaml data/postings --role "AI Engineer"
```

Gemini extraction: `uv sync --extra llm`, set `GEMINI_API_KEY` (or Vertex AI vars, see `.env.example`), then add `--method llm` or `--method hybrid`. Every call is logged to `logs/llm_calls.jsonl` with prompt version, tokens and latency.

## Limitations

- Demand counts mentions, not importance: "required" and "nice to have" weigh the same.
- Role filtering is a substring match on the title.
- Tier weights (1.0 / 0.7 / 0.3) are a judgement call, not fitted.
- The taxonomy covers software/AI roles only (~75 skills).

## Roadmap

Citation RAG over posting chunks (pgvector) · `/admin/llm` cost dashboard · Next.js gap page · nightly Cloud Run Job.
