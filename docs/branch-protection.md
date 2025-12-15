# Branch-Specific Workflows & Protection

Complete guide to preventing PR merges when schema drift is detected, while allowing flexible code generation on different branches.

## Table of Contents

- [Overview](#overview)
- [Strategy 1: Branch-Conditional Enforcement](#strategy-1-branch-conditional-enforcement)
- [Strategy 2: GitHub Branch Protection Integration](#strategy-2-github-branch-protection-integration)
- [Strategy 3: Manual Approval for Schema Drift](#strategy-3-manual-approval-for-schema-drift)
- [Strategy 4: Emergency Override](#strategy-4-emergency-override)
- [Production Example](#production-example)
- [Troubleshooting](#troubleshooting)

---

## Overview

**Problem:** You want to prevent accidental PR merges when generated code is out of sync with schemas, but allow automatic code generation on main branch.

**Solution:** Use branch-conditional workflows that enforce strict validation on PRs while being lenient on protected branches.

**Key Concepts:**

- **Branch-Conditional Logic**: Different behavior for PRs vs main branch
- **GitHub Branch Protection**: Require status checks to pass before merging
- **Manual Approval**: Allow maintainers to override drift checks
- **Emergency Override**: Bypass protection for critical hotfixes

---

## Strategy 1: Branch-Conditional Enforcement

**Use Case:** Strict validation on PRs, lenient on main branch

### Basic Pattern

```yaml
name: Branch-Aware Schema Validation

on:
  pull_request:
  push:
    branches: [main, dev]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run LUMOS validation
        id: lumos
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          # Fail on drift for PRs, warn for main
          fail-on-drift: ${{ github.event_name == 'pull_request' }}
          comment-on-pr: ${{ github.event_name == 'pull_request' }}

      - name: Block PR merge if drift detected
        if: |
          github.event_name == 'pull_request' &&
          steps.lumos.outputs.drift-detected == 'true'
        run: |
          echo "::error::Schema drift detected. Regenerate code before merging."
          exit 1
```

**Behavior:**
- **Pull Requests:** Fails CI if drift detected → blocks merge
- **Main Branch:** Warns but doesn't fail → allows push
- **Dev Branch:** Same lenient behavior as main

### Advanced: Branch-Specific Messages

```yaml
- name: Branch-specific feedback
  if: steps.lumos.outputs.drift-detected == 'true'
  run: |
    if [ "${{ github.event_name }}" == "pull_request" ]; then
      echo "::error::❌ PR blocked: Schema drift detected"
      echo "Run: lumos generate schemas/**/*.lumos"
      exit 1
    else
      echo "::warning::⚠️ Schema drift detected on ${{ github.ref_name }}"
      echo "Consider running: lumos generate schemas/**/*.lumos"
    fi
```

### Multi-Branch Protection

```yaml
on:
  pull_request:
    branches: [main, staging, production]
  push:
    branches: [main, staging, production]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Determine strictness
        id: config
        run: |
          # Strict for PRs to production/main
          if [[ "${{ github.event_name }}" == "pull_request" ]]; then
            if [[ "${{ github.base_ref }}" == "production" ]] || [[ "${{ github.base_ref }}" == "main" ]]; then
              echo "fail-on-drift=true" >> $GITHUB_OUTPUT
              echo "strictness=STRICT" >> $GITHUB_OUTPUT
            else
              echo "fail-on-drift=false" >> $GITHUB_OUTPUT
              echo "strictness=LENIENT" >> $GITHUB_OUTPUT
            fi
          else
            echo "fail-on-drift=false" >> $GITHUB_OUTPUT
            echo "strictness=LENIENT" >> $GITHUB_OUTPUT
          fi

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          fail-on-drift: ${{ steps.config.outputs.fail-on-drift }}
          comment-on-pr: true

      - name: Log policy
        run: |
          echo "Branch: ${{ github.ref_name }}"
          echo "Event: ${{ github.event_name }}"
          echo "Policy: ${{ steps.config.outputs.strictness }}"
```

---

## Strategy 2: GitHub Branch Protection Integration

**Use Case:** Enforce validation at the GitHub level, preventing merge button from working

### Setup: Require Status Checks

#### Step 1: Navigate to Repository Settings

1. Go to **Settings** → **Branches**
2. Click **Add branch protection rule**

#### Step 2: Configure Protection Rule

**Branch name pattern:** `main`

**Required settings:**

- ✅ **Require a pull request before merging**
  - Require approvals: 1+ (recommended)
  - Dismiss stale PR approvals when new commits are pushed

- ✅ **Require status checks to pass before merging**
  - ✅ Require branches to be up to date before merging
  - **Status checks that are required:**
    - Select: `validate` (the workflow job name)
    - Or: `Schema Validation` (the workflow name)

- ✅ **Require conversation resolution before merging** (optional but recommended)

- ✅ **Do not allow bypassing the above settings** (for strict enforcement)

#### Step 3: Create Workflow with Required Status Check

```yaml
name: validate  # This name becomes the status check

on:
  pull_request:
    branches: [main]

jobs:
  schema-validation:  # Or use this as the status check name
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          fail-on-drift: true  # Fails the status check
          comment-on-pr: true
```

**Result:** GitHub UI shows:

```
❌ validate — Required
   Merging is blocked

⚠️ This branch has not been approved
   At least 1 approving review is required
```

### Testing Branch Protection

```bash
# 1. Create test PR with drift
git checkout -b test-drift
echo "// Modified schema" >> schemas/test.lumos
git add schemas/test.lumos
git commit -m "test: Trigger drift"
git push origin test-drift

# 2. Create PR via GitHub CLI
gh pr create \
  --title "Test: Branch Protection" \
  --body "Testing schema drift detection"

# 3. Check PR status
gh pr view --json statusCheckRollup

# Expected output:
# ❌ validate — Required
# ⚠️ Merging is blocked
```

### Bypass for Administrators (Optional)

If you want to allow repository admins to bypass branch protection:

**Settings → Branches → Edit rule:**
- ❌ **Do not allow bypassing the above settings** (leave unchecked)
- Admins will see: "Merge without waiting for requirements to be met (bypass branch protections)"

### CODEOWNERS Integration

Combine with CODEOWNERS for schema file approval:

**Create `.github/CODEOWNERS`:**
```
# Schema files require approval from schema team
schemas/**/*.lumos @org/schema-maintainers
docs/           @org/documentation-team
```

**Branch protection rule:**
- ✅ **Require review from Code Owners**

**Result:** PRs touching `.lumos` files require approval from `@org/schema-maintainers` team.

---

## Strategy 3: Manual Approval for Schema Drift

**Use Case:** Allow maintainers to approve PRs with drift after review

### Workflow with Approval Override

```yaml
name: Schema Drift with Manual Approval

on:
  pull_request:

jobs:
  detect-drift:
    name: Detect Schema Drift
    runs-on: ubuntu-latest
    outputs:
      drift-detected: ${{ steps.lumos.outputs.drift-detected }}
      diff-summary: ${{ steps.lumos.outputs.diff-summary }}
    steps:
      - uses: actions/checkout@v4

      - name: Validate schemas
        id: lumos
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          fail-on-drift: false  # Don't fail yet - check approval first
          comment-on-pr: false  # Use custom comment below

      - name: Post drift warning with approval instructions
        if: steps.lumos.outputs.drift-detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const diffSummary = `${{ steps.lumos.outputs.diff-summary }}`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ⚠️ Schema Drift Detected

Generated code is out of sync with schemas.

**Options to Resolve:**

1. ✅ **Regenerate code locally (Recommended):**
   \`\`\`bash
   lumos generate schemas/**/*.lumos
   git add .
   git commit -m "chore: Regenerate code from schemas"
   git push
   \`\`\`

2. 🔐 **Request manual approval** from @org/schema-maintainers team:
   - Add label: \`schema-drift-approved\` (maintainers only)
   - Requires maintainer review + approval
   - Use for intentional drift (e.g., incremental migration)

**Affected Files:**
\`\`\`diff
${diffSummary}
\`\`\`

---

<details>
<summary>💡 Why is this blocked?</summary>

Schema drift means the generated code doesn't match the schemas. This can cause:
- Runtime errors (type mismatches)
- Serialization bugs (Borsh incompatibility)
- Confusion for other developers

**Always regenerate code after schema changes.**
</details>

<sub>💡 Tip: Run \`lumos diff\` locally to preview changes before committing</sub>
              `
            });

  require-approval:
    name: Check Approval Status
    needs: detect-drift
    if: needs.detect-drift.outputs.drift-detected == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Check for manual approval
        uses: actions/github-script@v7
        with:
          script: |
            // Fetch PR reviews
            const { data: reviews } = await github.rest.pulls.listReviews({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });

            // Fetch PR labels
            const { data: labels } = await github.rest.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            // Define maintainers who can approve drift
            const maintainers = ['alice', 'bob', 'charlie']; // Update with your GitHub usernames

            // Check for maintainer approval
            const hasApproval = reviews.some(r =>
              r.state === 'APPROVED' && maintainers.includes(r.user.login)
            );

            // OR check for manual approval label
            const hasLabel = labels.some(l => l.name === 'schema-drift-approved');

            if (!hasApproval && !hasLabel) {
              core.setFailed('❌ Schema drift requires maintainer approval or label "schema-drift-approved"');
              core.notice('Add label "schema-drift-approved" or get approval from: ' + maintainers.join(', '));
            } else {
              core.info('✅ Manual approval granted - allowing merge with drift');

              // Log who approved
              if (hasApproval) {
                const approver = reviews.find(r => r.state === 'APPROVED' && maintainers.includes(r.user.login));
                core.info(`Approved by: ${approver.user.login}`);
              }
              if (hasLabel) {
                core.info('Label "schema-drift-approved" present');
              }
            }
```

### Team-Based Approval (Using GitHub Teams)

```yaml
- name: Check team approval
  uses: actions/github-script@v7
  with:
    script: |
      // Fetch reviews
      const { data: reviews } = await github.rest.pulls.listReviews({
        owner: context.repo.owner,
        repo: context.repo.repo,
        pull_number: context.issue.number,
      });

      // Fetch team members
      const { data: teamMembers } = await github.rest.teams.listMembersInOrg({
        org: context.repo.owner,
        team_slug: 'schema-maintainers',
      });

      const teamUsernames = teamMembers.map(m => m.login);

      // Check if any team member approved
      const teamApproved = reviews.some(r =>
        r.state === 'APPROVED' && teamUsernames.includes(r.user.login)
      );

      if (!teamApproved) {
        core.setFailed('❌ Requires approval from @org/schema-maintainers team');
        core.notice('Request review from the schema-maintainers team');
      } else {
        const approver = reviews.find(r => r.state === 'APPROVED' && teamUsernames.includes(r.user.login));
        core.info(`✅ Team approval granted by: ${approver.user.login}`);
      }
```

### Label-Based Approval (Simplest)

```yaml
- name: Check for approval label
  uses: actions/github-script@v7
  with:
    script: |
      const { data: labels } = await github.rest.issues.listLabelsOnIssue({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
      });

      const hasApproval = labels.some(l => l.name === 'schema-drift-approved');

      if (!hasApproval) {
        core.setFailed('❌ Add label "schema-drift-approved" to proceed with drift');
      } else {
        core.info('✅ Approval label present - allowing merge');
      }
```

**Create the label in your repository:**

```bash
gh label create "schema-drift-approved" \
  --description "Maintainer approval for intentional schema drift" \
  --color "0E8A16"
```

---

## Strategy 4: Emergency Override

**Use Case:** Allow critical hotfixes to bypass schema validation

### Hotfix Label Bypass

```yaml
name: Schema Validation with Hotfix Override

on:
  pull_request:
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check for hotfix label
        id: hotfix
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          result-encoding: string
          script: |
            const { data: labels } = await github.rest.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const isHotfix = labels.some(l => l.name === 'hotfix');

            if (isHotfix) {
              core.warning('🚨 Hotfix label detected - bypassing schema drift check');
            }

            return isHotfix ? 'true' : 'false';

      - name: Run LUMOS validation
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          # Skip drift check for hotfixes
          fail-on-drift: ${{ steps.hotfix.outputs.result != 'true' }}
          comment-on-pr: true

      - name: Log hotfix bypass
        if: steps.hotfix.outputs.result == 'true'
        run: |
          echo "::warning::🚨 HOTFIX MODE: Schema drift check bypassed"
          echo "::warning::Remember to fix schema drift in follow-up PR"
```

**Create hotfix label:**

```bash
gh label create "hotfix" \
  --description "Emergency fix - bypasses schema validation" \
  --color "D93F0B"
```

### Branch Name Pattern Override

```yaml
- name: Check if hotfix branch
  id: branch-check
  run: |
    branch_name="${{ github.head_ref || github.ref_name }}"

    if [[ "$branch_name" =~ ^hotfix/ ]]; then
      echo "is-hotfix=true" >> $GITHUB_OUTPUT
      echo "::warning::Hotfix branch detected: $branch_name"
    else
      echo "is-hotfix=false" >> $GITHUB_OUTPUT
    fi

- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'
    fail-on-drift: ${{ steps.branch-check.outputs.is-hotfix != 'true' }}
```

**Branch naming convention:**
- `hotfix/fix-critical-bug` → Bypasses validation
- `feature/add-schema` → Normal validation
- `bugfix/minor-issue` → Normal validation

### Time-Based Emergency Mode

```yaml
- name: Check emergency mode
  id: emergency
  run: |
    # Check if emergency mode file exists (created manually)
    if [ -f ".github/EMERGENCY_MODE" ]; then
      echo "enabled=true" >> $GITHUB_OUTPUT
      echo "::warning::🚨 EMERGENCY MODE ACTIVE - Schema validation relaxed"

      # Check when it was enabled
      created=$(stat -f %B .github/EMERGENCY_MODE)
      now=$(date +%s)
      hours=$(( ($now - $created) / 3600 ))

      echo "::warning::Emergency mode has been active for $hours hours"

      if [ $hours -gt 24 ]; then
        echo "::error::Emergency mode active for >24h - please disable"
      fi
    else
      echo "enabled=false" >> $GITHUB_OUTPUT
    fi

- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'
    fail-on-drift: ${{ steps.emergency.outputs.enabled != 'true' }}
```

**Enable emergency mode:**

```bash
# Enable (requires push access to main)
touch .github/EMERGENCY_MODE
git add .github/EMERGENCY_MODE
git commit -m "EMERGENCY: Disable schema validation temporarily"
git push origin main

# Disable
git rm .github/EMERGENCY_MODE
git commit -m "chore: Re-enable schema validation"
git push origin main
```

---

## Production Example

Complete production-ready setup combining all strategies:

```yaml
name: Production Schema Protection

on:
  pull_request:
    branches: [main, staging]
  push:
    branches: [main]

jobs:
  # Job 1: Validate PRs (strict enforcement)
  validate-pr:
    name: Validate PR Schema Changes
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Check for emergency override
      - name: Check for hotfix label
        id: hotfix
        uses: actions/github-script@v7
        with:
          result-encoding: string
          script: |
            const { data: labels } = await github.rest.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            return labels.some(l => l.name === 'hotfix') ? 'true' : 'false';

      # Run validation
      - name: Validate schemas
        id: lumos
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          fail-on-drift: ${{ steps.hotfix.outputs.result != 'true' }}
          comment-on-pr: false  # Custom comment below

      # Post detailed comment
      - name: Post validation results
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const driftDetected = '${{ steps.lumos.outputs.drift-detected }}' === 'true';
            const isHotfix = '${{ steps.hotfix.outputs.result }}' === 'true';
            const diffSummary = `${{ steps.lumos.outputs.diff-summary }}`;

            let body = '';

            if (driftDetected && isHotfix) {
              body = `## 🚨 HOTFIX MODE: Schema Drift Bypassed

**WARNING:** This PR has schema drift but is marked as a hotfix.

**Affected Files:**
\`\`\`diff
${diffSummary}
\`\`\`

**⚠️ Required Follow-up:**
- [ ] Create follow-up PR to fix schema drift
- [ ] Link follow-up PR here
- [ ] Document why hotfix was necessary

**Reviewers:** Please verify this truly requires hotfix treatment.
              `;
            } else if (driftDetected) {
              body = `## ❌ Schema Drift Detected - PR Blocked

Generated code is out of sync with schemas.

**To Fix:**
\`\`\`bash
lumos generate schemas/**/*.lumos
git add .
git commit -m "chore: Regenerate code from schemas"
git push
\`\`\`

**Affected Files:**
\`\`\`diff
${diffSummary}
\`\`\`

**Alternative:** If drift is intentional:
1. Get approval from @org/schema-maintainers
2. Add label \`schema-drift-approved\`

**Emergency Override:** Add \`hotfix\` label (use sparingly!)
              `;
            } else {
              body = `## ✅ Schema Validation Passed

All schemas are valid and generated code is up to date.

**Validated:**
- ${{ steps.lumos.outputs.schemas-validated }} schema(s) validated
- ${{ steps.lumos.outputs.schemas-generated }} file(s) generated
- No drift detected

Safe to merge! 🚀
              `;
            }

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

      # Check for manual approval if drift detected
      - name: Check manual approval
        if: |
          steps.lumos.outputs.drift-detected == 'true' &&
          steps.hotfix.outputs.result != 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const { data: labels } = await github.rest.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const { data: reviews } = await github.rest.pulls.listReviews({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });

            const maintainers = ['alice', 'bob'];  // Update with your team

            const hasLabel = labels.some(l => l.name === 'schema-drift-approved');
            const hasApproval = reviews.some(r =>
              r.state === 'APPROVED' && maintainers.includes(r.user.login)
            );

            if (!hasLabel && !hasApproval) {
              core.setFailed('❌ Schema drift detected. Fix code or get maintainer approval.');
            } else {
              core.info('✅ Manual approval granted');
            }

  # Job 2: Generate on main (lenient, auto-commit)
  generate-main:
    name: Generate Code on Main
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          fail-on-drift: false  # Warn only, don't fail

      - name: Commit generated code
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A

          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "chore: Auto-generate code from schemas [skip ci]"
            git push
          fi
```

**Setup checklist:**

1. ✅ Update `maintainers` array with actual GitHub usernames
2. ✅ Create labels: `hotfix`, `schema-drift-approved`
3. ✅ Configure branch protection on `main`:
   - Require status check: `validate-pr`
   - Require 1+ approvals
4. ✅ Add to `.github/CODEOWNERS`:
   ```
   schemas/**/*.lumos @org/schema-maintainers
   ```
5. ✅ Test with a PR that has drift

---

## Troubleshooting

### Problem: Workflow doesn't block PR merge

**Check:**

1. **Status check configured correctly?**
   ```bash
   gh api repos/:owner/:repo/branches/main/protection | jq '.required_status_checks'
   ```

   Should show:
   ```json
   {
     "strict": true,
     "contexts": ["validate-pr"]
   }
   ```

2. **Job name matches?**
   - Branch protection requires: `validate-pr`
   - Workflow job name: `validate-pr` ✅

   If they don't match, update one to match the other.

3. **Workflow triggered?**
   ```bash
   gh run list --workflow="production-schema-protection.yml"
   ```

   If not triggered, check:
   - Workflow file in `.github/workflows/`?
   - Syntax errors in YAML?
   - `on:` triggers include `pull_request`?

**Fix:** Update branch protection to match exact job name:

```bash
gh api repos/:owner/:repo/branches/main/protection/required_status_checks \
  -X PATCH \
  -f strict=true \
  -f contexts[]="validate-pr"
```

### Problem: Manual approval not working

**Check:**

1. **Label exists?**
   ```bash
   gh label list | grep schema-drift-approved
   ```

   If not found:
   ```bash
   gh label create "schema-drift-approved" --color "0E8A16"
   ```

2. **Maintainer list correct?**
   - Check workflow file: `const maintainers = ['alice', 'bob'];`
   - Use actual GitHub usernames (case-sensitive)
   - Verify usernames:
     ```bash
     gh api users/alice | jq '.login'
     ```

3. **Review state correct?**
   ```bash
   gh pr view 123 --json reviews
   ```

   Should show:
   ```json
   {
     "state": "APPROVED",
     "user": {"login": "alice"}
   }
   ```

**Fix:** Update maintainers list or use team-based approval:

```yaml
# Instead of hardcoded list, use team
const { data: teamMembers } = await github.rest.teams.listMembersInOrg({
  org: context.repo.owner,
  team_slug: 'schema-maintainers',
});
```

### Problem: Hotfix bypass not working

**Check:**

1. **Label applied?**
   ```bash
   gh pr view 123 --json labels
   ```

2. **Script output correct?**
   - Check workflow run logs
   - Look for: `steps.hotfix.outputs.result`
   - Should be string `'true'` not boolean

**Common mistake:**

```yaml
# ❌ Wrong
result-encoding: json  # Returns boolean

# ✅ Correct
result-encoding: string  # Returns 'true' or 'false'
```

### Problem: Emergency mode file not working

**Check:**

1. **File committed to main?**
   ```bash
   git ls-files .github/EMERGENCY_MODE
   ```

2. **File checked out in workflow?**
   - Ensure `uses: actions/checkout@v4` runs before check
   - Verify with:
     ```yaml
     - run: ls -la .github/
     ```

3. **Permissions correct?**
   - Workflow needs: `contents: read`

### Problem: Auto-commit on main not working

**Check:**

1. **Permissions granted?**
   ```yaml
   permissions:
     contents: write  # Required for git push
   ```

2. **Bot identity configured?**
   ```bash
   git config user.name "github-actions[bot]"
   git config user.email "github-actions[bot]@users.noreply.github.com"
   ```

3. **Branch protection allows bot commits?**
   - Settings → Branches → Edit rule
   - ✅ **Allow specified actors to bypass required pull requests**
   - Add: `github-actions[bot]`

4. **`[skip ci]` preventing infinite loops?**
   - Commit message must include: `[skip ci]` or `[ci skip]`
   - Without this, commit triggers new workflow run

### Problem: Different behavior in fork PRs

**Issue:** Forks have restricted `GITHUB_TOKEN` permissions

**Solution:** Use `pull_request_target` with caution:

```yaml
on:
  pull_request_target:  # Runs in base repo context
    types: [opened, synchronize]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      # IMPORTANT: Checkout PR head, not base
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
```

**⚠️ Security Warning:** `pull_request_target` has write access. Only use for trusted operations.

### Problem: Status check always pending

**Check:**

1. **Job completed?**
   ```bash
   gh run view --log
   ```

2. **Job name in conclusion?**
   - Each job must complete (success/failure)
   - Skipped jobs don't satisfy status checks

**Fix:** Ensure job always runs:

```yaml
jobs:
  validate-pr:
    if: github.event_name == 'pull_request'  # Don't skip
    runs-on: ubuntu-latest
    steps:
      # ...
```

### Getting Help

**Debug workflow runs:**

```bash
# View recent runs
gh run list --workflow="schema-protection.yml" --limit 5

# View specific run
gh run view 1234567890 --log

# Re-run failed run
gh run rerun 1234567890
```

**Enable debug logging:**

```yaml
env:
  ACTIONS_STEP_DEBUG: true
  ACTIONS_RUNNER_DEBUG: true
```

**Test locally with act:**

```bash
# Install act
brew install act

# Test pull_request event
act pull_request \
  -W .github/workflows/schema-protection.yml \
  -e .github/workflows/test-payload.json
```

**Example test payload** (`.github/workflows/test-payload.json`):

```json
{
  "pull_request": {
    "number": 1,
    "base": {"ref": "main"},
    "head": {"ref": "feature/test"}
  }
}
```

---

**Need more help?** Open an issue: https://github.com/getlumos/lumos-action/issues
