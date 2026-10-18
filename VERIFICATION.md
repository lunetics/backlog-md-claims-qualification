# Claims: verification

How the claims feature was tested, what the numbers are, and which of the evidence is where.

## Method

Every behaviour was built test-first in its own loop: a written contract; test rows written by a separate test
author; a RED run of those rows measured on the previous head; the implementation by a third party; a GREEN run;
guard mutants per contract axis against the real implementation, so that each row is shown to catch exactly the
defect it names; a review round; then container runs of the whole suite per commit. One contract and one run
ledger per loop, 27 contracts and 25 ledgers in all. The contracts and ledgers are our working records and are
not published; the tests are in the fork, under `src/test/claim-*.test.ts`.

## Container runs per commit

The test suite ran in a container built from `oven/bun:1.3.14` with a read-only root filesystem, no network, one
CPU, 2 GB of memory and 256 processes, in four modes: the CLI suites, the storage adapter suites with their Git
servers, the configuration suites, and upstream's own CI line. 247 archived container runs over the work, of
which 154 ended with exit code 0 and 92 with a non-zero code (intended RED runs and harness iterations included;
one archive has no exit code).

The published head measures, per commit of the series on top of upstream `69e7b15`:

| commit | CLI suites (pass / fail) | adapter suites (pass / skip / fail) |
| --- | --- | --- |
| upstream test fixes | 509 / 0 | not applicable |
| git-backed storage adapters | 509 / 0 | 163 / 1 / 0 |
| journal, query, rights, execution, config | 654 / 0 | 617 / 1 / 0 |
| claim CLI, surface core, user documents | 962 / 0 | 925 / 1 / 0 |
| claim tools over MCP | 1016 / 0 | 989 / 1 / 0 |
| claim owner in the browser | 1049 / 0 | 1022 / 1 / 0 |
| qualification record | 1049 / 0 | 1022 / 1 / 0 |
| Git's own result on a held pipe | 1049 / 0 | 1025 / 1 / 0 |
| internal test rows named by topic | 1049 / 0 | 1025 / 1 / 0 |

Every archive carries the source it tested, compared by Git blob hash against the commit; types and formatting
were clean in every run.

## Upstream's CI line

Upstream's own CI workflow ran unchanged on a private mirror of this branch (ubuntu, macOS and Windows jobs,
compile-and-smoke tests, the nix package). The ubuntu job runs the full profile with two parallel workers in
isolation mode and a 10 s timeout per test.

| run on | ubuntu full profile (pass / skip / fail) | other jobs |
| --- | --- | --- |
| the measured revision | 3578 / 9 / 1 | macOS green, Windows red |
| the qualification record | 3576 / 9 / 3 | macOS green, Windows red |
| the first fixture commit | 3575 / 9 / 4 | macOS green, Windows red |
| the evidence commit | 3579 / 9 / 0 | all green, Windows included |
| the second fixture commit | 3579 / 9 / 0 | macOS green, Windows red |
| the documentation commits | 3579 / 9 / 0 | macOS green, Windows red |
| the clean-up commit | 3579 / 9 / 0 | all green, Windows included |
| the verification commit | 3579 / 9 / 0 | macOS green; Windows as described below |
| the held-pipe fix | 3582 / 9 / 0 | macOS green; ubuntu green on its second run after one upstream MCP test failed once; Windows red, then green |
| internal test rows named by topic | 3582 / 9 / 0 | macOS green; Windows as described below |
| the published head | 3582 / 9 / 0 | macOS green; Windows as described below |

The Windows job flips between green and red at the 10 s test timeout. On the revisions of this series that ran on
CI it was green on some runs and red on most; every red run failed only upstream tests at that timeout, in files
this series does not change, most often in the `Auto-commit configuration` and `Task ID Generation with Archives`
suites (`src/test/auto-commit.test.ts`, `src/test/id-generation.test.ts`). The base revision `69e7b15`, run on the
same mirror's Windows job, failed the same suites and one more test. The ubuntu job is the acceptance line of this
series.

The failures in the first three ubuntu runs were four tests of the claim test fixtures, none of the product: a
proxy that classified a connection by its first data event although Git writes a packet's length and payload in
two writes, and three rows that cut the connection after the server had already sent its status report. Both
fixtures were corrected in the two test-only commits before the published head; the rows carry positive controls
now, and the storage adapter suite, the mutation resolution suite and the git primitives suite ran green in the
container (132 / 1 / 0, 21 / 0, 25 / 0) and under the CI line's own shape (two CPUs, two workers: 0 failing claim
tests). One ubuntu run also showed a single upstream TUI test fail that the series does not touch.

The same CI line in a two-CPU container without network and without a terminal reproduces the claim results and
leaves 25 upstream tests failing that need what the container withholds (an editor, ssh-keygen, platform launcher
packages, a terminal for init, a writable parent directory); that set is identical across all our runs and is not
part of the claims work.

## Qualification against real Git servers

[QUALIFICATION.md](QUALIFICATION.md) is the summary with bounds. The method:

- One compose stack per run: a plain Git server (git daemon on `git://`, `git http-backend` behind CGI on
  `http://` and `https://` with a private CA, OpenSSH 10.0p2 on a high port) and Gitea 1.24.7 rootless (HTTP and
  its built-in SSH server), both fronted by toxiproxy 2.11.0 where a case injects network faults; three client
  containers, each running the shipped `backlog` CLI in its own project directory, and for the parity rows
  `backlog mcp start`; a driver that runs the frozen scenario files. libfaketime 0.9.10 shifts single clients.
  Git 2.47.3 on the server and the clients, Bun 1.3.14.
- Six server and transport combinations times three storage formats, 18 combinations. Four stages: stage A with
  124 checks (smoke per combination, acquire races, transfer, formats, preflight), stage B with 115 (lost replies and
  network loss, batch reclaim, CLI and MCP parity, secrets in output), stage C with 87 (clock skew and late writes,
  restore and new epochs, incompatible data, `enabled: false`, sizes up to 100 claims), the size run with 13 (1000
  claims, a bandwidth-limited link); 339 checks per full matrix.
- A stage counted only after a guard run, with the harness deliberately broken in one place, turned exactly the
  predicted checks red and nothing else: endpoints split across two repositories (93 checks red), the push gate
  bypassed (54), fault injection switched off (54), the clock shift switched off (9).
- The matrix ran five times: twice on the revision before the history was cleaned up (`k20a-green1`,
  `k20b-green2`, `k20c-final1`, `k20c-size3`; then `k20a-rep2`, `k20b-rep4`, `k20c-rep1`), once on `cb45f9f`
  (`k20a-native1`, `k20b-native1`, `k20c-native1`, `k20c-size-native1`), once on `bce2acc` (`k20a-native2`,
  `k20b-native2`, `k20c-native2`, `k20c-size-native2`) and once on the published revision `2c8d849`
  (`k20a-native3`, `k20b-native3`, `k20c-native3`, `k20c-size-native3`). Every official run ended green with
  0 findings; on the published revision every check kept its status and completeness. 22 official runs,
  2132 checks; 57 archived qualification runs in all, the rest being dry runs, guard runs and harness iterations.
- Evaluation: a frozen evaluator script whose sha256 the driver records in every archive and checks at
  evaluation; ten frozen scenario files identified by hash; the product in every archive compared by blob hash
  against the revision; a leak scan with eight needles over every captured output; coverage of every expected
  check for every combination; harness faults classified as problems, never as findings.

The three per-format reports with every row, combination, injection and measured value are in
[reports/](reports/README.md), with a README on the stack and how the archives were evaluated. The archives
themselves (logs, step documents, raw outputs) of the four runs on the published revision are in this repository,
sanitised as described in its README.

## Measurements beyond tests

- Concurrent claim reads in one checkout: a diagnosis with system-call traces and counter-probes found that a
  sibling read's temporary ref, seen with Git's strict ref checking, failed a claim fetch in 2 to 9 of 1000
  real rows; with the fix 0 of 1000. The fix is in the storage commit.
- The other direction, a user's own Git command in the same checkout during a claim read: measured twice, with
  10 to 5010 refs, loose and packed; the result is in [the fork's guide "Claims known limits"](https://github.com/lunetics/Backlog.md/blob/claims-native/backlog/docs/doc-6%20-%20Claims-known-limits.md).

## Rebases onto upstream

The tables above describe the series as published on 2026-10-06: head `9a08b07` on upstream `69e7b15`. Since then
the branch is rebased onto newer upstream releases. Each rebase is measured on the same container harness before it
is published. The qualification matrix against real Git servers is not rerun: it measured the claim modules, and a
rebase leaves them byte-identical (the command in the [README](README.md) shows it).

### 2026-10-15: onto upstream `d306c24`, head `f008b14`

- One conflict, in the import lines of `src/cli.ts`. 28 of the 29 commit patches are unchanged; the 29th differs only
  in that import line and its context.
- Types and formatting are clean on each of the 30 commits, the 29 rebased ones and one documentation commit.
- CLI suites 1057 pass, 0 fail; adapter suites 1025 pass, 1 skip, 0 fail; no leftover temporary directories, journal
  entries or processes.
- Upstream's CI line in a container: 3624 test cases with 25 failures, the same 25 cases that fail on upstream
  `d306c24` alone in the same container; the DOM pass 339 pass, 0 fail.
- The private CI mirror: ubuntu 3610 pass, 14 skip, 0 fail, the DOM pass 339 pass; macOS and Windows green.

### 2026-10-18: onto upstream `75e9e972` (v1.54.0 and one commit after it), head `9d4a573c`

- Three conflicts: the import lines of `src/cli.ts`, the import lines and the task routes of `src/server/index.ts`, and
  `src/test/web-dependency-cleanup-notice.test.tsx`, whose fix for a leaking retry upstream adopted in its own words;
  the series' first commit now carries only the exit-code fix of `src/test/list-window.test.ts`.
- Upstream's v1.54.0 replaced task completion with one archive flow (its task BACK-719): the task corpus names its
  off-board records `archivedTasks` instead of `completedTasks`, and `FileSystem.completeTask` is gone. One line of
  claim code reads the renamed field (`src/claims/surface/index.ts`, the corpus of `claim install-epoch`, which
  therefore creates refs for archived tickets too: they keep their IDs under the archive flow, as completed ones did
  before), and nine test files set up archived tasks where they set up completed ones: `claim-browser-view`,
  `claim-emergency`, `claim-epoch`, `claim-next`, `claim-next-cli`, `claim-next-git`, `claim-reclaim-git`,
  `claim-surface-administration` and `claim-surface-git`. The readiness fixture of `claim-next` holds its archived prerequisite in the final status,
  because under the archive flow an archived prerequisite satisfies its dependents only there.
- Upstream's v1.54.0 also stopped record creation from overwriting an existing file (its task BACK-732): a task saved
  without the path it was loaded from is a new record. The twin fixture of `claim-mcp-git` reloads a task before it
  rewrites its status to Done, where it wrote a fresh task object before; the first container run and the first mirror
  run of this rebase found this one failure, mcp-g11, and nothing else.
- The other 70 files the series adds are byte-identical to `2c8d849`.
- 25 of the 30 commit patches are unchanged; four differ in the lines above and their context, one in context only.
  A 31st commit names both upstream changes in the README's tested-code sentence.
- Types (`bun --check`, Bun 1.4.3) and formatting are clean on each of the 31 commits.
- CLI suites 1095 pass, 0 fail; adapter suites 1025 pass, 1 skip, 0 fail; no leftover temporary directories, journal
  entries or processes. The lane image runs Bun 1.4.3 now, the version upstream's CI sets.
- Upstream's CI line in a container: 4181 test cases with 25 failures, the same 25 cases that fail on upstream
  `75e9e972` alone in the same container (closed stdin, launcher, editor, TUI and MCP-stdio cases); the DOM pass
  411 pass, 0 fail; no leftover temporary directories, journal entries or processes.
- The private CI mirror on the head: ubuntu 4167 pass, 14 skip, 0 fail, the DOM pass 411 pass; macOS green; the
  Windows unit job fails upstream's own archive cases of `id-generation.test.ts` and `auto-commit.test.ts` at the
  10 s timeout in both attempts (four, then five cases). The same cases pass in upstream's CI on `75e9e972` at
  5.7 to 9.5 s, and the mirror's Windows runner is slower throughout: 114 and 138 s for the job against
  upstream's 66 s, and 2.4 times upstream's duration on cases that touch no archive. Upstream `75e9e972` alone,
  pushed to the mirror's main, fails the same four cases on that runner at the same timeout (474 pass, 4 fail).

## Calendar

2026-09-24 to 2026-10-05, twelve consecutive days with dated ledger entries, 304 archived container runs in all.
