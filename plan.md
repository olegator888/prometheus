# Prometheus Source Code Exploration Plan

This plan is for the first couple months of exploring Prometheus as a Go backend
engineer. The goal is not to understand every package immediately. The goal is
to build a working mental model of the system, practice reading production Go,
and create a path toward small, high-quality contributions.

## Principles

- Prefer running code over only reading code.
- Read tests together with implementation; tests often explain the real contract.
- Keep notes about questions, surprises, package responsibilities, and design tradeoffs.
- Make small local experiments in throwaway branches.
- Avoid broad refactors while learning. Use narrow changes to understand one behavior at a time.
- When reading a package, answer: what owns this state, what is concurrent, what can fail, and what is the public contract?

## Setup Checklist

1. Confirm local toolchain versions:
   - `go version`
   - `node --version`
   - `npm --version`
2. Build the backend binaries:
   - `go build ./cmd/prometheus`
   - `go build ./cmd/promtool`
3. Run focused tests first:
   - `go test ./model/...`
   - `go test ./config/...`
   - `go test ./promql/parser/...`
4. Run broader tests when you have time:
   - `GO_ONLY=1 make test`
   - `make test-short`
5. Create a local scratch config from `documentation/examples/prometheus.yml` and run Prometheus locally.

## Month 1: Build The System Map

### Week 1: Entry Points And Runtime Shape

Read:

- `README.md`
- `CONTRIBUTING.md`
- `cmd/prometheus/main.go`
- `cmd/promtool/main.go`
- `documentation/examples/prometheus.yml`
- `docs/command-line/prometheus.md`
- `docs/configuration/configuration.md`

Questions to answer:

- How does Prometheus parse flags and config?
- What major services are started by `cmd/prometheus`?
- How are reloads, shutdown, logging, and metrics wired?
- What is the difference between `prometheus` and `promtool` responsibilities?

Exercises:

- Run Prometheus locally with a minimal config.
- Change one log message locally, rebuild, and confirm you understand the build path.
- Trace where `--config.file` is read and converted into runtime components.

Output:

- Write a short note: "Prometheus startup path in 15 steps."

### Week 2: Configuration, Labels, Relabeling, And Targets

Read:

- `config/`
- `model/labels/`
- `model/relabel/`
- `discovery/targetgroup/`
- `scrape/target.go`
- `scrape/manager.go`

Questions to answer:

- How does static config become scrape targets?
- Where are labels validated, normalized, and relabeled?
- What is the lifecycle of a target?
- Which data structures are optimized because they are used heavily?

Exercises:

- Add or modify a config test that covers relabeling behavior.
- Use `promtool check config` against valid and invalid configs.
- Trace one target from config file to active scrape loop.

Output:

- Draw or write the flow: config file -> service discovery -> target group -> relabeling -> scrape target.

### Week 3: Scraping And Ingestion Path

Read:

- `scrape/`
- `model/textparse/`
- `storage/interface.go`
- `tsdb/head.go`
- `tsdb/db.go`

Questions to answer:

- How is an HTTP scrape executed?
- How are text/openmetrics samples parsed?
- What storage interface does scraping write through?
- Where are append errors handled?
- What happens when labels are invalid, duplicate, stale, or out of order?

Exercises:

- Run `go test ./scrape/...`.
- Pick one scrape test and explain why each fixture exists.
- Add a temporary debug log or breakpoint in the append path and scrape a local endpoint.

Output:

- Write a sequence diagram in notes: scrape loop -> parser -> appender -> TSDB head.

### Week 4: PromQL Parser And Evaluation Basics

Read:

- `docs/querying/basics.md`
- `docs/querying/operators.md`
- `docs/querying/functions.md`
- `promql/parser/`
- `promql/engine.go`
- `promql/functions.go`
- `promql/promqltest/README.md`

Questions to answer:

- How is PromQL parsed into an AST?
- What are the main expression node types?
- How does the engine evaluate instant and range queries?
- How are function definitions connected to parser and docs?
- How do `promqltest` files describe expected behavior?

Exercises:

- Run `go test ./promql/parser/...`.
- Run `go test ./promql/...`.
- Add one local-only PromQL test case for an existing function.
- Pretty-print or inspect the AST for several queries.

Output:

- Write notes for three PromQL expressions: parse tree, evaluation type, and data access pattern.

## Month 2: Deepen Around Storage And Production Concerns

### Week 5: TSDB Head, WAL, Chunks, And Blocks

Read:

- `docs/storage.md`
- `tsdb/docs/`
- `tsdb/head.go`
- `tsdb/db.go`
- `tsdb/wlog/`
- `tsdb/chunks/`
- `tsdb/index/`

Questions to answer:

- What data lives in the head block?
- What is written to the WAL?
- When are chunks cut and persisted?
- How are blocks created, queried, and compacted?
- Where does Prometheus trade CPU, memory, and disk IO?

Exercises:

- Run focused TSDB tests, for example `go test ./tsdb/...`.
- Run one TSDB benchmark and inspect allocations:
  - `go test ./tsdb -run '^$' -bench . -benchmem`
- Trace one sample append into memory and WAL.

Output:

- Write a storage glossary: head, series, chunk, WAL, block, compaction, tombstone.

### Week 6: Rules, Alerts, And Notifications

Read:

- `rules/`
- `notifier/`
- `docs/configuration/recording_rules.md`
- `docs/configuration/alerting_rules.md`
- `template/`

Questions to answer:

- How are recording and alerting rules loaded?
- How often are rule groups evaluated?
- How does a PromQL result become an alert?
- What state is kept for pending and firing alerts?
- How are notifications sent to Alertmanager?

Exercises:

- Use `promtool check rules` with a simple rule file.
- Run `go test ./rules/... ./notifier/...`.
- Add a local rule test that demonstrates pending vs firing behavior.

Output:

- Write the flow: rule file -> rule group -> PromQL eval -> alert state -> notifier.

### Week 7: Remote Read, Remote Write, And Agent Mode

Read:

- `storage/remote/`
- `prompb/`
- `tsdb/agent/`
- `docs/querying/remote_read_api.md`
- `docs/prometheus_agent.md`

Questions to answer:

- What protobuf types define remote read/write?
- How does remote write queue data and retry failures?
- What does agent mode remove compared with full Prometheus?
- How are backpressure and failure handled?

Exercises:

- Run `go test ./storage/remote/... ./tsdb/agent/...`.
- Trace where remote write is configured and started.
- Inspect tests around retry, queueing, and sample conversion.

Output:

- Write a comparison: normal Prometheus storage path vs agent remote-write path.

### Week 8: Contribution Practice

Read:

- Recent merged PRs in an area you like.
- Issues labeled `low hanging fruit`.
- Tests near the area you want to change.

Practice tasks:

- Pick one small package and improve your local notes with package responsibilities.
- Find one small bug, doc ambiguity, missing test case, or cleanup candidate.
- Before changing code, write the expected PR title and release note block.
- Make a tiny branch and run the relevant focused tests.

Quality checklist:

- The change is narrow.
- Tests cover behavior, not implementation details.
- Each commit compiles independently.
- Commit is signed off with `git commit -s`.
- PR title follows `area: short description`.
- PR body includes:

```release-notes
NONE
```

Use a real release note instead of `NONE` only when the change is user-facing.

## Suggested Reading Order By Package

Start here:

1. `cmd/prometheus`
2. `config`
3. `model/labels`
4. `model/relabel`
5. `scrape`
6. `storage`
7. `tsdb`
8. `promql/parser`
9. `promql`
10. `rules`
11. `notifier`
12. `storage/remote`
13. `discovery`

## Small Project Ideas

- Add a missing test case for config validation.
- Add a PromQL parser or function test for an edge case.
- Improve a doc page where behavior is correct but unclear.
- Investigate an allocation-heavy benchmark and write notes before changing code.
- Reproduce a small open issue locally and document the exact failing path.

## Weekly Routine

- Spend one session reading implementation.
- Spend one session reading tests.
- Spend one session running or debugging the code.
- Spend one session writing notes or a tiny experiment.

At the end of each week, keep only the notes that still make sense after running
the code. Delete assumptions that turned out to be wrong.
