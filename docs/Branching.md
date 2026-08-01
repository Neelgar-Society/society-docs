# Branching & Tagging

Enterprise git standard for all Neelgar Society repositories. Full detail
lives here; `CONTRIBUTING.md` (org `.github` repo) has a condensed
version for quick reference in PRs.

## Branch types

| Branch | Source | Target | Naming | Created by |
|---|---|---|---|---|
| Feature | `develop` | `develop` | `feature/<JIRA-ID>-<short-desc>` | any dev |
| Release | `develop` | `main` | `release/rel_YYYYMMDD` | Tech Lead / Release Manager |
| Hotfix | `main` | `main` + `develop` | `hotfix/hf_YYYYMMDD` | Tech Lead / Release Manager |

There is no `bugfix/*` branch type — routine bug fixes go through
`feature/*` with a Jira ID; urgent production fixes go through
`hotfix/*`.

### Feature branches
- Merge strategy: **Squash or Rebase** only, never a plain merge commit.
- Examples: `feature/ABC-123-user-creation`, `feature/ABC-456-add-payment-validation`.

### Release branches
- **Maximum one release per day.**
- Allowed changes once cut: bug fixes, configuration changes,
  documentation updates, version/metadata updates. **No new features.**
- Deleted after merging into `main`.
- Examples: `release/rel_20260117`, `release/rel_20260205`.

### Hotfix branches
- Sourced from `main`, merged back into **both** `main` and `develop`.
- Each hotfix results in a new hotfix tag.
- Examples: `hotfix/hf_20260118`, `hotfix/hf_20260210`.

## Tagging

- Tags are only created on `main`, only by Release Managers, and are
  **never moved or deleted**.
- Release tag: `rel-YYYY.MM.DD` — created after a release branch merges
  into `main`. Examples: `rel-2026.01.17`, `rel-2026.02.05`.
- Hotfix tag: `hf-YYYY.MM.DD` — created after a hotfix merges into
  `main`. Examples: `hf-2026.01.18`, `hf-2026.02.10`.

## Commit messages

Format: `[JIRA-ID] type: <subject>`

| Type | Meaning |
|---|---|
| `feat` | new feature |
| `fix` | bug fix |
| `chore` | non-src/test changes (e.g. dependency bumps) |
| `refactor` | code change that neither fixes a bug nor adds a feature |
| `docs` | documentation (OpenAPI, README, markdown) |
| `style` | formatting only, no logic change |
| `test` | new or corrected tests |
| `perf` | performance improvement |
| `ci` | CI/CD changes |
| `build` | build system or external dependency changes |
| `revert` | reverts a previous commit |

Examples:
```
[ABC-123] feat: implemented user creation api
[ABC-456] fix: fixing business logic for user creation
[ABC-457] test: added unit test cases for user service class
```

## Enforcement (GitHub Rulesets)

- `main` and `develop` are protected: PR required, mandatory review,
  passing CI status checks, no direct pushes.
- Creation of `release/*` and `hotfix/*` branches restricted to a
  release-managers team.
- Tag creation restricted to `main` only, and to the release-managers team.

## Do

- Cut release branches from `develop`.
- Use date-based naming consistently.
- Create tags only after approval.
- Merge hotfixes back into `develop`.

## Don't

- Create multiple release branches for the same date.
- Tag `develop` or `feature` branches.
- Merge features directly into `main`.
- Bypass CI or review requirements.
