# github-actions

Flipstone github actions library.

## Actions

- `find-stack-ghc-yamls` — finds the `stack*.yaml` files in a repository, for
  building a multi-GHC test matrix.
- `setup-dockerized-stack` — sets up a dockerized stack environment.

## Releases

Releases are minted automatically. A push to `main` that changes a composite
action publishes a tag; a push that only touches the README, this workflow, or
anything else outside an action directory publishes nothing, so downstream
repositories don't get bump PRs for changes that cannot affect them.

Tags look like:

```
v2026.8.28.4
 │    │ │  └── the workflow run number
 │    │ └───── day of the commit
 │    └─────── month of the commit
 └──────────── year of the commit
```

Three details of that format are load-bearing:

- **The year leads.** Dependabot stops comparing a bare counter correctly once
  it passes 1000 (dependabot-core#11198). A 4-digit year in front avoids that
  permanently.
- **The date is the commit's, not the day the workflow ran.** Re-running an old
  workflow therefore recreates the identical tag rather than minting a new one
  that sorts above newer releases while pointing at older code. The release step
  is a no-op if the tag already exists.
- **Month and day are unpadded** — `2026.8.6`, not `2026.08.06` — because
  Dependabot compares version segments numerically.

## For Flipstone developers

### Consuming these actions

Pin **both** the version and the commit sha. `uses:` takes only one ref
(`owner/repo/path@ref`), so there is no `@version@sha` form — the sha goes in the
ref and the version goes in a trailing comment:

```yaml
      - uses: flipstone/github-actions/setup-dockerized-stack@2c1f2781f8a1b92a0ad82223adf77bbf24b262c2 # v2026.6.16.1
```

Both halves carry weight, and the comment is not decoration:

- **The sha is what runs.** A tag can be moved to point at different code; a sha
  cannot. That is what makes the pin trustworthy rather than merely tidy.
- **The version is what makes it legible.** Nobody can tell how stale
  `2c1f2781…` is by reading it. `# v2026.6.16.1` says June 2026 at a glance.
- **Dependabot maintains the two together**, rewriting the sha and the comment in
  the same edit. A pin with no comment still gets its sha bumped, but you lose
  the only human-readable signal of how far behind it had drifted.

Use the full 40-character sha; that is what GitHub recommends for pinning and
what Dependabot writes.

Then add a `github-actions` entry to the repository's `.github/dependabot.yaml`
so new releases arrive as bump PRs:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    # One PR for all action bumps rather than one PR per action.
    groups:
      github-actions:
        patterns:
          - "*"
```

Note that this only works because releases exist. Before they did, a bare sha
pin gave Dependabot nothing to compare against, so pins never moved and drifted
apart silently — no error, no PR, and nothing to indicate anything was stale.

### This repository tracks itself

`.github/dependabot.yaml` watches the actions used here — both in the release
workflow and inside the composite actions themselves. The ecosystem needs a
directory entry per composite action, since `/` only covers `.github/workflows`.

### Changing an action

Composite actions have no build step and no tests here, so the change *is* the
release. Two consequences worth keeping in mind:

- A merge to `main` that touches an action directory immediately becomes a
  release, and Dependabot will start proposing it to every consuming repository.
  There is no staging step, so review accordingly.
- To try a change downstream before merging, pin your branch's sha in the
  downstream repository on a branch of its own and iterate. Once your change
  merges here, Dependabot opens the bump PR — discard the test pin rather than
  merging it.
