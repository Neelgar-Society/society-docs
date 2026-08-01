# Release Process

## Branch & Tag Flow

```
develop → release/rel_YYYYMMDD → main → tag rel-YYYY.MM.DD
main    → hotfix/hf_YYYYMMDD    → main + develop → tag hf-YYYY.MM.DD
```

Rule: **maximum one release per day.** Full branch/tag naming rules are
in `Branching.md`.

## Pre-Release Checklist (mandatory)

Must be completed in full **before** merging a release branch into
`main` and creating a release tag.

### Code & Branch Hygiene
- [ ] Release branch created from latest `develop`
- [ ] Branch name follows `release/rel_YYYYMMDD`
- [ ] No feature commits added after release cut
- [ ] All feature branches merged and deleted

### Testing & Quality
- [ ] All CI pipelines are green
- [ ] Unit tests executed successfully
- [ ] Integration tests executed (if applicable)
- [ ] No critical or high-severity bugs open
- [ ] Static code analysis passed

### Documentation & Configuration
- [ ] Configuration changes reviewed
- [ ] Environment variables verified
- [ ] API / OpenAPI / README updated (if needed)
- [ ] Migration scripts reviewed (if any)

### Approval & Governance
- [ ] Code review completed by required reviewers
- [ ] QA/UAT sign-off received
- [ ] Product Owner approval received
- [ ] Release date confirmed (one release per day rule)

### Release Execution
- [ ] Release branch merged into `main`
- [ ] Release tag created: `rel-YYYY.MM.DD`
- [ ] Release branch deleted after merge
- [ ] Release notes shared with stakeholders

## Hotfix Process

Urgent production fixes skip the release-branch flow entirely:

1. Branch `hotfix/hf_YYYYMMDD` from `main`.
2. Fix, test, get required review.
3. Merge into `main` **and** `develop`.
4. Tag `main` with `hf-YYYY.MM.DD`.
5. Deploy (see `Deployment.md` — backend/frontend deploys are automatic
   on push to `main`, with automatic rollback on health-check failure).

## Deploy mechanics

Once a release or hotfix merges into `main`, deployment to `server-H81`
is automatic via GitHub Actions (backend JAR swap + systemd restart,
frontend static build swap) — see `Deployment.md` for the full
step-by-step and rollback behavior. This checklist governs the
**human approval gate before that merge**, not the deploy itself.

## Roles

| Role | Responsibility |
|---|---|
| Dev | Feature branches, PRs, fixing review comments |
| Tech Lead / Release Manager | Creates release & hotfix branches, creates tags, owns this checklist |
| QA | UAT sign-off |
| Product Owner | Final release approval |
