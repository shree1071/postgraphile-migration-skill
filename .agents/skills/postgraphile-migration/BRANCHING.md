# Branch Naming and PR Routing Strategy

Reference rules for sub-branching and pull request targets in Zarthi modernization repositories.

---

## 1. Sub-Branch Naming Rule
In this repository, sub-branches (child branches) must follow the double-dash convention:

```text
<parent-feature-branch>--<sub-scope>
```

### Examples:
- Parent Feature: `modernization/task-189/db-migration-objectives-requestmoduleactionableservice`
- Valid Sub-Branch: `modernization/task-189/db-migration-objectives-requestmoduleactionableservice--shreeharsha-phase-2`
- Invalid Sub-Branch: `modernization/task-189/db-migration-objectives-requestmoduleactionableservice-phase-2--shreeharsha`

---

## 2. CI Verification Logic (pr-branch-strategy.sh)
The repository executes pre-merge validation (`pre-merge-checks/pr-branch-strategy.sh`).
Under **CHECK 5: Target branch routing**:

```bash
PARENT_FEATURE=$(echo "$SOURCE" | sed 's/--[^-].*$//')
if [ "$TARGET" != "$PARENT_FEATURE" ]; then
  fail_pr "Sub-branch '$SOURCE' must target its parent '$PARENT_FEATURE', not '$TARGET'."
fi
```

The script extracts the parent branch by stripping `--` and everything after it.
If any extra scope suffix is placed before `--`, the parent check will fail.

---

## 3. Pull Request Creation Checklist
1. Verify the current branch name:
   `git branch --show-current`
2. Confirm the parent prefix before `--` matches the intended base branch exactly.
3. Push the sub-branch to origin:
   `git push -u origin <sub-branch-name>`
4. Open the pull request targeting `<parent-feature-branch>`:
   `gh pr create --base <parent-feature-branch> --head <sub-branch-name> ...`
5. Verify CI checks:
   `gh pr checks <pr-number>`
   Confirm `PR Validation - Branch Strategy Enforcement` reports `pass`.
