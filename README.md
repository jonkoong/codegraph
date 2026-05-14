# codegraph (personal fork)

Personal patches on top of [colbymchenry/codegraph](https://github.com/colbymchenry/codegraph). Built from source for daily use across Mac + Windows; **never** published to npm.

For the upstream project description, features, and general docs, see [`README.upstream.md`](./README.upstream.md).

---

## Patches

Both live on the `local-patches` branch. The `fork:` prefix marks them as fork-only — not intended for upstream PR.

### `af8ab61` · fork: fix watcher drift — index untracked dirs + sweep orphans

- `src/extraction/index.ts`
- **Bug fix.** Two related issues with the file watcher:
    - `git status --porcelain` collapses a newly-created untracked directory into a single `?? dir/` entry, hiding every file inside from the indexer. Adding `-uall` (`--untracked-files=all`) forces individual enumeration so new files get indexed.
    - `git status` doesn't emit a status code for deleted untracked files — they just disappear from the `??` list. Without an explicit reconcile pass, the DB keeps stale entries forever. After each sync, sweep every DB entry not in git's visible set; `existsSync`-test those and delete the missing ones.

> ⚠️ Not benchmarked on huge repos — the orphan sweep does an `existsSync` per stale entry. Cheap in practice because the stale set is small, but a reason it stays fork-only.

### `2d0dcf4` · fork: drop MCP watcher debounce to 500ms

- `src/mcp/index.ts`
- **Preference.** Default debounce makes edits feel sluggish in the MCP index. 500ms is the sweet spot for me — fast enough to feel live, slow enough not to thrash on rapid saves.

---

## Install on a fresh machine

### 🍎 Mac

```bash
git clone -b local-patches git@github.com:jonkoong/codegraph.git
cd codegraph

# Remotes: fork = your push target, upstream = read-only source
git remote rename origin fork
git remote add upstream https://github.com/colbymchenry/codegraph.git
git fetch upstream

# Guard against accidental bare `git push` going to fork
git config branch.local-patches.pushRemote no_push

npm install
npm run build
npm link                          # makes the patched build the live `codegraph` binary
```

> 💡 If `npm link` errors with `EACCES`, your npm prefix is in a system dir. Either run `sudo npm link`, or fix the prefix to a user dir: `npm config set prefix ~/.npm-global` then add `~/.npm-global/bin` to `PATH`.

### 🪟 Windows

```powershell
git clone -b local-patches git@github.com:jonkoong/codegraph.git
cd codegraph

git remote rename origin fork
git remote add upstream https://github.com/colbymchenry/codegraph.git
git fetch upstream
git config branch.local-patches.pushRemote no_push

npm install
npm run build
npm install -g .                  # avoids npm link's symlink/admin-mode requirement on Windows
```

### Verify the patched build is live

```bash
which codegraph                   # mac
where codegraph                   # windows
codegraph --version               # reports upstream's version — see caveat below
```

To confirm the patches are actually in the running build:

```bash
# Should find `debounceMs: 500` (patch 2)
grep -n "debounceMs: 500" "$(npm root -g)/codegraph/dist/mcp/index.js"

# Should find the orphan sweep (patch 1)
grep -n "Reconcile orphans" "$(npm root -g)/codegraph/dist/extraction/index.js"
```

> ⚠️ `codegraph --version` reports upstream's `package.json` version (e.g. `0.7.6`). The fork **does not bump the version** — there's no semver to bump against. Identify the patched build by SHA (`git -C <fork-clone> rev-parse HEAD`) or by the grep checks above.

### Undo (revert to published codegraph)

```bash
# Mac (after npm link)
npm unlink -g codegraph
npm install -g @colbymchenry/codegraph

# Windows (after npm install -g .)
npm uninstall -g codegraph
npm install -g @colbymchenry/codegraph
```

---

## Branch policy

| Branch | Tracks | Purpose |
|---|---|---|
| `main` | `upstream/main` | Mirror of upstream. **Never commit here.** Used only as the rebase base for `local-patches`. |
| `local-patches` | `fork/local-patches` | All work happens here. Rebased on top of `main` whenever upstream moves. |

The `upstream` remote is **read-only by convention** — never `git push upstream`. The `pushRemote = no_push` config on `local-patches` means a bare `git push` (no args) errors out instead of going somewhere unintended; explicit `git push fork local-patches` works.

---

## Maintenance

### Add a new patch (everyday case)

On the dev machine (this Mac):

```bash
git checkout local-patches
# edit files…
git commit -am "fork: <what you changed>"
git push fork local-patches              # fast-forward, no --force needed
npm run build                            # patched build is live via npm link
```

### Pull upstream updates and rebase patches on top

```bash
git fetch upstream
git checkout main
git merge --ff-only upstream/main        # aborts if main has somehow diverged
git checkout local-patches
git rebase main                          # replays your patches on top
npm install && npm run build
```

If the rebase has conflicts: resolve → `git rebase --continue`. If it goes sideways: `git rebase --abort`.

Rebase rewrites SHAs, so the push must be force-with-lease:

```bash
git push --force-with-lease fork local-patches
```

`--force-with-lease` refuses if someone else has pushed in the meantime — safe because **you** are the only writer to this fork.

### Update consumer machines (the other Mac, Windows) after you push

```bash
cd path/to/codegraph
git fetch fork
git reset --hard fork/local-patches      # discards any local edits
npm install && npm run build
```

> ⚠️ `git reset --hard` is destructive. Only run on consumer machines where you don't edit code — never on the dev Mac.

---

## Hard rules

- 🚫 **Never `npm publish`.** Package name in `package.json` is `@colbymchenry/codegraph` — only colbymchenry can publish to that scope, so it would fail, but stating it out loud anyway.
- 🚫 **Never push to `upstream`.** Pull-only.
- 🚫 **Never commit to `main`.** It exists to mirror upstream and act as the rebase base.
- ✅ Patches stay on `local-patches`. Force-push to `fork` is fine **with `--force-with-lease`** when rebasing on new upstream.
