# prom-client

## Purpose

A fork of [siimon/prom-client](https://github.com/siimon/prom-client), the standard Node Prometheus client, republished as `@effyis/prom-client` on the internal registry. It carries exactly one functional change on top of upstream v15.1.3: a `Counter`'s `labels()` child also exposes `set`, where upstream exposes only `inc`. Everything else — histograms, summaries, gauges, the default metrics, cluster aggregation, the registry — is upstream's. If the fork is lost, the failure is at install rather than at runtime: services resolving `@effyis/prom-client` cannot install, and any code calling `.labels(...).set(...)` on a counter would break if swapped to upstream, while everything else would work unchanged.

## Where it sits in Socialgist

Cross-cutting infrastructure, not part of any pipeline stage. It is a dependency of most of the Node services in this set: the whole `reddit-*` family (`reddit-unblocker`, `reddit-rlp`, `reddit-mimic-proxy`, `reddit-session-store`, `reddit-session-generator`, `reddit-impit-proxy`) depends on `@effyis/prom-client` explicitly, while [pauk](../pauk), [pauk-crawl-launcher](../pauk-crawl-launcher), [pauk-build-manager](../pauk-build-manager) and the captcha repos use unscoped `prom-client` from the public registry. Downstream it feeds whatever scrapes those services' `/metrics` endpoints. Published to Verdaccio at `verdaccio.sgdctroy.net`.

## Key concepts and domain vocabulary

- **The `set` on a counter child** — the reason this fork exists. Upstream treats counters as monotonic and deliberately withholds `set` from the `labels()` child; this fork restores it, so a labelled counter can be assigned rather than only incremented. Using it means the metric is no longer strictly monotonic, which is a real departure from Prometheus convention — deliberate, but know that you are doing it.
- **Registry** — the collection metrics register into. Upstream vocabulary, but central here: `register` is the default global one, and `await registry.metrics()` is what an endpoint returns.
- **Default metrics** — the process and Node runtime metrics (`process_*`, `nodejs_*`) collected automatically, which is why every service in this set exposes them without asking.
- **Aggregator** — how a metric is combined across `cluster` workers. Defaults are sensible per type; custom metrics sum. Relevant because worker-local registries otherwise report only one worker's view.
- **Exemplars** — OpenTelemetry trace exemplars attached to samples, from the `@opentelemetry/api` dependency. Present upstream, unused by anything in this set.

## Architecture

The architecture is upstream's and unchanged; only the fork delta is worth documenting here.

- `lib/counter.js` — the one modified source file. In the `labels()` child object, `set: this.set.bind(this, labels)` sits alongside upstream's `inc`. Four lines changed in total, including the test.
- `package.json` — renamed to `@effyis/prom-client`, `repository` and `homepage` repointed to `Effyis/prom-client`, and `publishConfig.registry` set to the internal Verdaccio. The version tracks upstream's exactly (15.1.3).
- `test/counterTest.js` — the added coverage for the `set` child.
- `index.js`, `lib/` and `index.d.ts` — upstream, untouched apart from the above.

## Running it locally

- `npm install`, then `require('@effyis/prom-client')`. Node `^16 || ^18 || >=20` per `engines`.
- `npm test` runs the full upstream gate in sequence: `lint`, `check-prettier`, `compile-typescript` and `test-unit` (jest). Any one failing fails the lot.
- `npm run test-unit` alone is the fast loop; `npm run benchmarks` runs the benchmark suite.
- `npm run prepare` installs husky hooks on first install.
- No environment variables are read. `.npmrc` here sets only `package-lock=false` — this repo intentionally has no lockfile, unlike every other repo in the set.

## Conventions and constraints

- **Keep the fork delta minimal.** The whole maintenance strategy is that this is upstream plus four lines, so merges stay trivial. Anything added here has to be carried forward forever; prefer contributing upstream or wrapping in the consuming service.
- **Merging upstream is the update path.** History shows `Merge upstream v15.1.3` as the most recent commit, with the local changes rebased on top. Keep the version field in step with upstream's rather than bumping independently.
- **The version is upstream's, not ours.** `15.1.3` here means "upstream 15.1.3 plus our delta". There is no separate fork version, so two builds of `@effyis/prom-client@15.1.3` from different points in this repo's history are not necessarily identical.
- **`author` remains "Simon Nyberg" and the licence is upstream's Apache-2.0.** Both are correct and should stay; this is a fork, not a rewrite.
- **`package-lock=false`** in `.npmrc` means no lockfile is committed, following upstream. Do not add one.
- The README is upstream's and documents `prom-client`, not the scoped name — install instructions there will not match how this is consumed internally, and it does not mention the counter `set` addition at all.
- GitHub Actions is the CI here, inherited from upstream, making this the only repo in the set that is not Drone-based or CI-less.

## Data handling

No customer data, PII, or licensed source content passes through this library. It handles only metric names, label values and numeric samples produced by the consuming service. The one caution is generic to metrics rather than specific to this fork: label values become part of the exposed time series, so a service that labels a metric with a URL, session id or user identifier publishes that to whatever scrapes `/metrics`. Nothing is persisted by the library itself — metrics live in the in-process registry and are serialized on demand. No retention or compliance policy is documented in this repo.

## Testing and CI

GitHub Actions (`.github/workflows/ci.yml`) runs on pushes to `master`, `main` and `next`, and — unlike the Drone pipelines elsewhere in this set — **on pull requests to any branch**, so this is the one repo here where PRs are actually gated. The matrix covers Node 16, 18, 20, 21 and 22 across Ubuntu, Windows and macOS. `npm test` chains lint, prettier check, TypeScript compilation and jest, so formatting and type declarations are enforced alongside behaviour. A second workflow, `changelog.yml`, handles release notes. Manual verification after a merge from upstream: confirm `lib/counter.js` still has the `set` binding in the `labels()` child — a clean upstream merge will silently drop it if that block was refactored — and run `npm run test-unit` to check the added counter test still passes. That single test is the only guard on the only reason this fork exists.

## Ownership

Primary maintainer and contact: Aleksandar Mitic (amitic@socialgist.com).
