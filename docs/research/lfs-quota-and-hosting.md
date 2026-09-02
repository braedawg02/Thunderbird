# Git LFS quotas and hosting options for Thunderbird

Research note for issue #6. Written 2026-09-02. Every quota and price below was
read from the vendor's own documentation on that date; each claim carries its
source URL. Re-verify before making a purchase — GitHub changed this policy
once already in 2025 and its own marketing pages still disagree with its docs
(see "Discrepancy to be aware of").

---

## 1. Question

Thunderbird lives in a public repo, `braedawg02/Thunderbird`, owned by a
personal GitHub account. The workflow depends on Git LFS **and** on the LFS
file-locking API: `.gitattributes` marks Maya, After Effects, Premiere, and
Photoshop files `lockable`, and the README instructs everyone to
`git lfs lock` before editing a scene.

What are GitHub's LFS quotas now, what breaks when they run out, what does
extra capacity cost, and is there a better place to keep the bytes?

---

## 2. GitHub's current LFS policy

### 2.1 Free allowances

Per-account, per-month, for **both** storage and download bandwidth:

| Plan | Bandwidth | Storage |
| --- | --- | --- |
| GitHub Free (personal) | 10 GiB | 10 GiB |
| GitHub Pro (personal) | 10 GiB | 10 GiB |
| GitHub Free for organizations | 10 GiB | 10 GiB |
| GitHub Team | 250 GiB | 250 GiB |
| GitHub Enterprise Cloud | 250 GiB | 250 GiB |

Source: [Git Large File Storage billing](https://docs.github.com/en/billing/concepts/product-billing/git-lfs)
(GitHub Docs, read 2026-09-02), confirmed against the
[GitHub Enterprise Cloud copy of the same page](https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/git-lfs).

These are the post-2025 numbers. The old policy was 1 GB storage and 1 GB
bandwidth plus $5/month prepaid "data packs" of 50 GB each. GitHub replaced
that with metered billing and raised the free tier at the same time; the
transition is tracked on the public roadmap as
[github/roadmap#237, "Git LFS moves to metered billing"](https://github.com/github/roadmap/issues/237)
and explained in
[community discussion #61362, "LFS to Metered Billing FAQ"](https://github.com/orgs/community/discussions/61362).
The billing platform that carries it reached new personal accounts on
2025-02-13 and existing personal accounts by 2025-06-30, per
[Enhanced billing platform is now available for personal accounts](https://github.blog/changelog/2025-02-25-enhanced-billing-platform-is-now-available-for-personal-accounts/)
(GitHub Changelog, 2025-02-25). Enterprise accounts got metered LFS earlier, in
[New Enterprise accounts have metered billing for Git LFS](https://github.blog/changelog/2024-06-03-new-enterprise-accounts-have-metered-billing-for-git-lfs/)
(2024-06-03).

**Practical consequence for Thunderbird: the relevant number is 10 GiB, not
250 GiB.** The repo is owned by a personal account, and a personal account
cannot be on Team.

### 2.2 What counts, and against whom

From [About storage and bandwidth usage](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-storage-and-bandwidth-usage):

- **The repo owner pays for everyone.** "Bandwidth and storage usage always
  count against the repository owner's account. Forking and pulling a
  repository counts against the parent repository's bandwidth usage." Ten
  students cloning bills Braeden, not the students. So does every anonymous
  stranger who clones the public repo.
- **Quota is per account, not per repo.** It is shared with every other repo
  that account owns.
- **Pushes cost storage, not bandwidth.** Downloads cost bandwidth.
- **Every version is stored forever.** "If you make a 1 byte change and push
  the file again, you'll use another 500 MB of storage." A `lockable` scene
  file that gets re-pushed on every "done" accumulates a full copy per push.
- Storage is metered hourly, then averaged: GitHub's worked example is 1 GiB
  for 15 days plus 2 GiB for 15 days = 1,080 GiB-hours ÷ 720 = 1.5 GiB billed.
- Bandwidth resets at the start of each billing cycle; the accrued storage
  total also resets to zero each cycle.

The storage-is-forever rule is sharpened by
[Removing files from Git Large File Storage](https://docs.github.com/en/repositories/working-with-files/managing-large-files/removing-files-from-git-large-file-storage):
"After you remove files from Git LFS, the Git LFS objects still exist on the
remote storage and will continue to count toward your Git LFS storage quota."
The only remedies GitHub documents are to **delete and recreate the
repository**, or contact support. There is no "purge old versions" button.
Plan accordingly: a bad push of a 40 GB cache directory is permanent.

### 2.3 What happens when you exceed the quota

Three distinct outcomes, from
[Git Large File Storage billing](https://docs.github.com/en/billing/concepts/product-billing/git-lfs):

1. **No payment method on file.** Storage overage: you cannot push new LFS
   files, and clones "will only retrieve the pointer files" — meaning
   collaborators get 130-byte text stubs where the Maya scenes should be, and
   Maya fails to open them. Bandwidth overage: "Git LFS support is disabled on
   your account until the next month."
2. **Payment method on file, budget set to $0.** "You are not charged for
   overages, but Git LFS usage is blocked for the rest of the calendar month."
   Same functional outcome as (1), with a hard stop instead of a surprise bill.
3. **Payment method on file, no budget.** "There is no spending limit, and you
   are billed for all usage beyond the free quota."

Failure mode (1) is the one that bites a student film: it is silent from the
pusher's side and looks like data loss from everyone else's. It is also
recoverable — the objects are not deleted, service resumes next cycle or on
payment.

### 2.4 Cost of extra capacity

Data packs are gone. [Upgrading Git Large File Storage](https://docs.github.com/en/billing/how-tos/products/upgrade-git-lfs-storage):
"Git LFS billing used pre-paid data packs. These have been removed and replaced
with metered billing and you only pay for what you actually use."

Metered rates beyond the free allowance:

| Meter | Price |
| --- | --- |
| LFS storage | **$0.07 per GiB per month** |
| LFS download bandwidth | **$0.0875 per GiB** |

Source: GitHub's [pricing calculator](https://github.com/pricing/calculator)
and the roadmap/FAQ threads above. The docs pages themselves state the billing
mechanism but defer the rates to the calculator, so treat these two figures as
the number to re-confirm in the calculator before committing.

Other prices relevant to the decision, from [github.com/pricing](https://github.com/pricing):
GitHub Team is **$4 per user per month**; GitHub Pro is $4 per month for a
personal account. Pro does **not** raise the LFS allowance — Free and Pro are
both 10 GiB. Paying for Pro buys nothing for this problem.

#### Discrepancy to be aware of

As of 2026-09-02 the public [github.com/pricing](https://github.com/pricing)
page still advertises Git LFS as "$5 per month for 50 GB bandwidth and 50 GB of
storage" — the retired data-pack model — while the billing docs say data packs
"have been removed." The docs are the newer and more specific source, so this
note follows the docs. If someone tries to buy a data pack and cannot find the
option, that is why; see
[community discussion #156455, "Git LFS data pack option not available"](https://github.com/orgs/community/discussions/156455).

### 2.5 Hard limits that are not quotas

From [About large files on GitHub](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)
and [About Git Large File Storage](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-git-large-file-storage):

- Non-LFS files: warning over 50 MiB, **blocked over 100 MiB**.
- Maximum size of a single LFS object: **2 GB on Free and Pro**, 4 GB on Team,
  5 GB on Enterprise Cloud. A 4K EXR sequence rendered to a single file, or a
  big Alembic cache, can reach this.
- "We recommend repositories remain small, ideally less than 1 GB, and less
  than 5 GB is strongly recommended." This is advice, not enforcement, and it
  concerns the Git object database rather than LFS storage — but a film repo
  will exceed it, and GitHub Support has historically contacted owners of very
  large repos.

---

## 3. What Thunderbird actually needs

### 3.1 Measured today

Run in this worktree on 2026-09-02:

| Measurement | Value |
| --- | --- |
| LFS pointers at `HEAD` (`git lfs ls-files -s`) | 47 entries, **21 unique objects**, 21.6 MiB |
| LFS objects across all refs (`git lfs ls-files -s --all`) | **23 unique objects, 37.8 MiB** |
| Total commits | 11 |
| Largest object | `assets/characters/Thunder-Bird/tb_with_guns.ma`, **18 MB** |
| Versions of that one file already stored | **2** (two distinct OIDs, ~36 MB total) |
| File types in LFS today | 18 `.png`, 3 `.ma`, 1 `.psd`, 1 `.jpeg` |

The repo is 0.4% of the free storage quota. Nothing is wrong today. The
question is entirely about trajectory.

The two-versions-of-one-rig fact is the whole argument in miniature: in eleven
commits, one file has already consumed 36 MB of permanent storage for 18 MB of
content. `.gitattributes` routes roughly sixty extensions into LFS, including
`.exr`, `.abc`, `.vdb`, `.mov`, `.braw`, `.psd`, and `.psb` — every heavyweight
format a 3D short produces except final renders, which `.gitignore` excludes.

### 3.2 A useful anchor

**The 10 GiB free storage allowance is about 569 pushes of the existing 18 MB
rig.** Across ten students over a fifteen-week term, that is roughly four
scene-file saves per person per week before storage alone is exhausted —
ignoring textures, caches, audio, and reference video entirely.

### 3.3 Projection

Assumptions, stated so they can be argued with: ten students, a fifteen-week
term, everyone clones at term start, and each scene file is re-pushed on every
"done." "Working set" is the size of the LFS objects reachable from `main` at
the end of term; "versions" is the average number of times each object was
re-pushed. Bandwidth is term-start clones plus ongoing pulls, and the monthly
overage figures spread the term's bandwidth over four months.

| Scenario | Working set | Avg versions | LFS storage | Term bandwidth |
| --- | --- | --- | --- | --- |
| Lean (asset library plus shot scenes, disciplined pushes) | 15 GiB | 4 | 60 GiB | ~450 GiB |
| Likely (adds textures, caches, audio, reference video) | 40 GiB | 6 | 240 GiB | ~1,150 GiB |
| Heavy (playblasts and EXR sequences in the repo) | 80 GiB | 8 | 640 GiB | ~2,600 GiB |

Cost on a **personal account (10 GiB free)** at $0.07/GiB-month storage and
$0.0875/GiB bandwidth:

| Scenario | Storage overage | Bandwidth overage | Approx. monthly | Approx. per term |
| --- | --- | --- | --- | --- |
| Lean | $3.50 | $8.97 | **~$12** | ~$50 |
| Likely | $16.10 | $24.28 | **~$40** | ~$160 |
| Heavy | $44.10 | $56.00 | **~$100** | ~$400 |

The headline finding: **at ten-student scale, GitHub LFS overage is cheap.**
Even the heavy scenario is about $100/month — comparable to one seat of most
production-asset SaaS. The danger is not the price, it is the *default*: with
no payment method, the account silently stops serving LFS objects mid-term and
the team sees pointer files instead of scenes.

Two caveats push the risk up:

- The repo is **public**. Anyone can clone it and spend the owner's bandwidth,
  and there is no per-repo rate limit to stop it. A single link on a forum
  during term is a plausible way to burn a month's allowance.
- Storage is cumulative and unreclaimable. Overage grows monotonically across
  terms even if the team gets more disciplined.

---

## 4. The locking constraint

This is what disqualifies most of the alternatives, so it goes first.

File locking is an **optional** part of the Git LFS protocol, not a core one. A
server implements four endpoints — `POST /locks`, `GET /locks`,
`POST /locks/verify`, `POST /locks/:id/unlock` — and the spec is explicit that
skipping them is legal: "By default, an LFS server that doesn't implement any
locking endpoints should return 404. This response will not halt any Git
pushes."
([git-lfs/docs/api/locking.md](https://github.com/git-lfs/git-lfs/blob/main/docs/api/locking.md))

That 404 is the trap. If the LFS endpoint does not implement locking,
`git lfs lock` fails but **pushes still succeed**. The team loses its only
protection against two people animating the same `.ma`, and loses it quietly —
no error at push time, just a clobbered scene. Thunderbird's README makes
locking the central rule of the workflow, so any host that does not implement
these four endpoints is not a candidate, however cheap its storage.

### 4.1 GitHub.com does implement locking — verified

Secondary write-ups and several git-lfs issue threads claim GitHub does not
support the locking API (for example
[git-lfs#6308](https://github.com/git-lfs/git-lfs/issues/6308) and
[git-lfs#3400](https://github.com/git-lfs/git-lfs/issues/3400), both of which
report `Remote "origin" does not support the Git LFS locking API`). That claim
is wrong for github.com, and it is worth nailing down because it would change
the whole decision.

Empirical check against this repo's real remote on 2026-09-02:

```
$ git lfs locks
assets/characters/Thunder-Bird/tb_with_guns.ma	braedawg02	ID:49946906
```

The `/locks` endpoint on github.com returned a live lock. GitHub implements the
locking API, and the team is already using it — there is currently an
outstanding lock on the rig. Those error reports come from other remotes
(Netlify Large Media, old GitHub Enterprise Server versions, custom `lfs.url`
endpoints), not from github.com.

Sources for the errors themselves:
[git-lfs#3400](https://github.com/git-lfs/git-lfs/issues/3400),
[desktop/desktop#2812](https://github.com/desktop/desktop/issues/2812).
The per-endpoint escape hatch is `lfs.<url>.locksverify`, documented in
[git-lfs-config.adoc](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-config.adoc).

### 4.2 Locking follows the LFS endpoint, not the Git remote

Second, and less obvious: **locking follows the LFS endpoint, not the Git
remote.** If you keep the repo on GitHub but point `lfs.url` at a third-party
LFS server via `.lfsconfig`, `git lfs lock` talks to that third-party server.
GitHub's locking implementation is no longer in the picture. So "GitHub for
code, someone else's bucket for bytes" only works if the bucket's LFS server
implements locking — which the cheap S3-proxy-style servers generally do not.

GitHub documents the redirection itself
(`git config -f .lfsconfig remote.origin.lfsurl https://THIRD-PARTY-LFS-SERVER/path/to/repo`,
committed as `.lfsconfig` so every collaborator picks it up) in
[Configuring Git Large File Storage for your enterprise](https://docs.github.com/en/enterprise-server@3.17/admin/managing-accounts-and-repositories/managing-repositories-in-your-enterprise/configuring-git-large-file-storage-for-your-enterprise);
`git lfs env` shows which endpoint is in effect.

---

## 5. Alternatives

Every option below was checked against one question first: does it implement
the four `/locks*` endpoints? Options that fail that test are listed anyway,
because knowing *why* they fail is what stops someone adopting one by mistake.

### 5.1 Keep the repo on GitHub, move only the LFS objects

GitHub documents the redirection and it works. The problem is what goes with
it: because lock URLs are derived from `lfs.url`, `git lfs lock` starts talking
to the new server, and GitHub's locking implementation drops out. There is no
documented way to split objects onto a third-party server while keeping locks
on GitHub.

So this option is only viable if the replacement server implements locking —
and the cheap object-storage-backed servers, which are the whole reason to
consider it, do not:

| Server | Locking | Evidence | Backend |
| --- | --- | --- | --- |
| [lfs-test-server](https://github.com/git-lfs/lfs-test-server) | **Yes** | Route table in [`server.go`](https://github.com/git-lfs/lfs-test-server/blob/main/server.go) registers `GET/POST /{user}/{repo}/locks`, `/locks/verify`, `/locks/{id}/unlock` | Local disk + boltdb |
| [Rudolfs](https://github.com/jasonwhite/rudolfs) | **No** | No lock routes in the source tree; README documents features and non-features, never mentions locking. Also has **no client authentication by design** | S3 or local disk |
| [lfs-s3](https://github.com/nicolas-graves/lfs-s3) | **Not applicable** | It is a `lfs.customtransfer` agent, not a server — it bypasses the batch/locking HTTP API entirely, so locking is out of scope by construction | S3 |
| [Giftless](https://github.com/datopian/giftless) | **No** | [`giftless/view.py`](https://github.com/datopian/giftless/blob/main/giftless/view.py) contains no lock routes or handlers; no lock module in the package | Local, S3, Azure, GCS |
| [Estranged.Lfs](https://github.com/alanedwardes/Estranged.Lfs) | **No** | README describes only auth plus blob-store adapters issuing presigned URLs | S3, Azure Blob |

`lfs-test-server` is the only community server here that locks, and its own
name says what it is for. **Verdict: this is the trap option.** It looks like
the best of both worlds — GitHub's UI and PRs, someone else's cheap bytes — and
it quietly costs the team the one guarantee the workflow is built on.

### 5.2 Move the whole repo: hosts that do implement locking

**Gitea / Forgejo (self-hosted) — yes.** Gitea implements the full set in
[`services/lfs/locks.go`](https://github.com/go-gitea/gitea/blob/main/services/lfs/locks.go)
(`GetListLockHandler`, `PostLockHandler`, `VerifyLockHandler` with ours/theirs
partitioning, `UnLockHandler` with force), added in
[PR #2938](https://github.com/go-gitea/gitea/pull/2938) and covered by
integration tests. Forgejo is a Gitea fork and inherits this; that is lineage
inference, not a fresh read of Forgejo's current source, so confirm before
committing. LFS objects can sit on local disk or any S3-compatible bucket
including R2 and B2, configured separately from Git storage
([Forgejo storage docs](https://forgejo.org/docs/latest/admin/setup/storage/)).
Cost is a small VM plus near-zero object storage. Cost in *time* is a server
someone has to keep alive across a term, patched, backed up, and reachable from
the lab — with no faculty successor when the student who set it up graduates.

[Codeberg](https://codeberg.org) is hosted Forgejo and free, but its quota is
**1.5 GiB combined** across LFS, packages, releases, and attachments, with a
750 MiB repo limit ([Codeberg LFS docs](https://docs.codeberg.org/git/using-lfs/)).
Increases are possible by request but this is not a multi-GB animation host.

**GitLab.com — yes, on the Free tier.** GitLab distinguishes two features, and
only one of them is paid:

- **Exclusive file locks** — the real LFS locking API, driven by
  `git lfs lock` and the `lockable` gitattribute, "prevent modifications to
  locked files on any branch." **Available on Free, Premium, and Ultimate.**
- **Default branch file and directory locks** — GitLab's own UI-based locking,
  default branch only. Premium and Ultimate only.

Source: [File locking](https://docs.gitlab.com/user/project/file_lock/);
implementation history at
[gitlab-foss#35856](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/35856).

Free-tier limit on gitlab.com is **10 GiB per project covering repository plus
LFS combined**, and exceeding it makes the project **read-only**
([Storage usage quotas](https://docs.gitlab.com/user/storage_usage_quotas/)).
That is a harder wall than GitHub's, because it is per project rather than per
account and read-only is worse than pointer-only. No documented egress cap,
which is the one place GitLab Free beats GitHub Free outright.

**GitLab for Education** is the reason GitLab is on the shortlist: free
**Ultimate**, unlimited seats, 50,000 CI minutes/month, renewable annually, for
accredited degree-granting institutions doing instructional, non-commercial
work — [GitLab for Education](https://about.gitlab.com/solutions/education/),
[community programs](https://docs.gitlab.com/subscriptions/community_programs/).
UVU and a student film both fit that description. Confirm what storage quota
the Education Ultimate grant carries, since the published 10 GiB figure is the
Free-tier number.

### 5.3 Hosts that do not implement locking

**Bitbucket Cloud — definitively no.** Atlassian states it plainly: "At this
time, Bitbucket Cloud does not support Git LFS lock functionality."
([Current limitations for Git LFS with Bitbucket](https://support.atlassian.com/bitbucket-cloud/docs/current-limitations-for-git-lfs-with-bitbucket/)).
The enhancement request [BCLOUD-14454](https://jira.atlassian.com/browse/BCLOUD-14454)
has been open since 2017-06-23. Note the split: Bitbucket **Server / Data
Center** does support LFS locking. Cloud does not. Disqualified.

**Azure DevOps Repos — unconfirmed, lean no.** LFS itself is supported and
free ([Manage large files](https://learn.microsoft.com/en-us/azure/devops/repos/git/manage-large-files?view=azure-devops)),
but that page's limitations list never claims the `/locks*` endpoints are
served, and mentions locking only by pointing at upstream git-lfs. The closest
vendor-adjacent statement is a Microsoft staff reply on
[MS Learn Q&A #596027](https://learn.microsoft.com/en-us/answers/questions/596027/how-do-i-lock-files-in-a-git-repository-to-prevent)
naming GitHub and GitLab as providers that support LFS locking and omitting
Azure DevOps; a direct follow-up went unanswered. That is a community reply,
not documentation, so this stays "unconfirmed" rather than "no." If Azure
becomes a serious candidate, the decisive test takes a minute: push a test repo
and run `git lfs lock somefile`. A 404 from `/locks` is the spec's
not-implemented signal. Other limits worth knowing:
recommended repo under 10 GB, hard cap 250 GB, **5 GB maximum per push**, free
tier of 5 Basic users
([Azure Repos limits](https://learn.microsoft.com/en-us/azure/devops/repos/git/limits?view=azure-devops)).

### 5.4 Anchorpoint — it replaces locking rather than hosting it

This inverts the premise of the question, so read it carefully.

> "Anchorpoint does not support Git LFS file locking. It uses its own system
> which uses the Anchorpoint metadata server to share locking information."
> — [Anchorpoint file locking docs](https://docs.anchorpoint.app/docs/general/workspaces-and-projects/file-locking/),
> [Git file locking](https://docs.anchorpoint.app/git/file-locking/)

Anchorpoint's docs actively advise against combining `git lfs lock` and the
`lockable` attribute with Anchorpoint's own locking. Two consequences:

- Anchorpoint locks work on **any** remote — GitHub, GitLab, Bitbucket, Azure,
  self-hosted Gitea — because locking never touches the remote. Under
  Anchorpoint, Bitbucket Cloud's missing locking API stops mattering.
- **Anyone outside the Anchorpoint app sees nothing.** Plain `git lfs lock`,
  GitHub Desktop, the CLI, CI — none of them see or respect Anchorpoint locks,
  and Anchorpoint does not see theirs. A mixed-client team gets silent lock
  bypass, which is the exact failure the `lockable` flag exists to prevent.

Anchorpoint is bring-your-own-storage: it holds metadata (locks, comments,
tags) and you still provision the Git and LFS host yourself
([Git servers](https://docs.anchorpoint.app/docs/general/file-organization/git-servers/)).
It does not solve the quota problem. Pricing
([anchorpoint.app/pricing](https://www.anchorpoint.app/pricing)): Personal free
for a single non-commercial user; Team €20/user/month with no bundled storage;
**Education €1,999/year for up to 100 students and teachers.**

The rule this produces: **pick one client discipline.** Either everyone works
in Anchorpoint and locking is Anchorpoint's, or nobody does and locking is Git
LFS's. Half-and-half is the worst of both.

### 5.5 Leaving Git entirely

Worth naming, since studios do this and the locking story is stronger:

| Option | Locking | Free tier |
| --- | --- | --- |
| [Perforce Helix Core](https://www.perforce.com/products/helix-core/free-version-control) | Native exclusive checkout — the industry standard for binary assets | 5 users, 20 workspaces, no time limit; beyond that capped at 1,000 files |
| [Diversion](https://www.diversion.dev/pricing) | Native, proprietary VCS | Free first 5 users (up to 20), 100 GB, 5 repos; **free up to 10 users for academic institutions** |
| [Unity Version Control (Plastic)](https://support.unity.com/hc/en-us/articles/34748492914964-Understanding-Unity-DevOps-charges) | Native exclusive checkout | 3 seats, 25 GB storage, 100 GB egress; overage $0.14/GB-month |

All three trade away GitHub's collaboration surface and every student's
existing Git knowledge. Perforce's 5-user free tier is too small for a
ten-person crew; Diversion's education tier at 10 users and 100 GB is the only
one that actually fits, and it is a bet on a young vendor.

### 5.6 What the object storage would actually cost

Since several options above imply bringing your own bucket, the storage bill is
worth putting next to GitHub's, because it reframes the whole discussion.

| Provider | Storage | Egress |
| --- | --- | --- |
| [Cloudflare R2](https://developers.cloudflare.com/r2/pricing/) | $0.015/GB-month | **$0.00** |
| [Backblaze B2](https://www.backblaze.com/cloud-storage/pricing) | ~$0.00695/GB-month | Free up to 3× average monthly stored data, then $0.01/GB |
| [AWS S3 Standard](https://aws.amazon.com/s3/pricing/) | $0.023/GB-month (first 50 TB, us-east-1) | $0.09/GB after 100 GB/month free |

For a 50 GB repo with 500 GB of downloads over a four-month term: about **$2.40
on R2**, **$1.11 on B2**, and **$13.60 on S3** — rising to roughly $40 on S3 if
all ten clones land in the same month, which is exactly what happens at term
start. R2 and B2 are immune to that burst because egress is free or generous.

The point is not that a bucket is cheaper than GitHub. It is that **at this
scale, the bytes cost single-digit dollars anywhere**, GitHub included. Storage
economics are not the decision driver. Locking support and operational burden
are.

---

## 6. Does GitHub Education or Team change the picture?

**GitHub Education: no.** The
[GitHub Education for students](https://docs.github.com/en/education/about-github-education/github-education-for-students/about-github-education-for-students)
benefits are Copilot, 180 core-hours/month of Codespaces, GitHub Free with
unlimited private repos, the Student Developer Pack, and Classroom. Git LFS is
not mentioned anywhere on the page. The one storage figure it gives —
"equivalent to the amount included with GitHub Pro accounts" — is Codespaces
storage, not LFS. Since Pro and Free carry the same 10 GiB LFS allowance
anyway, student verification changes nothing here.

**GitHub Team: yes, materially — but only via an organization.** Team raises
both allowances from 10 GiB to 250 GiB at $4/user/month. The catch is that Team
is an organization plan; a repo owned by a personal account cannot be on it.
Moving Thunderbird into a GitHub organization is therefore the precondition for
any of this, and it is worth doing on its own merits (survives Braeden
graduating, gives the faculty advisor admin, lets seats be added and removed
per term).

The arithmetic once the repo is in an org:

- **Free org + metered overage.** 10 GiB free, then $0.07/$0.0875 per GiB. The
  "likely" scenario costs about $40/month.
- **Team, 10 seats.** $40/month in seat fees, which buys 250 GiB of each meter
  — enough to cover the lean and likely scenarios outright, with the same
  metered rates beyond that.

Those land in the same place for this workload, so pick on features rather than
price. Team's real value is the seat model and org controls, not the LFS
allowance. Note also that only Team raises the per-object cap from 2 GB to
4 GB, which matters if anyone commits a large cache or a long ProRes plate.

One more free option worth asking about before paying: UVU may already have
**GitHub Enterprise Cloud** through the campus program, in which case the
project could live in a university-owned org at 250 GiB with no departmental
spend. Worth one email to campus IT.

---

## 7. Facts the hosting decision needs

Stripped to what actually determines the choice:

1. **The quota is 10 GiB storage and 10 GiB bandwidth per month**, because the
   repo is owned by a personal account. GitHub Pro does not raise it. Only an
   organization on Team or Enterprise Cloud gets 250 GiB.
2. **The quota is shared across the owner's whole account and paid by the
   owner**, including bandwidth spent by strangers cloning this public repo.
3. **LFS storage is permanent.** Every push of a scene file adds a full copy,
   and removing files does not free the quota. The only documented remedy is
   deleting the repository.
4. **Running out is silent and looks like data loss.** With no payment method,
   clones return pointer files and Maya fails to open scenes; with a $0 budget,
   LFS is blocked for the rest of the month. Both recover on the next cycle.
5. **Today the repo is 38 MiB across 23 objects.** Ten GiB is roughly 569
   pushes of the existing 18 MB rig — about four scene saves per student per
   week for a term, before textures, caches, or audio.
6. **Overage is cheap: $0.07/GiB-month storage, $0.0875/GiB bandwidth.** The
   likely-case projection is about $40/month, roughly $160 for a term. Even the
   heavy case is about $100/month.
7. **GitHub does implement LFS locking** — verified live against this remote,
   which currently holds a lock on `tb_with_guns.ma`. Contrary claims online
   refer to other hosts.
8. **Locking follows `lfs.url`, not the Git remote.** Moving only the LFS
   objects to a cheap bucket-backed server breaks locking, because Rudolfs,
   Giftless, Estranged.Lfs, and lfs-s3 do not implement the endpoints.
9. **Hosts that do implement locking:** GitHub, GitLab (Free tier included),
   Gitea/Forgejo self-hosted, Bitbucket Server. **Bitbucket Cloud explicitly
   does not.** Azure DevOps is unconfirmed and should be assumed not to.
10. **GitLab for Education is free Ultimate with unlimited seats** and is the
    only alternative that is both free and locking-capable without running a
    server. GitLab Free's cap is 10 GiB per project covering repo plus LFS, and
    breaching it makes the project read-only.
11. **Anchorpoint's locking is not LFS locking** and does not interoperate with
    `git lfs lock`. Anchorpoint also does not host the objects; it is a client
    plus a metadata server on top of a remote you still have to provide.
12. **GitHub Education adds nothing here.** Its benefits are Copilot,
    Codespaces, private repos, and Classroom. LFS is not mentioned.
13. **Object storage is a rounding error at this scale** — $1 to $14 per term
    on R2, B2, or S3. Cost is not the reason to move.
14. **Hard limits to design around:** 2 GB maximum per LFS object on Free and
    Pro (4 GB on Team); 100 MiB for any non-LFS file; GitHub advises keeping
    repositories under 5 GB.

---

## 8. Recommendation

**Stay on GitHub. Move the repo into a GitHub organization, put a payment
method and a budget on it, and revisit at the end of the first term.**

The research started from an assumption worth discarding: that GitHub's LFS
quotas are the problem. They are not. The quota is real and the team will
exceed it — probably around week four or five — but the overage costs roughly
$12 to $100 per month at ten-student scale, and GitHub is one of only a handful
of hosts that implements the file-locking API the workflow is built on.
Migrating to save $40 a month, at the cost of locking or of a server nobody
owns after graduation, is a bad trade.

The three things to do, in order:

1. **Create an organization and transfer the repo.** This costs nothing and
   fixes the worst structural risk, which is not quota at all — it is that the
   project currently lives under one student's personal account. An org
   survives graduation, gives the faculty advisor admin, and is the
   precondition for Team's 250 GiB if you ever want it. Do this first.
2. **Attach a payment method and set an explicit budget** — start around $50
   per month. This converts the failure mode from "clones silently return
   pointer files mid-term" into "someone gets an email." A $0 budget still
   blocks LFS, so pick a real number. Watch the usage page monthly; the first
   month of real production tells you which projection scenario you are in.
3. **Ask campus IT whether UVU has GitHub Enterprise Cloud**, and in parallel
   [apply to GitLab for Education](https://about.gitlab.com/solutions/education/).
   Both are free and both raise the ceiling substantially. The GitLab
   application is the cheap hedge: it costs an afternoon, and if it lands you
   have a locking-capable free Ultimate instance to fall back on. Do not
   migrate on spec — hold it in reserve.

Alongside those, two habits that cost nothing and matter more than the hosting
choice, because LFS storage is unreclaimable:

- **Keep caches and playblasts out of the repo.** `.gitattributes` currently
  routes `.abc`, `.vdb`, `.exr`, `.mov`, and `.braw` into LFS. Simulation
  caches and dailies are regenerable and enormous; they belong in `.gitignore`
  next to `renders/`, not in permanent storage. This one change is the
  difference between the lean and heavy projections.
- **Push "done," not every save.** The 18 MB rig has two stored versions after
  eleven commits. That ratio, extrapolated, is the entire storage bill.

One thing to watch that has nothing to do with quotas: the repo is **public**,
so anyone can clone it and spend the owner's bandwidth, and the LFS locking
verification the README depends on is only as good as everyone using the same
client. If the team adopts Anchorpoint, adopt it for everyone and drop
`git lfs lock` from the README — the two locking systems do not compose, and a
half-adopted lock protocol is worse than none.

Revisit this note at the end of the first full production term, when there is
real usage data instead of a projection. The decision to move hosts should be
made against a bill, not a forecast.
