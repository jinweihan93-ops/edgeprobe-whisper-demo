# EdgeProbe — whisper CI demo

This repo simulates a **whisper.cpp maintainer** using
[EdgeProbe](https://github.com/jinweihan-ai/edge-probe) as a CI
check. Every PR here:

1. synthesizes a trace from `params.json` (deterministic pure function)
2. runs the EdgeProbe Action pinned at `action-v0.1.1`
3. sticky-comments the rendered verdict

Three demo PRs are **permanently open** — point and click to see what
the Action looks like on a real GitHub pull request.

## The three demo PRs

| PR | Diff | What you'll see |
|----|------|-----------------|
| regression | `beam_size: 1 → 5` | **Red** — decode +4×, headline +114%, share URL |
| green | `n_threads: 4 → 8` | **Green ✓** — encoder speedup, headline −18% |
| first-run | new project, no baseline | First-trace card, share URL, no regression math |

Once the PRs are linked from the EdgeProbe root README, you can
one-click from there to the rendered sticky comments.

## Why synthetic, not real whisper?

GitHub runner CPU jitter corrupts real benchmarks: two runs on the
same machine can differ by 10-15%, which is already more than the
Action's default regression threshold. A demo that "sometimes" fires
a false regression builds the wrong intuition about EdgeProbe.

Instead, `bin/synthesize.ts` is a deterministic pure function of
`params.json`:

```
whisper_ms = BASE_ENCODE[model] * (1.5 / sqrt(threads))   # encoder → `whisper` column
decode_ms  = BASE_ENCODE[model] * 0.4 * (beam ^ 0.9)      # text decoder → `decode` column
```

Plus **±3% SHA-seeded noise** so numbers look real but re-runs of the
same PR are byte-identical — no flaky CI, ever.

For a genuine run-whisper-in-CI setup, see `examples/whisper-real-bench/`
in the EdgeProbe main repo (not yet implemented — Layer 2 in the
fidelity matrix; add when Layer 1 graduates).

## Running locally

```bash
# 1. Synthesize current-run trace
bun run bin/synthesize.ts > /tmp/trace.json

# 2. Dry-run the Action against the baseline (requires edge-probe checked out next to this repo)
bun run ../edge-probe/action/src/entry.ts \
  --trace /tmp/trace.json \
  --baseline baselines/main.json \
  --threshold 0.15 \
  --dry-run
```

The last command prints the PR comment to stdout — same shape GitHub
will post, minus the share URL (dry-run skips the backend).

## Regenerating the baseline

When legitimate perf work produces a new "expected" number, refresh:

```bash
bun run bin/synthesize.ts > baselines/main.json
git commit -am "demo: refresh baseline"
```

All three demo PRs then automatically diff against the new baseline
on their next workflow run (push any commit to the branch, or hit
"Update branch" on GitHub).

## Files

| File | What it is |
|------|-----------|
| `params.json` | Whisper knobs: `model_size`, `beam_size`, `n_threads` |
| `bin/synthesize.ts` | Pure function: `params.json` → trace.json |
| `baselines/main.json` | Baseline generated from default `params.json` |
| `.github/workflows/ci.yml` | Runs the Action on every PR |

## Related

- **[EdgeProbe main repo](https://github.com/jinweihan-ai/edge-probe)** — the Swift SDK, backend, dashboard, and the Action itself live there
- **[Action code](https://github.com/jinweihan-ai/edge-probe/tree/action-v0.1.0/action)** — pinned at the tag this demo uses
