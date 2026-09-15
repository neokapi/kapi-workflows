# kapi-workflows

Reusable GitHub Actions workflows for [kapi](https://github.com/neokapi/neokapi) — the whole continuous-localization happy path as one `uses:` line each. They compose [`setup-kapi`](https://github.com/neokapi/setup-kapi) and [`kapi-action`](https://github.com/neokapi/kapi-action); use those directly when you need a custom job shape.

## `up.yml` — catch up on a schedule, deliver a PR

```yaml
name: Translations
on:
  schedule:
    - cron: "0 6 * * 1-5"
  workflow_dispatch:

jobs:
  up:
    uses: neokapi/kapi-workflows/.github/workflows/up.yml@v1
    permissions:
      contents: write
      pull-requests: write
    secrets:
      bowrain-auth-token: ${{ secrets.BOWRAIN_AUTH_TOKEN }}   # server-connected projects
      # anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}   # or local-engine runs
```

Runs `kapi up` — the kapi loop — and opens a pull request with the produced translations and a kapi up report (outcome, passes, parked locales). A run that **parks** (work remains that needs a person) still delivers what it caught up; set `fail-on-parked: true` to block instead.

Inputs: `project`, `args`, `create-pull-request` (default `true`), `fail-on-parked`, `plugins` (default `bowrain`), `kapi-version`, `server`, `runs-on`. Secrets: `bowrain-auth-token`, `anthropic-api-key`, `deliver-token` (a PAT or App token when CI/deploys should react to the delivered translations — the default `GITHUB_TOKEN` triggers no workflows). Outputs: `outcome`, `passes`, `parked-locales`, `has-changes`, `pull-request-url`.

### What a run does

Each pass, for every language behind its ship gate: **reuse** exact translation-memory matches first (free), **translate** what remains with the configured AI provider plus the project's terminology, then **check** what was produced — placeholder integrity, inline tags, do-not-translate terms. A unit with a failing finding counts as *drafted*, not translated, so bad output can never lift a language over its gate. Passes repeat until every gate clears or nothing progresses.

```mermaid
flowchart LR
    S[source changes] --> U[kapi up]
    subgraph PASS ["each pass, per language behind its gate"]
        TM["1 · reuse<br/>TM exact matches"] --> AI["2 · translate<br/>AI + terminology"] --> CK["3 · check<br/>placeholders · terms · tags"]
    end
    U --> PASS
    CK -->|every gate met| CV["up to date<br/>PR with translations"]
    CK -->|needs a person| PK["parked<br/>the review queue"]
    PK --> RV["review & approve<br/>committed under .kapi/state"]
    RV -.->|next run sees it| U
```

Parked work is the review queue, not an error: a person reviews and approves it, `kapi commit` records the decision under `.kapi/state/` (or the connected server records it), and the `reviewed` coverage the ship gate measures goes up. The next run and the next gate see it.

## `gate.yml` — fail PRs on unmet content quality gates

```yaml
name: Ship gate
on:
  pull_request:
    paths: ["content/**", "src/locales/**"]

jobs:
  ship-gate:
    uses: neokapi/kapi-workflows/.github/workflows/gate.yml@v1
    permissions:
      contents: read
      pull-requests: write
```

Runs `kapi check --ship`: the project's bound gates (voice, terminology, rule-based checks) plus its ship and source coverage gates. An unmet gate exits `3`, fails the job with a distinct "gate unmet" annotation, and posts one sticky report comment on the PR. Ordinary builds never fail on target-language drift; the gate is the explicit, opt-in enforcement point.

Inputs: `project`, `args` (default `--ship`), `plugins`, `kapi-version`, `pr-comment` (default `true`), `fetch-depth` (default `1`), `server`, `runs-on`. Output: `gate` (`pass`/`fail`).

A diff-scoped check reads the commits it compares, so pass `fetch-depth: 0` with `--diff-range` or `--diff-against`:

```yaml
jobs:
  ship-gate:
    uses: neokapi/kapi-workflows/.github/workflows/gate.yml@v1
    with:
      args: "--diff-range ${{ github.event.pull_request.base.sha }}...${{ github.event.pull_request.head.sha }}"
      fetch-depth: 0
```

## Versions

`@v1` is a floating major tag. The workflows install kapi 1.2.0 by default. Set `kapi-version` to another release to pin it, or to `latest` for the newest stable release.

## License

Apache-2.0 — see [LICENSE](LICENSE).
