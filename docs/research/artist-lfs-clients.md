# Artist-facing Git LFS clients: buy or build?

Research for issue #5. Date: 2 September 2026. All claims cite primary sources — vendor
documentation, vendor pricing pages, GitHub documentation, the Git LFS specification, and
maintainer statements in the tools' own issue trackers.

## The question

Thunderbird needs a workflow an animation student can run without learning Git:

- **Trunk with locks.** Push straight to `main`. No pull requests.
- **`start`** — pick a shot or asset folder, lock every lockable file in it.
- **`done`** — one-line description, commit, push, release all locks.
- **On failure, stop and preserve.** Never auto-resolve a binary conflict.
- Braeden maintains it alone.
- Buying is acceptable within reason.

Should an existing artist-facing client replace a custom `start`/`done` script?

## Short answer

**No. Build the script.** Buy a GUI only as an optional viewer alongside it.

Two facts decide this, and neither is about any particular vendor:

1. **Folder locking does not exist in Git LFS.** Not in any client — the protocol has no
   directory concept. Every product that offers it loops over files. So the `start`
   requirement is a script no matter what you buy.
2. **The one tool built for exactly this use case does not use Git LFS locks.** Anchorpoint
   runs its own lock service. Its locks are invisible to GitHub, to `git lfs locks`, and to
   anyone not running Anchorpoint.

Everything else follows from those.

---

## Constraint 1: Git LFS has no folder locking

The Git LFS locking API is per-file, full stop.

- `git lfs lock` takes a single `<path>`. No directories, no globs, no multiple paths. Its
  only options are `--remote` and `--json`.
  <https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-lock.adoc>
- `git lfs unlock` takes one `<path>` or `--id`, plus `--force`.
  <https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-unlock.adoc>
- The API spec describes only per-path locks — "files are locked exclusively to one user" —
  with paths like `"path": "foo/bar.zip"`. There is no directory-level lock, and none
  proposed. <https://github.com/git-lfs/git-lfs/blob/main/docs/api/locking.md>
- Locking is advisory. A server that does not implement it returns 404 and pushes are not
  halted. GitHub does implement it, which is why the current README workflow works.

**Consequence:** "lock every lockable file in this folder" is, in every possible
implementation, "enumerate the folder, filter by the `lockable` patterns in
`.gitattributes`, and issue N lock calls." That loop is the `start` script. No purchase
removes it.

## Constraint 2: no mainstream Git GUI takes LFS locks

Locking has been an open request across the GUI ecosystem for roughly nine years.

| Client | LFS lock/unlock in UI | Evidence |
|---|---|---|
| GitHub Desktop | **No** | Feature requests <https://github.com/desktop/desktop/issues/2437> and <https://github.com/desktop/desktop/issues/8419>, both closed. The one implementation attempt, PR <https://github.com/desktop/desktop/pull/8907>, was **closed unmerged** by maintainer outofambit on 23 Jan 2020 |
| Sourcetree | **No** | Open, unresolved feature requests SRCTREE-4663 (<https://jira.atlassian.com/browse/SRCTREE-4663>) and SRCTREEWIN-6963 (<https://jira.atlassian.com/browse/SRCTREEWIN-6963>); Atlassian community thread confirming no locking support: <https://community.atlassian.com/forums/Sourcetree-questions/Now-that-GIT-LFS-supports-File-locking-will-Source-Tree-support/qaq-p/599215> |
| GitButler | **No** — no Git LFS support at all | Tracking issue open, labelled for V3: <https://github.com/gitbutlerapp/gitbutler/issues/2595> |
| GitKraken | **No** | LFS menu offers only Initialize, Pull LFS files, Prune. <https://help.gitkraken.com/gitkraken-desktop/git-lfs/> |
| Tower | **No** (undocumented) | LFS help covers tracking patterns and "Use LFS Clone" only. <https://www.git-tower.com/help/guides/integration/git-lfs/mac> |
| **Fork** | **Yes** | "With Fork, you can now lock and unlock files from LFS menu in file context menu, avoiding potential binary merge conflicts in LFS files." <https://fork.dev/blog/posts/forkwin-1.48/> |

Fork is the only general-purpose client found that takes and releases real Git LFS locks.

---

## Tool-by-tool findings

### Anchorpoint

The purpose-built artist tool, and the closest thing to a turnkey answer — with one
disqualifying caveat.

- **Installs Git and Git LFS:** effectively yes. Its bundled terminal "uses Anchorpoint's
  version of Git LFS" (<https://docs.anchorpoint.app/git/settings-and-commands/>), and
  marketing states "No need to configure Git LFS. Anchorpoint does that for you."
  (<https://www.anchorpoint.app/>). It writes `.gitattributes` automatically. No page says
  "install Git first." Treat as high-confidence inference, not a quoted guarantee — verify
  on a clean machine.
- **GitHub sign-in:** yes, OAuth. Org admins must approve the Anchorpoint OAuth app.
  <https://docs.anchorpoint.app/docs/version-control/troubleshooting/github/>
- **Plain GitHub repo:** yes. "Connect existing repository via https."
  <https://docs.anchorpoint.app/docs/version-control/first-steps/existing-projects/> There
  is no Anchorpoint file storage: "Production files: Not uploaded to Anchorpoint's cloud."
  <https://www.anchorpoint.app/policies/facts>
- **Clone:** yes, including reconciling files already on disk by SHA, and shallow clone.
- **Locks — the dealbreaker:** "Anchorpoint does not support Git LFS file locking. It uses
  its own system which uses the Anchorpoint metadata server to share locking information."
  And: "Git LFS file locking is an entirely different system and should not be used with
  Anchorpoint file locking." <https://docs.anchorpoint.app/git/file-locking/>
  Locks live in Anchorpoint's AWS Frankfurt metadata service and are enforced only for
  teammates who are also running Anchorpoint.
- **Lock ergonomics (within its own system):** excellent. Binary files auto-lock on
  modification; locks release automatically on push ("These locks are removed as soon as
  the files are pushed to the remote server. However, the rest of the team must pull the
  files before the read-only protection is removed"). Files **or folders** can be locked
  from the right-click menu
  (<https://docs.anchorpoint.app/docs/general/workspaces-and-projects/file-locking/>),
  though no documented recursive or sticky folder lock covers files added later.
- **OS:** "Windows and macOS (macOS 12+)." No published minimum Windows version.
  <https://www.anchorpoint.app/policies/facts>
- **Trunk, no PRs:** yes. Pull requests appear nowhere in the Git docs; the primary verb is
  Sync (commit + fetch + pull + push). Their Unity guide recommends trunk-based development
  explicitly. <https://docs.anchorpoint.app/git/unity/>
- **Failure behavior:** stop and preserve. Conflicts require a manual left/right version
  choice; "In Anchorpoint you are mostly committing your files before resolving a conflict,
  which means that files will never get lost."
  <https://docs.anchorpoint.app/docs/version-control/features/resolving-conflicts/>
  Caution: with the Rebase merge strategy the conflict dialog appears once per local commit.
- **Cost:** Personal tier is free but **single user, non-commercial, and has no file
  locking**. Team is €20/user/month or €240/user/year. Education is "the full functionality
  of the Team Plan for an annual subscription of €1999 for up to 100 students and teachers."
  <https://www.anchorpoint.app/pricing> For 10 students: €2,400/year at list, or €1,999/year
  flat on the education plan. There is no free path for a 10-person team.
- **DCC integrations:** Maya plugin exists (adds a publish step; "the plugin won't modify
  your Maya file"). <https://docs.anchorpoint.app/integrations/maya/> Also Blender, C4D,
  Unity, Unreal. **No After Effects integration** — AE files would rely on generic
  auto-locking in the file browser.

**Verdict:** the best artist UX by a distance, but adopting it means Thunderbird's locks
stop being Git LFS locks. The `.gitattributes` `lockable` flags, `git lfs locks`, the
read-only enforcement described in the README, and any student on the CLI all become
irrelevant or actively misleading. It also costs ~€2,000/year and adds a dependency on
Anchorpoint's cloud being up.

### GitHub Desktop

- **Installs Git and Git LFS: yes, both.** "When you install GitHub Desktop, Git Large File
  Storage (Git LFS) is installed, too."
  <https://docs.github.com/en/desktop/configuring-and-customizing-github-desktop/about-git-large-file-storage-and-github-desktop>
  Confirmed in the shipped Git distribution, `desktop/dugite-native`, which bundles Git LFS.
  Caveat from the same page: "To use Git LFS with GitHub Desktop, you must configure Git LFS
  using the command line" — already done here, since `.gitattributes` is committed.
- **GitHub sign-in, clone:** yes to both, native.
  <https://docs.github.com/en/desktop/adding-and-cloning-repositories/cloning-a-repository-from-github-to-github-desktop>
- **Locks: no.** See the table above. The abandoned locking PR documents why this is hard in
  Desktop's UI model: "There is no way to view or change existing locks for files that aren't
  part of the changes list," and locked files could still be selected for commit, failing only
  with an error popup. <https://github.com/desktop/desktop/pull/8907> Desktop lists only
  *changed* files, so there is nowhere to select an unmodified file to lock.
- **OS:** Windows and macOS. **Cost:** free, open source.
  <https://docs.github.com/en/desktop/overview/about-github-desktop>
- **Trunk, no PRs:** yes. Pull requests are optional.
- **Risk:** it nudges users into merging binary conflicts, the exact failure mode the
  lock workflow exists to prevent.

### Sourcetree

- **Installs Git / Git LFS:** partly, and **not fully verified**. Sourcetree ships an embedded
  Git and offers a "Use System Git LFS" option, and its site states "Sourcetree supports Git
  LFS, allowing teams to track large assets in one single place" (<https://www.sourcetreeapp.com/>).
  Whether Git LFS itself is bundled on each platform was not confirmed from Atlassian
  documentation — treat as an open item and test on a clean machine.
- **GitHub sign-in, clone:** yes, via account connection in the app.
- **Locks: no** (see table). Sourcetree's Custom Actions feature can bind arbitrary shell
  commands to menu entries, so lock/unlock buttons could be wired up by hand — but with no
  lock-state column, no lock-holder display, no commit-time guard, and no folder handling,
  that is scripting around the gap, not a feature.
- **OS:** Windows and macOS. **Cost:** free. <https://www.sourcetreeapp.com/>
- **Risk:** a much larger, more intimidating Git surface than students need.

### GitButler

**Rule out.** It has no Git LFS support at all. The tracking issue
(<https://github.com/gitbutlerapp/gitbutler/issues/2595>) is still open and labelled for the
V3 release. The stated cause is architectural: GitButler's checkout goes through `git2`,
which does not run configured worktree filters — and Git LFS *is* a clean/smudge filter — so
LFS files surface as spurious changes. A rewrite onto gitoxide is the planned fix. Until
that lands, pointing GitButler at an LFS repo risks mangling pointer files rather than merely
lacking a feature. It also layers virtual branches on top of Git: more for a student to
learn, not less.

### Diversion

**Not Git.** "Diversion is not Git-based."
<https://www.diversion.dev/compare-diversion-to-git-lfs> It is a separate cloud VCS with a
Git-like interface; the repo lives on Diversion's cloud, not GitHub. A one-time Git import
exists (<https://docs.diversion.dev/basic/import-from-git>), and bidirectional GitHub
mirroring is marketed (<https://www.diversion.dev/github-mirroring>) but is a
request-early-access page, and the CLI sync flag requires contacting support.

Its locking is the best on offer: hard locks give "exclusive write access to a specific path"
and apply to **files and folders**, with auto-lock by extension
(<https://docs.diversion.dev/hard-locks>). Windows and macOS desktop apps
(<https://docs.diversion.dev/quickstart>). Pricing: free for the first 5 users, Pro $25/user/
month, and **hard locks require the Studio tier at $35/user/month**; education is "free for
up to 10 users" (<https://www.diversion.dev/pricing>) — confirm in writing whether that
education grant includes hard locks.

Adopting Diversion means abandoning GitHub as the source of truth. That is a bigger decision
than replacing a shell script, and it is outside the settled requirements.

### Other tools checked

- **Fork** — the only client with real LFS lock/unlock in the UI
  (<https://fork.dev/blog/posts/forkwin-1.48/>). $59.99 per seat, perpetual, macOS 10.11+ /
  Windows 7+, free evaluation, no advertised education discount (<https://fork.dev/>).
  ~$600 for 10 seats. Does not bundle Git or Git LFS; locks one file at a time; no folder
  locking; no GitHub-specific onboarding beyond ordinary clone.
- **Tower** — free licenses for students, teachers, and educational institutions; one license
  covers macOS and Windows; LFS tracking supported but **no documented locking UI**.
  <https://www.git-tower.com/pricing>
- **GitKraken** — no LFS locking; does not install Git LFS for you (requires ≥3.0.0
  separately); free tier excludes private repos. <https://help.gitkraken.com/gitkraken-desktop/git-lfs/>
- **Snowtrack** — acquired by Perforce, rebranded P4 One; `snowtrack.io` now redirects to
  Perforce. Not Git. <https://www.perforce.com/press-releases/announcing-p4-one>
- **Unity Version Control (ex-Plastic SCM)** — not Git; ties you to Unity's cloud.
- **Maya / After Effects Git front ends** — none credible exist. The only Maya Git plugin
  found (<https://github.com/renee0506/MayaGit>) is a zero-star personal prototype. The AE
  ecosystem tools are filename incrementers, not version control. Anchorpoint's "Git for
  Maya" marketing describes a standalone desktop app, not a Maya plugin.
- **Prism Pipeline** — a pipeline manager with Maya and AE plugins and a usable free tier
  (<https://prism-pipeline.com/plans/>), but it is not version control and its docs mention
  no VCS integration. Potentially complementary; not a substitute.
- **AYON** — its version control addon targets Perforce, not Git LFS.
  <https://help.ayon.app/en/articles/2946473-version-control-introduction>

---

## GitHub-side facts the decision needs

- **Free LFS quota:** GitHub Free and Pro get 10 GiB storage and 10 GiB bandwidth. **GitHub
  Team and Enterprise Cloud get 250 GiB each.**
  <https://docs.github.com/en/billing/concepts/product-billing/git-lfs>
- **Overage is metered**, billed per GiB with budget controls; current rates via
  <https://github.com/pricing/calculator?feature=lfs>.
- **Max LFS file size:** 2 GB on Free/Pro, 4 GB on Team, 5 GB on Enterprise Cloud.
  <https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-git-large-file-storage>
- **Verified academic faculty can apply for GitHub Team** for teaching or academic research.
  <https://docs.github.com/en/billing/concepts/discounted-plans>

For a film's worth of Maya scenes and AE comps, 10 GiB will not last. Getting the repo under
a GitHub Team org via the education route is likely worth more to this project than any
client purchase — it is a 25× quota increase and a 2× file-size ceiling increase.

## Scorecard

| | Installs Git + LFS | GitHub sign-in | Clone | **Real LFS locks** | Folder lock | mac + Win | Plain GitHub repo | 10 students |
|---|---|---|---|---|---|---|---|---|
| Custom `start`/`done` script | No (README step) | Uses `gh`/credential helper | One command | **Yes** | Yes, by loop | Yes | Yes | **$0** |
| Anchorpoint | Yes | Yes | Yes | **No — proprietary locks** | Yes (own system) | Yes (macOS 12+) | Yes | €1,999/yr education |
| GitHub Desktop | **Yes** | Yes | Yes | No | No | Yes | Yes | Free |
| Sourcetree | Unconfirmed | Yes | Yes | No (Custom Actions hack) | No | Yes | Yes | Free |
| Fork | No | Yes | Yes | **Yes** | No | Yes | Yes | ~$600 one-time |
| GitButler | No | Yes | Yes | No LFS at all | No | Yes + Linux | Yes | Free |
| Diversion | n/a — not Git | Bridge only | n/a | Own hard locks | **Yes** | Yes | **No** | Free ≤10 (education) |
| Tower | No | Yes | Yes | No | No | Yes | Yes | Free (education) |

## What a student must still understand, under every option

No tool removes all of this:

1. A commit is a checkpoint, and pushing is a separate act from committing.
2. Pull before push, and a rejected push means someone else moved first.
3. A binary conflict means someone's work is discarded — it cannot be merged.
4. A lock is a claim other people can see, and it must be released or it blocks the team.
5. LFS pointer files exist and can leak into the working copy if LFS is not set up.
6. The repo has a storage quota that costs money.

Anchorpoint hides the most of this (one Sync button, auto-lock, auto-unlock on push) but
adds its own concept a student must learn: that locks are Anchorpoint-only, and that
teammates must pull before read-only lifts. GitHub Desktop hides commit and push mechanics
but leaves locking as a memorized terminal ritual. Fork hides nothing but surfaces locks.

## Recommendation

**Build the `start`/`done` script. Recommend GitHub Desktop alongside it as an optional
viewer. Do not buy Anchorpoint.**

Reasoning:

1. **The folder-lock requirement is a loop over `.gitattributes` patterns, always.** Git LFS
   has no directory locking. That code has to be written regardless, so the script's core
   value cannot be bought away.
2. **Anchorpoint is the only serious "buy," and it trades away Git LFS locking itself.** For
   ~€2,000/year, Thunderbird would replace a standard, portable, GitHub-native mechanism with
   a vendor metadata service that is invisible to `git lfs locks` and to any student not
   running the app. That is a worse position than today, not a better one.
3. **Every free GUI fails the locking requirement outright**, and GitHub Desktop actively
   encourages merging binary conflicts — the specific failure mode the requirements say must
   stop and preserve.
4. **The one client with real locks (Fork, ~$600) still locks one file at a time**, and buying
   it would not remove the script; it would only give students a way to see and release locks
   manually.
5. **GitHub Desktop is a free, zero-risk complement.** It installs Git and Git LFS on both
   platforms, handles GitHub OAuth and cloning, and gives students a visual working-copy view.
   Use it for install, sign-in, and clone; use the script for `start` and `done`. That splits
   the work exactly along each tool's strength and costs nothing.
6. **Maintenance load is low and matches "Braeden maintains alone."** The script wraps six
   Git LFS and Git commands with a `.gitattributes` glob match, an error check, and a
   confirm prompt. It has no server, no accounts, and no subscription to renew after the
   students graduate.

**Spend the budget on GitHub Team instead.** The education route to a Team org raises the LFS
quota from 10 GiB to 250 GiB and the per-file ceiling from 2 GB to 4 GB. That constraint will
bite this project long before client ergonomics do.

**Revisit if:** GitHub Desktop revives LFS locking (watch desktop/desktop#8419); Anchorpoint
adds real Git LFS lock interoperability; or the team decides GitHub itself is negotiable, in
which case Diversion's education tier deserves a proper evaluation.

## Facts the build-versus-buy decision needs

| Question | Answer |
|---|---|
| Can any tool lock a folder via Git LFS? | No. The protocol has no directory locks. |
| Can any free GUI take an LFS lock? | No. Only Fork, at $59.99/seat. |
| Does the purpose-built artist tool use Git LFS locks? | No. Anchorpoint uses its own service. |
| Cost to buy for 10 students | Anchorpoint €1,999/yr (education) or €2,400/yr (list); Fork ~$600 one-time; Diversion free ≤10 (education, hard locks unconfirmed); GitHub Desktop, Sourcetree, Tower free |
| Cost to build | One script, no recurring cost, maintained by one person |
| What buying removes | Install and sign-in friction (GitHub Desktop does this for free) |
| What buying does not remove | The folder-lock loop, the failure policy, and every Git concept in the list above |
| Biggest unaddressed risk | The 10 GiB LFS quota on a GitHub Free account |

## Open items to verify before committing

- Whether Anchorpoint bundles Git itself on a clean machine (inferred, not documented).
- Whether Sourcetree bundles Git LFS on either platform, or expects a system install.
- GitHub's current metered LFS rates, via the pricing calculator.
- Whether UVU qualifies for a GitHub Team education org for this repo.
