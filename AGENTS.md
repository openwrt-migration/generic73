# Agent working notes — OpenWrt kernel migration

Focus: move targets from an old LTS kernel (currently `6.18`) to a new one
(currently `7.3`) by porting and refreshing target patches/configs. `generic/`
and the base target (`x86`) are the reference. The current 6.18 → 7.3 run is
complete for all target branches; these notes are also intended for re-deriving
individual targets or repeating the process for a future kernel (e.g. 7.4).
Plan, status and known-broken packages live in `KERNEL-<new>.md`.

Local overrides: this file wins over the agent's generic defaults on conflict.

## Working method
- Cite `file:line`; never cite your own summary. Re-read the source before
  asserting what a patch or register does.
- Attribute before confessing: check pristine upstream / `git show` before
  blaming a patch — offsets and patch-size deltas lie.
- When build output contradicts the sources on disk, suspect stale
  `build_dir/` first: clean the affected package and rebuild before theorizing.
- Before writing a patch, enumerate 2-3 candidate mechanisms and pick the
  smallest diff that carries the design.
- `build_dir/` is not a stable reference. Copy files out before analysing and
  re-check mtime/diff before quoting a line number read minutes earlier.
- NEVER build scratch kernel trees with `cp -al` / hardlinks: a single in-place
  write silently corrupts the shared "pristine" source and invalidates every
  later patch/verify result. Use `cp -a`. Re-extract from `dl/linux-*.tar.xz`
  and `diff -rq` against a fresh extraction if a base tree is ever suspect.
- Never silently replace a user artifact: back it up, say where, say why.
- Enumerate small sets completely; sampling is for large sets only, then state
  the basis.
- Scratch work goes in `/tmp/opencode` (pre-approved). `/tmp` is a 7.9G tmpfs
  and a full kernel tree is ~1.8G: clean scratch trees after each target, and
  remove `build_dir/toolchain-*` (~4G each) after `staging_dir/toolchain-*`
  is installed, or parallel targets will exhaust disk/inodes. Keep at least
  5G free on `/`; stop and clean before the next build if below that.
- Before any history rewrite, save old branch heads with names (e.g.
  `git rev-parse <branch> > /tmp/opencode/heads.txt`). `git reflog` tail is
  not a reliable "before" baseline.

## Porting patches (the main work)
- NEVER hand-write or hand-edit patch hunks. Edit the code (or a scratch copy),
  then generate hunks with canonical `diff -u` / `git diff`.
- A patch's base is "the tree produced by all preceding patches". Reconstruct
  that in `/tmp`; never generate against a tree you did not produce, never
  against the live `build_dir/`. Concretely the base is: pristine `<new>` +
  `target/linux/generic/files/` + generic backport/pending/hack-<new> patches +
  the target's `files/` and `files-<new>/` (OpenWrt copies these into the kernel
  tree BEFORE patches — see `include/quilt.mk:93-96`). A scan that omits them
  reports false pass/fail and disagrees with real builds.
- Prefer zero offsets. An offset means the base is stale: find the preceding
  patch that moved the target, regenerate, re-measure the series. Refresh to a
  fixed point: refreshing an early patch shifts context for later ones, so
  replay the whole series until no patch shows offset/fuzz; large series can
  need several full passes. A clean replay is necessary but NOT sufficient —
  see below. If an explicit override tolerates offsets in untouched patches,
  record it; fuzz and FAILED remain unacceptable.
- After any edit, verify hunk counts from the bodies, run `patch --dry-run`
  (fuzz/FAILED is never acceptable), and compute the diffstat (`--stat`).
  Zero offset/fuzz does not prove the rebase is correct: alignment-based
  rebases can apply cleanly yet insert code in the wrong function, the wrong
  part of a function, or outside its function. Only a compile catches that
  class of error.
- A diffstat/body mismatch (e.g. `5 insertions(+), 5 deletions(-)` but only
  five `+` lines) is a strong signal a hunk was dropped in the rebase. Compare
  the patch with its old-kernel counterpart and restore the missing code lines
  before declaring the replay clean.
- A patch file carries the diff only; rationale belongs in the commit. Keep
  in-code comments that carry a non-obvious invariant.
- Run `scripts/checkpatch.pl --no-tree` on new/changed hunks; clear ERRORs
  (OpenWrt SOB/header noise excepted). `Documentation/process/coding-style.rst`
  is the style authority. Some rebased patches inherit byte-identical upstream
  style ERRORs; compare ERROR counts against the old-kernel counterpart, not
  just raw totals, to separate inherited noise from newly introduced errors.
- Patch files end with a newline. `diff`/`git diff` exit 1 on differences and
  kill `&&` chains — use `;`. Satisfy both `patch` and `git apply`.
- Patch placement: extend the patch that owns adjacent logic; stand alone when
  touching pristine upstream code. Never duplicate upstream content or graft
  unrelated behaviour onto someone else's patch.
- Dropping a patch requires proof it is in `<new>` upstream (sha/subject). When
  a patch exists upstream, rebuild from the upstream original, not from the
  old-kernel forward-port.
- A failed hunk is not proof a patch is obsolete: the new kernel may have moved
  the surrounding nodes/helpers into another file. Rebase unless the unique
  change exists in pristine `<new>`.
- Refresh in series order with quilt: set up a scratch pristine `<new>` tree
  with `patches/series`, `quilt push` each patch, and run
  `QUILT_DIFF_OPTS="-p" quilt refresh -p ab --no-index --no-timestamps`
  (same invocation as `include/quilt.mk`) on patches showing offset/fuzz.
  Refreshing an earlier patch moves later hunk positions.
- Verify sequentially: a batch of `patch --dry-run` calls without applying
  each predecessor reports false offsets/fuzz. Use `quilt push`, or
  `patch --dry-run && patch` per patch, before claiming zero offsets.
  Re-verify the FINAL patch files after all refreshes: refreshing predecessors
  can shift context so a previously clean patch now fails on replay.
- After restoring a patch with `git show` or resolving a merge, check for
  conflict markers, a missing diffstat/separator blank, and stale `@@`
  headers before trusting it.
- Back up `patches-<ver>/` before bulk edits; `git show <copy-commit>:path`
  recovers a deleted patch.
- One invariant, one implementation: fold drifting copies into one helper.
- Prefer an existing carrier (skb field, metadata, existing parameter, caller
  lock) over new plumbing; list every caller before changing a signature.

## Per-target migration
The kernel move is performed as four commits per target, in the exact order
used on the existing branches. Always use the existing branches as the
authoritative reference for what the commits contain and how they are named.
Do not invent steps; mirror the pattern found there. Use `bcm53xx` as the
primary reference branch throughout; it is a stable pointer to branch shape,
not an immutable commit — re-derive content from current branch history. If an
unusual situation presents itself, inspect other migrated branches for
guidance. Empty 2/4 and 3/4 commits are valid when there is nothing to delete
or refresh. Coverage check: compare
`git ls-tree --name-only generic:target/linux` with the distribution's target
index; every non-generic target except the base target needs a branch.

1/4 copy patches and config from the old kernel
- IMPORTANT: the baseline for everything is the local **`generic` branch**.
  Copy patches and config from it (`git show generic:target/linux/<target>/...`);
  do not name remote-tracking refs such as `origin/generic`, since remotes can
  be renamed. Keep the baseline's old-kernel files current and verify against
  the reference branches' 1/4 commits before proceeding — a stale baseline
  makes the migration diverge from the established pattern.
- Create `target/linux/<target>/patches-<new>` from `patches-<old>`:
  `cp -a target/linux/<target>/patches-<old>/* target/linux/<target>/patches-<new>/`
- For each subtarget, copy the kernel config:
  `cp target/linux/<target>/<subtarget>/config-<old> target/linux/<target>/<subtarget>/config-<new>`
  For targets with a single config, copy `target/linux/<target>/config-<old>`
  → `config-<new>`. Check for symbols renamed between kernel versions before
  committing; e.g. this migration renamed `CONFIG_MTD_VIRT_CONCAT` to
  `CONFIG_MTD_VIRT_CONCAT_DT`.
- If `target/linux/<target>/files-<old>/` exists, copy it to `files-<new>/`:
  `cp -a target/linux/<target>/files-<old> target/linux/<target>/files-<new>`
  The build uses `files-$(KERNEL_PATCHVER)`; omitting this leaves dangling
  `source "..."` lines for drivers whose Kconfig lives only there. Unversioned
  `target/linux/<target>/files/` is copied for every kernel version; do not
  create a `files-<new>` copy of it.
- Commit as `<target> <new> (1/4): copy patches and config from <old>`
- Reference branch for this step: `bcm53xx`.

2/4 delete outdated backports
- In `target/linux/<target>/patches-<new>/`, remove patches that are now
  upstream in `<new>`. These are typically version-prefixed backports for
  kernels older than `<new>` that were required for `<old>` but are merged.
- Identify candidates by comparing `patches-<old>` vs `patches-<new>` after the
  copy and by inspecting the existing branches for the exact files they
  removed.
- Reliable obsolete-patch detection:
  * Clean the build tree:
    `rm -rf build_dir/target-*/linux-<target>_*/linux-<new>.*`
  * Build with `CONFIG_TESTING_KERNEL=y` until the first
    `Patch failed! Please fix` is hit.
  * Verify upstream presence: inspect the extracted `<new>` source in
    `build_dir/.../linux-<new>.*/` for the change the patch introduces. Use
    `grep -rn` for unique symbols / nodes / macros from the patch.
  * If the change is already present in the pristine `<new>` tree, the patch
    is redundant/obsolete – delete it from `patches-<new>/` and clean
    `build_dir/` before retrying. Do not regenerate hunks for a change that
    is already upstream.
  * Repeat until the series applies cleanly with quilt. This catches
    non-version-prefixed backports that have landed in `<new>`.
- Also delete patches whose change is already present in pristine `<new>` even
  if they have no version prefix, and patches that delete or move a
  Kconfig/source file that `<new>` upstream now owns (the latter produce
  dangling `source "..."` lines and break syncconfig). The version rule alone
  is not sufficient.
- Back up `patches-<ver>/` before bulk edits; beware that re-adding a patch
  after step 2 often needs the whole series replayed, not just that patch.
- Commit as `<target> <new> (2/4): delete outdated backports`
- Reference branch for this step: `bcm53xx`.

3/4 update patches
- Ensure every patch in `patches-<new>` applies cleanly to the `<new>` kernel
  tree.
- Run `make target/linux/refresh` on a tree with `<new>` enabled, or regenerate
  failing patches manually from a scratch copy in `/tmp` with the correct
  base: "the tree produced by all preceding patches". Never generate against
  `build_dir/`.
- Update hunks to reflect `<new>` context changes, keep patch count and
  `patch --dry-run` fuzz/FAILED = 0. Run `scripts/checkpatch.pl --no-tree`
  on changed hunks.
- Add a new patch only when the existing branches added one for the same
  reason.
- Verify header / implementation signature consistency for patches that change
  kernel APIs: generic backports that modify headers must be matched by a
  corresponding implementation update. Check macro argument counts against
  call sites in packages; add a minimal implementation patch if upstream
  `<new>` has not yet updated the file.
- Verify includes for helpers moved to new headers in `<new>`. When a
  backported driver uses such helpers, add the correct include to the patch
  rather than leaving an implicit declaration.
- Commit as `<target> <new> (3/4): update patches`
- Reference branch for this step: `bcm53xx`.

4/4 fix configuration
- Set `KERNEL_TESTING_PATCHVER:=<new>` in `target/linux/<target>/Makefile`
  alongside the existing `KERNEL_PATCHVER:=<old>`. Do not change the default
  `KERNEL_PATCHVER`.
- If the build reports `Missing kernel version/hash file for .`, the empty
  version means `KERNEL_TESTING_PATCHVER` is unset/empty in the target
  Makefile, not that `kernel-<new>` is missing (`include/kernel-version.mk`).
- Run `make kernel_oldconfig` with `CONFIG_TESTING_KERNEL=y` to import new
  symbols and drop removed ones. The config files are then updated to add
  `# CONFIG_... is not set` lines for symbols removed in `<new>`.
- NEVER discover new symbols by compiling: non-interactive `syncconfig` stops
  at the first `(NEW)` prompt, so one build reveals one symbol and sibling
  subtargets rediscover the same ones. Do one non-compiling `oldconfig` pass
  per target/subtarget instead:
  * Merge the same inputs OpenWrt uses with `scripts/kconfig.pl`: one `+`
    operator per extra file, e.g. `kconfig.pl + + generic/config-<new> <root
    config-<new>> <subtarget config-<new>>`. Include the filtered top-level
    `.config` from `make defconfig` (only `CONFIG_*=` and
    `# CONFIG_* is not set` lines) when you need symbols that are visible
    only under `CONFIG_TESTING_KERNEL` or the selected target; without it,
    target-specific symbols are missed and compiles still prompt.
  * Get Kconfig ARCH from `target/linux/<target>/Makefile` with any subtarget
    `target.mk` override; map OpenWrt architecture names to kernel ARCH names
    (e.g. aarch64→arm64, mipsel/mips64→mips, riscv64→riscv, x86_64→x86).
  * In a scratch kernel tree made with `cp -a` (NEVER hardlinks), run one
    `yes '' | make -s -C <scratch> ARCH=<arch> KCONFIG_CONFIG=<merged> oldconfig`.
    The log lists **all** `(NEW)` prompts at once and the rewritten config
    gives each resolved default (bool, int, or quoted string path).
  * Ignore symbols matched by `target/linux/generic/config-filter`
    (`FRAME_WARN`, `NR_CPUS`, `RPS`, `NET_FLOW_LIMIT`, `INITRAMFS_SOURCE`,
    `COMPAT_32BIT_TIME`, …). Add only the remaining symbols, with their
    resolved values, to the owning `config-<new>` (root and/or subtarget), and
    fold them into commit 4/4.
  * Include digit-prefixed symbols and choice prompts in `(NEW)` detection
    (`[0-9A-Z]`). If oldconfig resolves a choice silently but syncconfig still
    prompts, take the value from the build tree `.config` when that happens.
  * Helpers that sweep multiple subtargets must read already-edited working-tree
    configs, not only `git show <branch>:path`.
  * Earlier branches' `config-<new>` files are not authoritative: re-derive
    new symbols from the actual kernel and merged config.
- When adding a family of defaults, never rewrite a symbol that is already
  set — a broad `LEDS_*` batch must not turn `CONFIG_LEDS_GPIO=y` into
  "not set". Prefer the existing value / the old-kernel baseline on conflicts.
- De-duplicate `config-<new>` after bulk edits: exactly one line per
  `CONFIG_<symbol>`. `kconfig.pl` uses the last occurrence, so duplicates hide
  the effective value. Any script that rewrites commits must preserve the
  readme commit — a `reset --hard` to the (4/4) commit silently drops it.
- Finish the full oldconfig sweep before creating the readme; if a config is
  edited after the readme commit, regenerate the readme prerequisites from the
  final diffs so counts and lists do not drift.
- Use a valid `SUBTARGETS` value from `target/linux/<target>/Makefile` in the
  seed `.config`; `<target>/generic` does not exist for many targets.
- Update `target/linux/<target>/config-<new>` and/or each
  `target/linux/<target>/<subtarget>/config-<new>` accordingly. The exact set
  of symbols to disable is target-specific; use the existing migration
  branches for the same class of target as guidance.
- Commit as `<target> <new> (4/4): fix configuration`
- Reference branch for this step: `bcm53xx`.

After the four commits, verify:
- `make kernel_oldconfig` completes without manual prompts for the target.
- Patches apply clean with `patch --dry-run`.
- The target builds with `CONFIG_TESTING_KERNEL=y`.
- Keep a build ledger: compile each target/subtarget **once**, record the
  result, never re-run a passing build, and after a fix re-run only the
  specific builds that failed. Report findings after each cycle rather than
  looping silently. A config-only change does not need another compile of
  builds that already passed.
- Every branch must keep the four migration commits plus its
  `readme.<target>.md` / `knowngood*.config`. Some branches legitimately carry
  extra non-migration commits; document them rather than assuming a fixed
  count. Rewriting commits (reset/amend/cherry-pick) can silently drop those
  files or leave the branch at an intermediate commit — re-check `git log`,
  `git status` and the file list after any such surgery.

Keep patches that move to a later kernel only when still legitimate; re-key
version gates to the real boundaries of the target kernel.

### readme.<target>.md
Create a per-target build notes file `readme.<target>.md` once the four
commits are done. Use `readme.ramips.md` at branch ramips as the template.

- Copy the file and rename to `readme.<target>.md`.
- Modify only the **Kernel and target prerequisites** section to reflect
  the target’s specific changes:
  * `target/linux/<target>/Makefile` `KERNEL_TESTING_PATCHVER:=<new>`
  * Number and list of outdated backports removed from `patches-<new>`
  * Number and list of patches refreshed for `<new>`
  * Number of new defaults appended to `config-<new>`
  * Any target-specific `Kernel/Prepare` tweaks
- At the top of the readme, list every successfully compile-tested subtarget
  together with the exact seed config used: `knowngood.config` for the first,
  then `knowngood2.config`, `knowngood3.config`, … for each additional
  subtarget. Save the seed that produced the passing build under that name in
  the target branch and link it from the readme. Never list a subtarget as
  tested without an archived seed config — an ad-hoc `.config` that was not
  saved does not document how the success was achieved.
- A seed must select an **actual device** inside the subtarget, not just the
  subtarget itself: `CONFIG_TARGET_<board>=y`,
  `CONFIG_TARGET_<board>_<subtarget>=y` and
  `CONFIG_TARGET_<board>_<subtarget>_DEVICE_<device>=y`, as in
  `ramips:knowngood.config`. A subtarget-only seed does not reproduce a real
  image build and can skip device-specific packages and DTS files.
  After `scripts/diffconfig.sh`, assert the DEVICE line is present; if it was
  omitted as a default, append it. Targets that define no devices (e.g. UML)
  are the unavoidable exception.
- Regenerate the prerequisites bullets from the final commit diffs after any
  late patch/config commit; counts and lists drift otherwise.
- Keep **Feed packages that need patches** and **Materialising the feed
  patches** sections essentially unchanged; they document generic kernel API
  breakage that affects all targets.

## Verification
- Ladder, each level proves only itself: dry-run < real apply < compiles <
  runs once < survives re-entry/error paths. Never present a lower rung as a
  higher one. A zero-offset replay is NOT a compile: rebased patches can apply
  cleanly yet put code in the wrong function or outside it.
- Compile the touched file per affected config, not just the default `.config`
  (driver Kconfigs can make a legal alternative break).
- A non-interactive `syncconfig` fails on new Kconfig symbols rather than
  taking defaults. Check the build log for `(NEW)` prompts before concluding
  the config is complete; if one appears, the oldconfig sweep missed a symbol
  (usually because the seed config was incomplete), so fix the config in one
  pass — never discover the remaining symbols by repeated compiles.
- Any change touching open/stop/setup/teardown needs a runtime down/up test.
- Label handover confidence: proven (named test), inferred (named evidence),
  untested. An untested item never reads as done.
- Trust facts already established for the target kernel in `KERNEL-<new>.md`
  before re-deriving them; for 7.3 see `KERNEL-7.3.md`.

## Environment facts
- Single git repo: OpenWrt tree at `~/openwrt73`.
- Full image builds are the user's job; do not start one unless asked, keep
  compiles short, and never write into `build_dir/`.
  When compilation testing IS requested, `make -j8 target/linux/compile` with
  a minimal `defconfig` and `CONFIG_TESTING_KERNEL=y` is cheap enough to run
  for every target and is the rung that catches Kconfig, DTS and kernel API
  breakage. Clean `build_dir/target-*/linux-<target>_*/linux-<new>.*` after each
  target to reclaim disk.
- Single-TU compile is the cheapest rung that catches interface/Kconfig
  breakage: get the command line from `make -n <object>`, then swap in your
  file and a different `-o`. Hand-rolled `-fsyntax-only` is unreliable.
- Toolchains do not predefine `__GLIBC__`; guard aarch64-only code with
  `__aarch64__`.
- `feeds/` is git-ignored; patches there are lost on feeds update. Feed
  breakage is fixed in the feed, not this tree.
- `bin/`, `build_dir/`, `staging_dir/` can be stale; first `make clean` the
  affected package/target before concluding anything about the code.
