# AGENTS.md — operating instructions for THIS kit

> Read this first, every session. It tells you what this repo is, how to pull, what to read, how to
> report, and the rules that keep the serve alive. It does **not** duplicate facts — those live in the
> files named below.

## 0. TL;DR

- This is a **deployment recipe**, not a library: `./start.sh` sets up both DGX Sparks and serves
  DeepSeek-V4.1-Flash-1017 EXL3 2.9 bpw across them through an OpenAI-compatible API. `README.md` is the
  recipe. **`LOCAL.md` is our delta** — read it before touching anything.
- **Never** change a value or restart the serve without first reading `LOCAL.md` §2 (current values) and
  §5 (change log) so you do not repeat a rejected experiment.
- The serve is **shared infrastructure**. Treat restarts as disruptive, run long commands detached, and
  verify boot with `LOCAL.md` §4 before declaring success.

## 1. What to read, in order

| Read | Why | Size |
|---|---|---|
| `LOCAL.md` | **our** current `.env` values, why, local code edits, rejected trials, boot checks. Single source of truth for this kit. | small |
| `README.md` | the recipe: every knob, all measurements. Authoritative over LOCAL.md on conflicts. | ~25 KB |
| `scripts/config.sh` | the shipped defaults `start.sh` assumes (the recipe's "`.env.example`"). | ~15 KB |
| `CHANGELOG.md` | release notes, newest first. | small |
| `scripts/local.sh.example` | the optional bash override file (we do **not** use it; our values are in `.env`). | tiny |

`LOCAL.md` is the entry point; consult README only for the knob you are about to change. **If `LOCAL.md`
and `README.md`/`config.sh` disagree, the recipe wins unless `LOCAL.md` §1/§2 records the deviation.**

## 2. The config layers (know the difference)

| File | Tracked? | Role |
|---|---|---|
| `scripts/config.sh` | yes | shipped defaults + all knob names/effects |
| `.env` | ours, untracked | the live config (plain `KEY=value`, read, never executed) |
| `scripts/local.sh` | ours, untracked | optional bash overrides, sourced; **wins over `.env`**; absent here |
| environment | — | `PORT=9000 ./start.sh` wins over everything |

Precedence at boot: **environment > `scripts/local.sh` (if present) > `./.env` > `scripts/config.sh`
defaults** (`config.sh:1`). `.env` only fills keys that nothing above it set, so a `.env` line for a key
`local.sh` also sets is ignored. `.env` is **not tracked**, so `git pull` never updates it and it silently
drifts from `config.sh` defaults. When you pull, diff and report drift (AGENTS §4).

Two `.env`-only caveats:
- The parser is `KEY=value` lines (optional `export`, optional quotes). It is **not** shell.
- `API_KEY` **is** a recipe setting here (unlike the GLM kit): `API_KEY=1` makes rank 0 require
  `Authorization: Bearer <key>` on every request but `/health`. See `LOCAL.md` §2/§7.

## 3. Standard pull workflow

When the user says *"pull latest"* / *"pull changes"*:

1. `git fetch --all --prune`, then `git log --oneline HEAD..origin/main` and
   `git diff --stat HEAD..origin/main` — **read before pulling.**
2. If on `main` and clean of intended changes, `git pull --ff-only`. Do not merge/commit.
3. Explain **what the new commits are about** (format: AGENTS §5).
4. Diff `.env` against `scripts/config.sh` defaults for new/removed/renamed keys and changed defaults;
   report drift.
5. **Check for conflicts with our §1 code edits** — currently **none**. If a future edit is added there,
   a pull touching it can clobber it; say so and offer to re-apply.
6. If `LOCAL.md` now contradicts the code, say so and propose the edit.
7. If a restart/rebuild is implied (patches, image hash, Dockerfile, config), **tell** the user; do not
   restart without being asked.
8. **Leftover sweep** (AGENTS §9/§10): once the pull (and any prepare/restart it leads to) is done, roll
   `LOCAL.md` §5 to the recent window (§8.4), delete wording that has gone stale (future tense like "will
   serve at the next start" after it has booted; contradictions from concurrent edits), and check both
   Sparks' disk after a build change; if the image hash changed, sweep old `native-spill-*` namespaces on
   both Sparks (stopped, §9).

Untracked files you will see normally: `.env`, `LOCAL.md`, `AGENTS.md`. Do not commit them unless asked.

## 4. Comparing `.env` to recipe defaults

The recipe has no `.env.example`; its defaults are the `${VAR:-default}` lines in `scripts/config.sh`.

```bash
# every settable key is a line like NAME="${NAME:-value}" in config.sh
grep -oE '^[A-Za-z_][A-Za-z0-9_]*=' scripts/config.sh | tr -d = | sort -u > /tmp/k_cfg
grep -E '^[A-Za-z_][A-Za-z0-9_]*=' .env | sed 's/=.*//' | sort -u > /tmp/k_env
comm -23 /tmp/k_env /tmp/k_cfg     # keys in .env with no config.sh knob — suspect
comm -13 /tmp/k_env /tmp/k_cfg     # knobs we do not set — recipe default applies (fine, but report)
```

A missing key in `.env` is not a bug: `config.sh` supplies its documented default. But report it, because
it means the setting is invisible/unmanaged. Classify each difference as **intentional** (recorded in
`LOCAL.md` §2) or **drift to fix**.

## 5. How to report a pull (format the user expects)

Keep it short and scannable:

1. **Range**: `old_sha → new_sha`, commit count, PR numbers/titles.
2. **What it is about**: per PR, one or two lines of *substance*, not the commit message.
3. **Impact on this kit**: does it change our values? break the §1 edits? need a restart/rebuild?
4. **`.env` drift**: new/renamed knob names, changed defaults, stale `.env` comments.
5. **Leftovers**: stale wording/contradictions fixed, §5 rolled to the recent window, disk checked on both Sparks.
6. **`LOCAL.md` updates**: what you propose to add/edit (do it unless told otherwise).

## 6. Running the launcher — rules

- **Detached, always** for anything that may outlive the current command (`stop`, `start`, `restart`,
  rebuild, `prepare.sh`):
  ```bash
  setsid nohup ./start.sh restart > /tmp/opencode/restart.log 2>&1 < /dev/null &
  ```
  Killing the launcher mid `docker save | ssh docker load` (or mid worker copy) leaves a partial load.
- **First run is long**: `prepare.sh` pulls/builds the image on both Sparks, then writes two ~99 GiB
  weight files and copies Engram's 190 GB to the worker. Later starts ~35 s to load. `FOREGROUND=1 ./start.sh`
  stays attached to rank 0's log (systemd style) and exits with its code.
- `./start.sh` (bare) **leaves a running server alone**; use `./start.sh restart` to apply new settings.
  A typo in the arguments aborts *before* the running server is stopped.
- `./stop.sh` stops both ranks and frees GPU memory.
- Extra args go to rank 0's `tensorfold-native serve` after the defaults and win:
  `./start.sh restart --max-tokens 16384`.
- Boot gate is **`/health`**, but do not call a boot healthy on `/health` alone — run the `LOCAL.md` §4
  checks and the temp-0 smoke test. Worker log: `ssh ASUS-Ascent-GX10 docker logs --tail 50 dsv41-1017-exl3-tf`.

## 7. Measuring / changing config

- Default correctness gate: `python3 tools/toolcheck.py` (typed tool call). **Do not run
  `python3 tools/bench.py --suite needle` (long-context needle) by default** — it is slow (~2 min at
  122K tokens); run it only when asked or when long-context correctness is in question.
  `tools/validate.py` (text, thinking, tools, image) is likewise opt-in. All read `API_URL` (or `PORT`)
  and are safe against a live server.
- **Our served id is `deepseek-v4.1-flash`, but `tools/` default to `MODEL=DeepSeek-V4.1-Flash-1017`.**
  Always pass `MODEL=deepseek-v4.1-flash` (recorded in `LOCAL.md` §2/§7).
- Performance: use [sparkDash](https://github.com/MiaAI-Lab/sparkDash) through the API from another
  machine — the recipe's own numbers were taken that way (README *Status*). Do not invent a second
  harness.
- Never trust a prefill number where the prompt came from cache. Give a unique nonce per cold run.
- Changing a knob is not "done" until: the serve is up, the `LOCAL.md` §4 checks pass, the smoke test
  passes, and `LOCAL.md` records what changed and why (AGENTS §8).

## 8. Keeping `LOCAL.md` current — update it yourself, do not wait to be asked

`LOCAL.md` is the kit's memory. **Every time one of the triggers below happens, edit `LOCAL.md` in the
same turn** — do not ask whether to, and do not leave it for later. It is untracked, so edits are cheap.

### 8.1 Trigger → where to write

All `§` refs are **`LOCAL.md`** sections (AGENTS is always written out).

| Trigger (what just happened) | Update | What to write |
|---|---|---|
| A `.env` value adopted/changed | §2 row | our value, recipe default, *why*, and the measurement/date if any |
| A `.env` value changed but not a standing deviation (e.g. reverted) | §5 row | `from → to`, date, reason; then fix §2 to match reality |
| A change to a **tracked repo file** (`start.sh`, `scripts/*`, `zig/patches/*`) | §1 row **and** §5 row | file, what changed, whether it needs a rebuild |
| An experiment **adopted** | §2 row **and** §5 row | the new value, the evidence that justified it |
| An experiment **rejected** | §6.2 **and** §5 row | what it was, the overrides, the A/B numbers, verdict, rollback |
| A benchmark/measurement run | §6.1 (or the relevant §6.x) | medians + the config they were taken on; never a number without its config |
| A new investigation that settles a question | §3 entry | the question, the answer, "do not redo" |
| A new boot check or smoke test step | §4 | the exact command + expected output |
| A `git pull` that changes our defaults/behavior | §5 pull-review note (+ §2 if our values change) | PR numbers, substance, impact, `.env` drift |
| A pull/prepare/restart just finished | §5 roll + a sweep | fold §5 to the recent window (§8.4); delete stale wording; check disk on both Sparks (§9) |
| A knob we deliberately leave **off** | §2 "deliberately not enabled" | the knob and why |
| `LOCAL.md` contradicts `README`/the code | fix `LOCAL.md` (recipe wins) | reconcile, or record the sanctioned deviation in §2 |

### 8.2 How to write it (style)

- **Delta, not a README fork.** State *our* value, *why*, and *when it was tried / what happened*. Never
  duplicate README's knob documentation.
- **Every number carries its config** (streams, window, drafts, build/image stamp, date).
- **Dates + SHAs.** Name the repo SHA for code claims, the boot/image stamp for measurements.
- **Rejected things stay rejected**, with the reason, in §6.
- Keep the **header History line** (≤ ~600 characters, §8.4) and the **§5 changelog** advancing whenever anything material changes.
- If a `.env` knob has a **stale comment**, fix the comment too and note it.

### 8.3 When NOT to touch `LOCAL.md`

- A pure `git pull` with no change to our values, behavior, or a settled question → report only.
- Speculation / untested ideas → do not enter them as if measured. Mark them **unverified** if worth
  remembering.

### 8.4 Keep it compact — every edit is also a pruning pass

`LOCAL.md` must stay a scannable delta, **not a growing wall of text**. Whenever you edit it, compact it in
the same turn:

- **Fold prose into tables/lines.** A finished pull review becomes **one row** in §5 (date · `old→new` SHA ·
  PRs · substance · impact · `.env` drift). Do not keep separate multi-paragraph review blocks.
- **Newer supersedes older.** When a value/boot/default changes, update the §2/§3/§6 entry in place; keep the
  prior value only as a terse line (e.g. "previous: …"), never a second full paragraph.
- **Tighten, don't re-document.** One line per fact; delete restated README knob docs, duplicated commands,
  and detail already implied by a `§` reference. Prefer `see §3.5` over re-explaining it.
- **§5 is a rolling log, not an archive.** Keep only the recent window (roughly the last ~2–3 releases, ~2
  weeks) plus every entry that changed **our** values, behavior, or boot numbers. Upstream release detail
  belongs to the repo's `CHANGELOG.md` — do **not** re-copy it.
- The **header History line** is a rolling one-liner per session/day, not a diary: **≤ ~600 characters**, one clause per milestone (`date · build/change · SHA`), at most ~10 milestones; **no boot times, MemAvailable or per-boot numbers** (those live in §5/§6.1). When it grows past that, drop or fold the oldest clauses — §5 keeps the detail.
- **Budget:** keep `LOCAL.md` under ~155 lines, §5 at ~6 rows and the History line under ~600 characters (fold older entries into the Earlier line).
  If it exceeded that after your update, you did not compact enough — prune before finishing. (Facts still win
  over the budget: never drop a value, date, SHA, settled finding, or rejected trial to hit it — roll old ones
  into the summary line instead.)

## 9. Hard rules

- **Do not commit** unless explicitly asked. `LOCAL.md`, `AGENTS.md`, `.env` stay untracked.
- **Do not restart, stop, or rebuild the serve** without the user asking. It is a live endpoint.
- **Do not re-run a settled investigation** in `LOCAL.md` §3 — the answer is recorded.
- **Do not enable** anything in the `LOCAL.md` §2 "deliberately not enabled" list.
- **Keep secrets out of git**: `API_KEY`, `HF_TOKEN`, `GHCR_TOKEN` live only in `.env` (or the shell).
- **Guard the §1 code edits** (currently none): if a pull removes one, re-apply it or record the removal.
- Keep `LOCAL.md` as a **delta + changelog**, not a README fork.

### NFS and worker-disk housekeeping — after any build or model change

- **NFS is required by this recipe.** The head exports `~/.cache/huggingface` read-only to the worker's CX7
  address (`10.100.96.2`); the worker mounts it as the docker volume `dsv41-1017-exl3-hf` at `/hf`. If the
  export is missing the worker cannot see the checkpoint and `prepare.sh` dies at step 6. Re-export:
  `sudo exportfs -ra`; verify with `exportfs -v`.
- **Do not delete the head's base-model dir** `hub/models--deepseek-ai--DeepSeek-V4.1-Flash-1017`. Rank 0
  reads Engram's tables from it at run time (`HEAD_ENGRAM_DIR` is empty). The worker has its own copy at
  `~/.cache/dsv41-tensorfold/engram-1bae0e10`.
- The EXL3 checkpoint snapshot (**196 GiB** of unique blobs on the head; `du -shL` shows 213 GiB because the
  `.incomplete` leftovers were counted too) is **only needed by `prepare.sh`**. It is safe to delete once settled,
  but re-preparing then re-downloads 196 GiB — record the decision in `LOCAL.md` §3.
- After a build change, check free space on **both** Sparks (`df -h ~`); the worker is the tighter disk.
  Delete `*.incomplete` leftovers under `~/.cache/huggingface/hub/blobs/` — they are dead downloads.
- **Spill stores are keyed by the image/patch hash.** A rebuild changes the derived tag, so the store opens a new
  `native-spill-<compat>` namespace and starts empty; the old namespace is dead weight (at 128 GiB each, two stores
  can take 256 GiB a Spark). The engine prunes older namespaces when it opens the store (`keep_builds=1`: it keeps
  one previous), but to reclaim all, **stop both ranks first** (`./stop.sh` gives the files back to you), then on
  **both** the head and the worker `rm -rf <SPILL_DIR>/native-spill-*` (or delete only the older ones). Never delete
  spill files under a live rank (`owner.lock`). If the image hash did not change, leave the store alone — it is warm.
  If the running container's image differs from the tree's derived tag, say so; stop only when the user asks.
  **Currently off:** spill fails `FingerprintMismatch` in `engram` (LOCAL.md §6.2); do not re-enable until that is fixed.

## 10. Definition of done (for any task)

1. Code/config change made and syntax-checked (`bash -n start.sh scripts/*.sh`, `python3 -c "import ast"`
   on `tools/*.py`, `shellcheck` if available).
2. Serve verified per AGENTS §6 if a restart was involved, else explicitly noted as not restarted.
3. `LOCAL.md` updated per AGENTS §8 (the trigger table) **and compacted** per AGENTS §8.4 — proactively, not
   on request.
4. **No leftovers** (AGENTS §9): `LOCAL.md` has no stale wording (future tense that has passed, contradictions
   from concurrent edits), §5 is rolled to the recent window, disk was checked on both Sparks after a
   build change, and old `native-spill-*` namespaces were swept on both when the image hash changed
   (stopped, §9).
5. Report given in the AGENTS §5 format, with the exact commands used.
