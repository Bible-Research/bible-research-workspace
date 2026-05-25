---
description: Create a GitHub pull request without shell quoting issues
---

Always follow these steps when creating a pull request. Never pass PR body text inline via `--body` (causes shell quoting failures). Always use `--body-file` instead.

1. Write the PR description to a temporary markdown file at the repo root (e.g. `pr-body.md`).

2. Run the `gh pr create` command using `--body-file`:
```
gh pr create --title "<title>" --base <base-branch> --head <feature-branch> --body-file pr-body.md
```

3. After the PR is created successfully, delete the temporary file:
```
rm pr-body.md
```

This pattern avoids all shell quoting/escaping issues with multi-line or special-character PR bodies.
