# FORK.md — nicolai-creates/model27

This repository is Nicolai's fork of [durchgemacht/model27](https://github.com/durchgemacht/model27). These rules
come on top of `AGENTS.md`, and win where the two disagree. They live here, not in `AGENTS.md`, so that rebasing onto
upstream does not conflict with durchgemacht's frequent edits there.

## Who

- **The maintainer of this fork is Nicolai** (GitHub `nicolai-creates`). Talk to them in English.
- Where `AGENTS.md` says "the maintainer", it means Nicolai on this fork, durchgemacht upstream.
- Nicolai tests firmware changes on their own Model:Cycles. Rule 3 of `AGENTS.md` applies to them: a mod becomes
  `tested` only after Nicolai reports that it works on the device.

## Branches and remotes

| Remote | Repository | Use |
|---|---|---|
| `upstream` | `durchgemacht/model27` | fetch only (push URL disabled) |
| `github` | `nicolai-creates/model27` | this fork |

- **`main`** mirrors `upstream/main` exactly and only ever fast-forwards. Never commit to it.
- **`nicolai`** is the personal branch: `main` plus Nicolai's commits. GitHub Pages serves its `docs/`, at
  <https://nicolai-creates.github.io/model27/>, as the flasher Nicolai flashes from.
- Updating:

  ```sh
  git fetch upstream
  git switch main && git merge --ff-only upstream/main && git push github main
  git switch nicolai && git rebase main
  # regenerate generated files instead of resolving their conflicts (below), then:
  git push --force-with-lease github nicolai
  ```

  `nicolai` is rebased, not merged. Nobody else builds on it. This overrides `AGENTS.md`'s rule to merge `main` into
  shared branches.

  Locally, `main` tracks `upstream/main` (fast-forward only) and pushes to `github`, so on `main` a plain
  `git pull && git push` does the mirroring.

- **Every day a workflow mirrors `main`, and only `main`** (`.github/workflows/sync-upstream.yml`, 05:17 UTC, or by
  hand from the Actions tab).
  - It fast-forwards `main` to upstream's, with the secret `SYNC_TOKEN`: a fine-grained token for this repository,
    with Contents and Workflows read and write.
  - It never touches `nicolai`. **Updating `nicolai` is Nicolai's deliberate step**: rebase, run the checks, maybe flash
    and test, then push. Only then does the site change.
  - When `nicolai` is behind, the workflow tries the rebase in the runner, without pushing, and reports in one open
    issue labelled `upstream-sync`: how far behind, whether the rebase applies cleanly or where it conflicts, and the
    new upstream commits.
    - It comments on the issue (a notification) only when upstream has moved since the last report.
    - It closes the issue once `nicolai` contains upstream's `main`.

## Pull requests to upstream

- This repository is a GitHub fork of `durchgemacht/model27` (recreated as one on 2026-11-09), so pull requests go
  from it to upstream directly.
- **Nicolai's usual way: develop on `nicolai`, cherry-pick for a pull request.**
  1. Work happens on `nicolai`.
  2. When something is ready for upstream:

     ```sh
     git switch -c feature/<x> main
     git cherry-pick <commit>…          # only the commits upstream should get, never a merge of nicolai
     git push github feature/<x>        # then the pull request against durchgemacht/model27 (below)
     ```

  3. Once upstream has merged it, the next `git rebase main` of `nicolai` drops the copies on `nicolai`. If
     durchgemacht changed the commit, the rebase stops on it: `git rebase --skip` keeps upstream's version.
- **So that commits can be picked, on `nicolai`:**
  - one commit, one concern: never an upstream-worthy change and fork-only files (`FORK.md`, `notes/fork/`) in one
    commit;
  - generated files in commits of their own (`tweaks.js`, `REF_MAINOS`/`REF_MODS`), which are not picked. A pull
    request leaves them to the maintainer (`CONTRIBUTING.md`), apart from `REGISTRY.md` and the mod's own object and
    tweak;
  - a commit meant for upstream does not depend on fork-only commits.
- **A feature planned for upstream from the start** can go the other way round:
  1. **Upstream first.** For a new mod, open an issue upstream first (`CONTRIBUTING.md`: the maintainer gives its
     `order`).
  2. **Branch from `main`, not from `nicolai`:** `git switch -c feature/<x> main`.
     - Commit only what upstream should get. Nothing from `FORK.md` or `notes/fork/`.
     - Upstream's rules apply: `AGENTS.md`, `CONTRIBUTING.md`, and generated files left as `CONTRIBUTING.md` says.
  3. **Use it yourself:** `git switch nicolai && git rebase feature/<x>`. This puts the stack on top of the feature.
     After that, `nicolai` is rebased onto `main` as usual. Once upstream has merged the feature, the rebase drops
     the commits it already finds there.
  4. **Offer it:**
     - `git push github feature/<x>`;
     - then `gh pr create --repo durchgemacht/model27 --base main --head nicolai-creates:feature/<x>`, with `gh`
       logged in as `nicolai-creates`;
     - or open the pull request on the web.

## Guard hooks (local, not versioned)

`.git/hooks/` is not part of the repository. To refuse commits on `main`, and pushes of a `main` that upstream does
not contain:

```sh
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
[ "$(git symbolic-ref --short -q HEAD)" = main ] && { echo "main mirrors upstream (FORK.md)" >&2; exit 1; }
exit 0
EOF
cat > .git/hooks/pre-push <<'EOF'
#!/bin/sh
while read -r lref lsha rref rsha; do
  [ "$rref" = refs/heads/main ] || continue
  git merge-base --is-ancestor "$lsha" upstream/main 2>/dev/null ||
    { echo "main may only take commits of upstream/main (FORK.md)" >&2; exit 1; }
done
EOF
chmod +x .git/hooks/pre-commit .git/hooks/pre-push
```

On GitHub, a branch ruleset on `main` (Settings › Rules) that blocks force pushes and deletion protects the mirror
there too.

## Keeping the stack easy to rebase

- **Agent instructions** go into this file, never into `AGENTS.md` or the folders' `AGENTS.md`. `CLAUDE.md` imports
  it after `AGENTS.md`.
- **Notes of this fork** go into `notes/fork/`, numbered on their own (`01-…`). Upstream's numbering in `notes/`
  then never collides with them.
- **Generated files** are never resolved by hand in a rebase: regenerate them (`tools/gen_flasher_tweaks.py`,
  `tools/ref_mainos.py`, `tools/registry.py`, …). This covers `docs/flasher/tweaks.js`, `REF_MAINOS`/`REF_MODS` in
  `docs/flasher/app.js` and `tweaks/model-cycles_OS1.13/REGISTRY.md`. Keep regenerated outputs in a late commit of
  the stack, so that only that commit has to be rebuilt after a rebase.
- **A fix that upstream needs too** is its own small commit, offered to durchgemacht as an issue or pull request. Once
  upstream has it, the rebase drops it.

## Local setup

- The official files are in `firmware/` (git-ignored): `model-cycles_OS1.13.syx` and `Syntakt_OS1.42.syx`.
- m68k toolchain: `m68k-elf-*` from the AUR, GCC 16.1. The committed objects come from GCC 16.2. Most of them give the
  same code under 16.1, and `--check` then differs only by the compiler's version string. `sampler`, `resample`,
  `cpu` and `chordtables` give different code (checked 2026-11-09). An exact `--check` needs GCC 16.2.
- Emulation: `uv run -q --with unicorn,numpy,pillow python3 tools/emu/test_<x>.py --cycles firmware/…`. Some proofs
  run for more than an hour, so don't give them a short timeout.
