# Does marking every LFS type `lockable` work for this pipeline?

Research for issue #7. Investigated 2026-09-02 against git-lfs 3.7.1 (the version
installed locally), the git-lfs source on `main`, the git-lfs man pages on this
machine, the Git LFS File Locking API spec, and Autodesk/Adobe documentation.
Local experiments were run in a scratch branch of this repo; no locks were
created on the GitHub server and nothing was pushed.

**Short answer: no.** Marking every type `lockable` breaks derived-artifact
regeneration (notably Arnold `.tx`) and buys nothing for files nobody edits
by hand. Locks are strictly per-file, folders cannot be locked at all, and the
read-only bit is advisory — the real enforcement happens at push time.

## 1. What `lockable` actually does

### The attribute is independent of LFS

`lockable` is a plain gitattributes flag. Nothing in the write-flag machinery
requires `filter=lfs` on the same line: `Client.refreshLockablePatterns()`
collects every attribute path where `p.Lockable` is set, and
`fixSingleFileWriteFlags` consults only that filter.

- Source: [`locking/lockable.go`](https://github.com/git-lfs/git-lfs/blob/main/locking/lockable.go)

So a path can be lockable without being in LFS, and vice versa. This matters
for `.tx` below.

### The read-only bit is set by hooks, not by the smudge filter

`.git/hooks/post-checkout` and `.git/hooks/post-commit` (both installed in this
repo) shell out to `git lfs post-checkout` / `git lfs post-commit`. Those
commands call `FixLockableFileWriteFlags` (for the changed files) or
`FixAllLockableFileWriteFlags` (full scan). `post-merge` does the same.

- Source: [`commands/command_post_checkout.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_post_checkout.go),
  [`commands/command_post_commit.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_post_commit.go)

Consequences:

- If a collaborator has not run `git lfs install`, the hooks are absent and
  **no file is ever made read-only for them**.
- A file is only touched if `git ls-files` lists it. New, untracked files stay
  writable.

### Exact permission change

`tools.SetFileWriteFlag(path, false)` does `mode &^ 0222` — it clears *all*
write bits. Setting writable does `mode | 0200` — owner write only.

```go
if writeEnabled {
    mode = mode | 0200  // set owner write only
} else {
    mode = mode &^ 0222 // disable all write
}
```

- Source: [`tools/filetools.go:412`](https://github.com/git-lfs/git-lfs/blob/main/tools/filetools.go)

Verified locally in this worktree: deleting `shots/TBIRD_001_0001.ma` and
running `git checkout -- shots/TBIRD_001_0001.ma` produced mode `-r--r--r--`
(0444) from a starting 0644.

### Timing: writable until committed

Verified locally with a scratch `*.psd` (a lockable pattern here):

| Step | Mode |
| --- | --- |
| file created in working tree | `-rw-r--r--` |
| after `git add` | `-rw-r--r--` |
| after `git commit` | `-r--r--r--` |
| after delete + `git checkout --` | `-r--r--r--` |

The `post-commit` hook's own comment states why: *"This is mainly to catch added
files, since modified files should already be locked. If we didn't do this, any
added files would remain read/write on disk even without a lock."*

### The bit does not stop Git

Verified locally: `git checkout HEAD~1 -- scratch_test.psd` overwrote a 0444
file without error and left it 0444. A teammate's `git pull` will happily
replace the bytes under an artist's read-only file. Read-only protects against
the *application* saving, not against Git.

### Every user can turn it off

`GIT_LFS_SET_LOCKABLE_READONLY` (env) and `lfs.setlockablereadonly` (gitconfig)
default to `true`; setting either to `0`, `no`, or `false` makes every lockable
file writable for that clone. `lfs.lockignoredfiles` defaults to `false`, so
gitignored files matching a lockable pattern are *not* made read-only.

- Source: `man git-lfs-config` (git-lfs 3.7.1), sections
  `GIT_LFS_SET_LOCKABLE_READONLY` and `lfs.lockignoredfiles`; mirrored at
  [`docs/man/git-lfs-config.adoc`](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-config.adoc)

**The read-only bit is a courtesy, not a control.**

### Editing `.gitattributes` by hand is not enough

`git lfs track --lockable <pattern>` writes the attribute *and* immediately
calls `FixFileWriteFlagsInDir(relpath, readOnlyPatterns, writeablePatterns)`,
chmod'ing matching files right away. Hand-editing `.gitattributes` — which is
how this repo's file was written — does nothing until the next checkout, commit,
or merge fires a hook. That explains why the two `.ma` files in this worktree
were still 0644 before the experiment above.

- Source: [`commands/command_track.go:231`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_track.go)

## 2. `git lfs lock`

### You cannot lock a folder. Ever.

`lockPath()` stats the argument and returns an error if it is a directory:

```go
if stat, err := os.Stat(abs); err == nil && stat.IsDir() {
    return path, errors.New(tr.Tr.Get("cannot lock directory: %s", file))
}
```

- Source: [`commands/command_lock.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_lock.go)

The API has no directory, prefix, or glob concept either. `POST /locks` takes a
single `path` string and the server "should ensure ... that files are locked
exclusively to one user" — one path, one lock.

- Source: [Git LFS File Locking API](https://github.com/git-lfs/git-lfs/blob/main/docs/api/locking.md)

**"Lock a folder" can therefore only mean: enumerate the files under the folder
and issue one `POST /locks` per file.** N files = N round trips.

### Files created in the folder after the lock are not covered

There is no watch, no prefix rule, no server-side inheritance. A file added to a
"locked" folder afterwards belongs to nobody. Worse, per section 1 it stays
writable for everyone until it is committed, at which point it flips to 0444 for
everyone *including its author* unless they hold a lock on it.

Any folder-lock tool must re-enumerate and lock new paths — the natural hook
points are post-commit and pre-push.

### Does the file have to be committed first?

The man page says yes: *"Locking a file requires the file to exist in the
working copy."* (`man git-lfs-lock`, git-lfs 3.7.1).

The client does not enforce it. `lockPath` ignores the `os.Stat` error for a
missing file, and `Client.LockFile` calls the server first, then chmods the file
only *"if the file exists"*:

```go
lockRes, _, err := c.client.Lock(c.Remote, &lockRequest{Path: path, ...})
...
// If the file exists, ensure that it's writeable on return
if tools.FileExists(abs) {
    if err := tools.SetFileWriteFlag(abs, true); err != nil { ... }
}
```

- Source: [`locking/locks.go:104`](https://github.com/git-lfs/git-lfs/blob/main/locking/locks.go)

The API spec places no existence requirement on `path` either. So the answer is
**working-copy existence is documented but not enforced client-side, and
committed status is never required**. Whether GitHub's server rejects a path
that does not exist in the target ref is **unverified** — the ticket forbade
creating locks on the server. Verify before designing around pre-emptive locks.

### Locks are scoped to a ref

`git lfs lock` sends `ref.name` = the remote ref for your current branch
(`git.NewRefUpdate(cfg.Git, cfg.PushRemote(), cfg.CurrentRef(), nil)`), and
`POST /locks/verify` sends the ref being pushed. The spec calls this
"single branch locking" and warns the design is provisional.

**Design risk:** a lock taken while on `feature-x` may not be reported when
someone pushes `main`. If the folder-lock design assumes global locks, pick a
branch policy (everyone locks from the branch they will push) and state it.

### GitHub is a known locking server

git-lfs hardcodes `https://github.com` and `ssh://github.com` in
`hostsWithKnownLockingSupport`, so `lfs.<url>.locksverify` resolves to enabled
for this remote without configuration.

- Source: [`commands/uploader.go:320`](https://github.com/git-lfs/git-lfs/blob/main/commands/uploader.go)

## 3. `git lfs push` when someone else holds the lock

Flow: `pre-push` hook → `uploadForRefUpdates` → `verifyLocksForUpdates` →
`POST /locks/verify`, which returns locks partitioned into `ours` and `theirs`.

If the push updates a path in `theirs`:

```
Unable to push locked files:
* <path> - <owner>
```

then, depending on `lfs.<url>.locksverify`:

| `locksverify` | Behavior |
| --- | --- |
| `true` (resolved for github.com) | `Cannot update locked files.` — **push aborts** |
| unset / `null` | warning only: `warning: The above files would have halted this push.` Push proceeds. First successful verify call sets the value to `true`. |
| `false` | lock check skipped entirely |

- Source: [`commands/uploader.go:173-317`](https://github.com/git-lfs/git-lfs/blob/main/commands/uploader.go),
  [`commands/lockverifier.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/lockverifier.go),
  `man git-lfs-config` (`lfs.<url>.locksverify`)

If *you* hold the lock, the push succeeds and prints
`Consider unlocking your own locked files: (git lfs unlock <path>)`.

### Verification is NOT limited to LFS files

This is the finding that matters most for `.tx`. `NewGitScannerForPush` is
documented as scanning *"for objects to push to the given remote **and for locks
on non-LFS objects held by other users**"*, and `runCatFileBatchCheck` routes
plain Git blobs into the lockable channel when the path is in the lock set:

```go
} else if b := scanner.GitBlobOID(); len(b) > 0 {
    if name, ok := lockableSet.Check(b); ok {
        lockableCh <- name
    }
}
```

- Source: [`lfs/gitscanner.go:59`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner.go),
  [`lfs/gitscanner_catfilebatchcheck.go:19`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner_catfilebatchcheck.go),
  [`lfs/gitscanner_refs.go:74`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner_refs.go)

A lock on a non-LFS path is still enforced at push. A folder-lock tool can lock
any tracked path, LFS or not.

Note that nothing blocks `git commit` locally. The wall is at push, which means
an artist can do a full day's work on a file someone else holds and only find
out when they try to push.

## 4. `git lfs unlock` and the read-only bit

`git lfs unlock <path>`:

1. Refuses if the file has uncommitted changes:
   `Cannot unlock file with uncommitted changes` (`unlockAbortIfFileModified`).
   `--force` downgrades this to a warning.
2. Calls the server to delete the lock.
3. If the path is still lockable and the file exists, calls
   `tools.SetFileWriteFlag(abs, false)` — **back to 0444**.

```go
// Make non-writeable if required
if c.SetLockableFilesReadOnly && c.IsFileLockable(unlockRes.Lock.Path) && tools.FileExists(abs) {
    return tools.SetFileWriteFlag(abs, false)
}
```

- Source: [`locking/locks.go:188`](https://github.com/git-lfs/git-lfs/blob/main/locking/locks.go),
  [`commands/command_unlock.go:139`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_unlock.go),
  `man git-lfs-unlock`

`git lfs unlock --force <path>` breaks another user's lock. Nothing on the
protocol side prevents this; anyone with push access can do it.

`git lfs locks --verify` marks your own locks with `O` and detects locks held by
you from a different clone, plus "broken" locks where someone force-unlocked
you (`man git-lfs-locks`).

## 5. Do Maya, After Effects, and Photoshop tolerate read-only files?

**Caveat:** none of these applications is installed on this machine, so nothing
in this section was tested. No first-party Autodesk or Adobe page was found that
states read-only behavior directly; the vendor material below is about
*fixing* read-only errors, which tells us the failure mode but not the exact
dialogs. Treat the per-app dialog behavior as **unverified** and worth ten
minutes of hands-on testing per app before the design ships.

What is structurally certain: mode 0444 blocks in-place writes and nothing else.
Reading, opening, file-referencing, and texture loading all succeed.

### Maya — source scenes: fine

Opening a `.ma`/`.mb` and loading it as a file reference are reads. `File > Save
Scene` to the same path fails; `Save Scene As` to a new path works. Autodesk's
own support article on the read-only save error is
["The file that is being saved is set to read-only"](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/The-file-that-is-being-saved-is-set-to-read-only.html)
(returns 403 to automated fetch; reachable in a browser).

Maya writes sidecars *next to* the scene — `.ma.swatches` and `incrementalSave/`,
both gitignored here. Those are separate paths, so a read-only scene does not
block them. Directories are never chmod'ed by git-lfs (`fixFileWriteFlags`
iterates `git ls-files`, which lists files only), so the directory stays
writable.

One trap: this repo gitignores `*.ma.[0-9][0-9]*` (Maya incremental saves).
Keep `lfs.lockignoredfiles` at its default `false`, or those incremental saves
become read-only and Maya's incremental save fails.

Autodesk also notes that on Linux and macOS a directory needs `555`
(`r-xr-xr-x`) permissions for Maya to use it; `444` is not enough
([Files/Projects preferences](https://help.autodesk.com/view/MAYAUL/2024/ENU/?guid=GUID-8E332DD5-879C-4AFD-8C6C-7C403FEF310F)).
Not an issue here since git-lfs never touches directory modes.

### Maya — texture references: fine

Maya and Arnold read texture files and never write back to a source PNG, EXR,
or TIFF. Read-only source textures are safe. They are also non-lockable in the
current `.gitattributes`, which is correct.

### `.tx` sidecars: **this is the one that breaks**

Arnold auto-generates `.tx` and writes it beside the source texture:

> "TX files will be generated in the same folder as the source texture, unless
> `options.texture_auto_tx_path` is set, in which case all new TX files will be
> generated into that folder (preserving the existing folder tree to avoid
> collisions)."

and regenerates when outdated: `ARNOLD_AUTO_GENERATE_TX` accepts `never` or
`when_outdated`; under `when_outdated`, an existing matching TX "won't be
regenerated unless the source texture contents or colorspace has changed."

- Source: [Arnold User Guide — Textures (settings)](https://help.autodesk.com/cloudhelp/ENU/AR-Core/files/ac-render-settings/arnold_user_guide_ac_render_settings_ac_textures_settings_html.html)
- Source: [How to enable Auto-Convert Textures to TX in Maya with Arnold](https://knowledge.autodesk.com/support/maya/troubleshooting/caas/sfdcarticles/sfdcarticles/How-to-enable-Auto-Covert-Textures-to-TX-in-Maya-with-Arnold.html)

If a `.tx` is read-only and its source texture changes, Arnold's in-place
regeneration write fails. Marking `.tx` lockable therefore breaks the render
pipeline for every artist who does not hold the lock — which is everyone except
one person at a time.

Note also that a read-only *source* PNG does not block autotx, because the `.tx`
is a different file. Only the `.tx`'s own mode matters.

### After Effects and Photoshop: open works, save-in-place fails

Same shape as Maya: opening a read-only `.aep` or `.psd` is a read; saving to
the same path raises Adobe's locked-file error
(`Could not save … because the file is locked or you do not have the necessary
access permissions`), and `Save As` to a new path works. The vendor-side
material is community threads rather than product docs:

- [Adobe Community — "Could not save the file because the file is locked" (Photoshop)](https://community.adobe.com/t5/photoshop/cant-save-a-file-quot-because-the-file-is-locked-you-do-not-have-access-permissions-quot/m-p/9570391)
- [Creative COW — "AE will not let me save, project is locked"](https://creativecow.net/forums/thread/ae-will-not-let-me-save-project-is-locked/)
- [Adobe — Project types in After Effects](https://helpx.adobe.com/after-effects/using/projects.html)

After Effects also writes auto-save copies into a sibling
`Adobe After Effects Auto-Save/` directory, which this repo gitignores. Separate
paths, so read-only projects do not block auto-save.

**Unverified and worth testing:** whether Maya throws a modal on *opening* a
read-only scene, and whether After Effects opens a read-only `.aep` at all or
refuses. Both would change the artist experience even though neither breaks
correctness.

## 6. The `.tx` situation in this repo today

Measured in this worktree:

- 15 `.tx` files, 11 MB total.
- All are tiled OpenEXR (`file` reports `OpenEXR image data, version 2,
  storage: tiled, compression: dwaa/zip`), not TIFF.
- Naming is `<source>.<udim>_<srcColorspace>_<renderColorspace>.png.tx`, e.g.
  `TB_UVs_Black_BaseColor.1001_sRGB_ACEScg.png.tx` beside
  `TB_UVs_Black_BaseColor.1001.png`. That is Arnold autotx output, not
  hand-authored.
- `git check-attr -a` on one of them returns only `text: auto`. **`.tx` is
  absent from `.gitattributes`**, so these are ordinary Git blobs, outside LFS,
  and matched only by the catch-all `* text=auto` line.

Two problems independent of locking:

1. **Repo bloat.** 11 MB of binary sits in the Git object database and every
   historical revision ships with every clone forever. The source `.png` files
   they derive from *are* in LFS, so the derived artifact costs more than the
   original.
2. **`text=auto` on binary.** Git runs its CRLF heuristic on them at check-in.
   Binary detection (a NUL byte in the first 8000 bytes) saves EXR in practice,
   but it is a heuristic, not a guarantee, and it should not be load-bearing.

The right fix is to gitignore `.tx` as a derived artifact, or — if the team
wants pinned `.tx` for render-farm determinism — add
`*.tx filter=lfs diff=lfs merge=lfs -text` **without** `lockable`.

## 7. Facts the folder-lock design needs

1. **There is no folder lock.** Expand a folder to a file list (`git ls-files`
   under the prefix, filtered to lockable patterns) and issue one
   `POST /locks` per path. Budget N HTTP calls and GitHub rate limits; a shot
   folder with hundreds of files is slow and expensive. Lock only unmergeable
   source files, not everything under the prefix.
2. **New files in a locked folder are uncovered.** Add a re-scan that locks new
   lockable paths under folders the user holds. Post-commit and pre-push are the
   natural trigger points, since a new file is writable to everyone until its
   first commit and read-only to everyone after it.
3. **Committed status is not required to lock**, and working-copy existence is
   documented but not client-enforced. Confirm GitHub's server-side rule before
   relying on locking paths that do not exist yet.
4. **Locks are per-ref.** Adopt an explicit branch policy and say so, or locks
   taken on one branch will silently fail to protect a push from another.
5. **The read-only bit is advisory.** Any user can disable it with
   `lfs.setlockablereadonly=false`, and it never appears at all if they skipped
   `git lfs install`. Do not treat it as enforcement. The enforcement point is
   `POST /locks/verify` at push.
6. **`git lfs push` verification covers non-LFS paths too**, so the tool may
   lock any tracked path regardless of LFS status.
7. **Unlock refuses on dirty files.** The "release the folder" step must handle
   partial failure and report which files are still dirty rather than silently
   leaving locks behind.
8. **Hand-edited `.gitattributes` does not chmod anything.** After changing the
   lockable set, either run `git lfs track --lockable <pattern>` or tell everyone
   to force a full pass (a checkout, or `git lfs post-checkout 0 0 0`).
9. **Read-only does not stop `git pull`.** A collaborator's pull will overwrite
   a locked artist's on-disk file. Locking is not a merge-conflict shield.
10. **Keep `lfs.lockignoredfiles` at `false`** so gitignored Maya incremental
    saves, caches, and AE auto-saves stay writable.

## 8. Types that should NOT be lockable

| Pattern | Why not |
| --- | --- |
| `*.tx` | Arnold regenerates these in place when the source or colorspace changes. Read-only breaks autotx for everyone but the lock holder. Derived — gitignore it, or LFS-track it without `lockable`. |
| `*.png *.jpg *.jpeg *.tif *.tiff *.exr *.hdr *.tga *.dds *.iff *.bmp *.gif` | Read concurrently by every artist and every render. A lock blocks nothing useful, and read-only trips any tool that rewrites in place. Currently non-lockable — keep it that way. |
| `*.abc *.usd *.usda *.usdc *.usdz *.fbx *.obj *.vdb *.ass *.rs *.vrmesh *.mcx *.dae *.glb *.gltf *.ptc` | Published outputs, overwritten wholesale by a publish step. A read-only bit breaks the publisher's in-place write. Currently non-lockable — keep it that way. |
| `*.mogrt *.ffx *.otf *.ttf *.ttc *.zip *.7z *.rar` | Shared read-only-by-nature assets nobody edits in place. Locking adds ceremony with no benefit. |
| Audio and video (`*.wav *.mov *.mp4` …) | Delivered plates and reference. Replaced by new versions, not edited in place. |
| Anything gitignored | Not chmod'ed by default, and locks on them are unenforceable since they are never pushed. |
| Any directory | Impossible: `cannot lock directory`. |

**Keep lockable — the current set is right:** `*.ma *.mb *.max *.c4d *.blend
*.lxo *.hip *.hipnc *.aep *.aepx *.aet *.prproj *.prel *.sesx *.psd *.psb *.ai`.
These are genuinely unmergeable, human-authored, single-editor files, and the
applications that own them tolerate read-only on open.

## Sources

- `man git-lfs-lock`, `man git-lfs-unlock`, `man git-lfs-locks`,
  `man git-lfs-track`, `man git-lfs-config` — git-lfs 3.7.1, installed locally
- [Git LFS File Locking API spec](https://github.com/git-lfs/git-lfs/blob/main/docs/api/locking.md)
- [`commands/command_lock.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_lock.go)
- [`commands/command_unlock.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_unlock.go)
- [`commands/command_track.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_track.go)
- [`commands/command_post_checkout.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_post_checkout.go)
- [`commands/command_post_commit.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/command_post_commit.go)
- [`commands/uploader.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/uploader.go)
- [`commands/lockverifier.go`](https://github.com/git-lfs/git-lfs/blob/main/commands/lockverifier.go)
- [`locking/locks.go`](https://github.com/git-lfs/git-lfs/blob/main/locking/locks.go)
- [`locking/lockable.go`](https://github.com/git-lfs/git-lfs/blob/main/locking/lockable.go)
- [`lfs/gitscanner.go`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner.go),
  [`lfs/gitscanner_refs.go`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner_refs.go),
  [`lfs/gitscanner_catfilebatchcheck.go`](https://github.com/git-lfs/git-lfs/blob/main/lfs/gitscanner_catfilebatchcheck.go)
- [`tools/filetools.go`](https://github.com/git-lfs/git-lfs/blob/main/tools/filetools.go)
- [Arnold User Guide — Textures (settings)](https://help.autodesk.com/cloudhelp/ENU/AR-Core/files/ac-render-settings/arnold_user_guide_ac_render_settings_ac_textures_settings_html.html)
- [Autodesk — How to enable Auto-Convert Textures to TX in Maya with Arnold](https://knowledge.autodesk.com/support/maya/troubleshooting/caas/sfdcarticles/sfdcarticles/How-to-enable-Auto-Covert-Textures-to-TX-in-Maya-with-Arnold.html)
- [Autodesk — The file that is being saved is set to read-only](https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/The-file-that-is-being-saved-is-set-to-read-only.html)
- [Maya Help — Files/Projects preferences](https://help.autodesk.com/view/MAYAUL/2024/ENU/?guid=GUID-8E332DD5-879C-4AFD-8C6C-7C403FEF310F)
- [Adobe — Project types in After Effects](https://helpx.adobe.com/after-effects/using/projects.html)
- [Adobe Community — file is locked, Photoshop](https://community.adobe.com/t5/photoshop/cant-save-a-file-quot-because-the-file-is-locked-you-do-not-have-access-permissions-quot/m-p/9570391)
- [Creative COW — AE project is locked](https://creativecow.net/forums/thread/ae-will-not-let-me-save-project-is-locked/)
