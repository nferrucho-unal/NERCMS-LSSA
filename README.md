# NERCMS-LSSA
National Emergency Response and Crisis Management System
Large-Scale Software Architecture
Group 2 Project

## Repository Structure

```text
/
|-- .github/
|   `-- CODEOWNERS
|-- metamodel.tx
|-- team-2a/
|-- team-2b/
|-- team-2c/
|-- team-2d/
`-- team-2e/
```

Each `team-*` folder is owned by one team leader. The root `metamodel.tx` is shared and impacts all teams.

## Branching Strategy

- Protected branch: `main`
- Working branches: `feature/<team>-<short-description>` or `fix/<team>-<short-description>`
- No direct pushes to `main`
- All code changes should be integrated using Pull Requests (PRs)

Examples:

- `feature/2a-user-auth`
- `fix/2c-validation-bug`

## Pull Request Workflow

1. Create a branch from `main`.
2. Commit changes in your team scope.
3. Open a PR to `main`.
4. Request review from required code owners automatically assigned by `CODEOWNERS`.
5. Merge only after required approvals and passing checks.

## Responsibilities

### Developers

- Always work through Pull Requests.
- Do not push directly to `main`.
- Keep changes scoped to your team folder whenever possible.

### Team Leaders

- Review and approve PRs that affect their owned paths.
- Coordinate cross-team changes.
- Perform merges once approval requirements are met.

## Cross-Team and Shared Changes

- If a PR changes multiple team folders, multiple leader approvals are expected (one per affected owner path).
- If a PR changes `metamodel.tx`, reviews from all leaders are expected due to shared impact.

Note: GitHub enforces required approvals and code-owner review rules. Requiring specific combinations (for example, all 5 leaders) depends on repository settings and team discipline.

## Bypass Policy (Team Discipline)

Leaders may bypass PR requirements only when all these are true:

- The change is isolated to their own team folder.
- The change does not modify `metamodel.tx`.
- The change does not affect any other team folder.

This bypass policy is not fully enforceable automatically with standard GitHub settings in a personal repository. It must be followed by team agreement.