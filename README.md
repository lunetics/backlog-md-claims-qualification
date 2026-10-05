# Qualification of the claims feature of Backlog.md, revision 2c8d849

The claims feature of [lunetics/Backlog.md](https://github.com/lunetics/Backlog.md), branch `claims-native`, revision
`2c8d849`, was exercised end to end against real Git servers from three clients at once: a plain Git server reached
over `git://`, HTTP, HTTPS and OpenSSH, and a Gitea server over HTTP and its built-in SSH. The servers held pushes so
that writes really overlapped, a network proxy cut, stalled, slowed and throttled the connection, single clients ran
with a shifted clock, and the size went up to 1000 claims. All 339 checks in four stages passed, with Git 2.47.3,
OpenSSH 10.0p2, Gitea 1.24.7 and Bun 1.3.14. Linux containers only; macOS, Windows, NFS and other Git hosts and
versions were not tested.

The commits after `2c8d849` on the fork's branch change documentation only, so the claim code of the head is the
tested code. In a clone of the fork this command prints nothing:

```sh
git diff --stat 2c8d849 claims-native -- . ':!*.md'
```

## Where to read

| file | what it answers |
| --- | --- |
| [QUALIFICATION.md](QUALIFICATION.md) | what was tested and with which bounds, and what was not: servers, transports, clocks, sizes |
| [VERIFICATION.md](VERIFICATION.md) | how the feature was tested over the whole work: method, container runs per commit, the matrix runs, measurements beyond tests |
| [reports/](reports/README.md) | every check of the matrix, one report per storage format, with the stack and the evaluation explained |
| the four `.tar.zst` archives, `SHA256SUMS` | the runs themselves, every step and output, sanitised; how to check them is described below |
| [the fork](https://github.com/lunetics/Backlog.md/tree/claims-native) | the feature, its guide `CLAIMS.md`, the workflow proposals and the known limits |

## Checking the archives

Each bundle is one run of one stage of the qualification matrix on Backlog.md revision `2c8d849`, exactly as the
driver left it, with three kinds of change: the private key, the Gitea token and the Gitea admin password of that
run are not included (listed in `EXCLUDED.txt` inside the archive), the two Gitea secrets in
`compose-effective.yaml` and `container.json` read `<redacted>`, and host-specific names in the harness metadata
and logs are replaced by placeholders such as `<lane-dir>`, `<host>`, `<project>`. The step documents, the raw
outputs and the product tarball are unchanged.

| bundle | stage | official run | checks | sha256 |
| --- | --- | --- | --- | --- |
| `k20a-k20a-native3.tar.zst` | A | `k20a-native3` | 124 of 124 | `a756d045525ffea6894341d1bbe6763edc65049bc7f607a9b2196ca5c406edbf` |
| `k20b-k20b-native3.tar.zst` | B | `k20b-native3` | 115 of 115 | `aae28d4ca79862f90624e2badedf4fc5b331577aa961b6e6b1c4ba5068c34f7d` |
| `k20c-k20c-native3.tar.zst` | C | `k20c-native3` | 87 of 87 | `14cd87c1d40b91ace7afc7d94c54aa531b3a7c139f1e919f608f657291ae4937` |
| `k20c-size-k20c-size-native3.tar.zst` | size run | `k20c-size-native3` | 13 of 13 | `4cf0c6548c10eb6a67d40d8288b319b086e3cd3f445bc5ba35b03858edaf8050` |

### 1. Check a bundle

```sh
sha256sum -c SHA256SUMS
tar --zstd -xf k20a-k20a-native3.tar.zst
cd k20a-k20a-native3 && sha256sum -c --quiet CHECKSUMS.txt
```

`SHA256SUMS` lists the four bundles. `CHECKSUMS.txt` inside each archive lists every file of that archive.

### 2. What an archive contains

| file | content |
| --- | --- |
| `tests.log` | the run's record: one `K20` line per step, one `K20-ROW` per check, one `K20-RUN` per run, `K20-LEAK` for the leak scan |
| `docs.jsonl`, `outputs/` | the step documents and the raw product outputs the checks were read from |
| `source.tar.gz` | the product tree the client image was built from; its sha256 is the `source.archiveSha256` field of the `K20-RUN` line |
| `EVALUATION.txt` | the output of the frozen evaluator for this archive (section 3) |
| `tests.xml`, `tests-1.xml`, `tests-2.xml` | the test results in JUnit form, one file per test process |
| `exit-code`, `driver-exit-code`, `source.sha256` | the run's exit code, the driver's exit code, the sha256 of `source.tar.gz` as the driver recorded it |
| `driver.log`, `compose.log`, `containers.log`, `*.log`, `*.json`, `compose-effective.yaml` | driver, compose and container logs, image and container metadata, the effective compose file |
| `pki/ca-cert.pem`, `pki/client_id_ed25519.pub`, `pki/known_hosts` | the run's private CA certificate, the client's public key and the host keys of the stack's own containers |
| `EXCLUDED.txt`, `CHECKSUMS.txt` | the files left out of the archive; the checksums of the files in it |

```sh
sha256sum k20a-k20a-native3/source.tar.gz
grep -m1 '^K20-RUN ' k20a-k20a-native3/tests.log | cut -d' ' -f2- | python3 -c 'import json,sys; print(json.load(sys.stdin)["source"]["archiveSha256"])'
```

The product tarball is the same in all four archives (sha256 `0269748e8cbbfb70eb2543624b53a678cdbab5ba88e686a4d3d406dbd8e1342e`).

### 3. The evaluation

`EVALUATION.txt` is the output of the evaluator that read the archive: the evidence counts, the versions of the
stack, the hashes of the ten frozen scenario files against their frozen values, the comparison of every file in
`source.tar.gz` by Git blob hash against revision `2c8d849` (`616 archived, 616 in tree, 0 differ`), the leak scan
with its canary, the test results, the coverage of the expected checks, the clock check of the stages with shifted
clocks, the size figures, and the verdict. Every archive ends `problems: 0`, `VERDICT: GREEN`. The evaluator is
identified in the `K20-RUN` line by its sha256, `c44b87ff63364736be8a08bf059685739fd04eaa179ce5e68d82486f157c9221`;
the evaluator itself is not part of the bundles.

### 4. What was left out or replaced, and why

| item | treatment |
| --- | --- |
| `pki/client_id_ed25519` | left out — the run's SSH private key |
| `gitea/token`, `gitea/admin-password` | left out — the run's Gitea API token and admin password |
| `needles/*/ssh-key-open-perms` (stage B only) | left out — private keys the leak checks planted on purpose; the leak scan's needle list in `tests.log` keeps their names |
| `GITEA_ADMIN_PASSWORD`, `SECRET_KEY` values in `compose-effective.yaml`, `container.json` | `<redacted>` |
| lane directory, compose project, image namespace, label prefix, lane and host names | placeholders `<lane-dir>`, `<compose-project>`, `<image-namespace>`, `<label-prefix>`, `<lane>`, `<image-build-lane>`, `<other-lane>`, `<project>`, `<lanes-root>`, `<desktop-host>`, `<host>`, `<org>`, `<agent>` |

Scans run on the archives before bundling: a secret scanner reports only the GPG key fingerprint of the Python
base image (`GPG_KEY` in `server-image.json` and `container.json`, a public key) and, in the stage C archive, one
path (`sizes/gitea/ssh/blob/100`) in `tests.log` — both false positives. A fixed-string scan with the left-out
values themselves (key body lines, token, admin password, the two compose secrets) finds nothing in the archives
and finds every value in the unpublished originals.
