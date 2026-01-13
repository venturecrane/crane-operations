# New Venture Playbook

Step-by-step guide for adding a new venture to the Crane operations system.

## Overview

The Crane system supports multiple ventures with:
- Shared infrastructure (Crane Relay, Crane Command Center)
- Isolated operations repos per venture
- Standardized label system across all ventures
- Multi-venture visibility in Command Center

**Existing ventures:**
- Venture Crane (`venturecrane/crane-operations`)
- Silicon Crane (`siliconcrane/sc-operations`)
- Durgan Field Guide (`durganfieldguide/dfg-console`)

## Prerequisites

Before starting, ensure you have:
- [ ] GitHub organization created for new venture
- [ ] Owner/admin access to the organization
- [ ] GitHub personal access token with `repo` scope
- [ ] Access to Crane Relay worker secrets
- [ ] Access to Crane Command Center Vercel project

## Step 1: Create Operations Repository

Create the GitHub repository for issue tracking.

### 1.1 Create Repository

```bash
# Using gh CLI
gh repo create <org>/<venture>-operations \
  --public \
  --description "Operations repository for <Venture Name>"
```

**Example:**
```bash
gh repo create acmecrane/acme-operations \
  --public \
  --description "Operations repository for ACME Crane"
```

### 1.2 Configure Repository Labels

Apply the standardized label system:

**Status Labels (8):**
```bash
gh label create "status:backlog" -c "d4c5f9" -R <org>/<venture>-operations
gh label create "status:ready" -c "0e8a16" -R <org>/<venture>-operations
gh label create "status:in-progress" -c "fbca04" -R <org>/<venture>-operations
gh label create "status:needs-qa" -c "d876e3" -R <org>/<venture>-operations
gh label create "status:needs-pm" -c "ff6b6b" -R <org>/<venture>-operations
gh label create "status:blocked" -c "b60205" -R <org>/<venture>-operations
gh label create "status:ready-to-merge" -c "0e8a16" -R <org>/<venture>-operations
gh label create "status:done" -c "ededed" -R <org>/<venture>-operations
```

**Routing Labels (3):**
```bash
gh label create "route:dev" -c "1d76db" -R <org>/<venture>-operations
gh label create "route:qa" -c "5319e7" -R <org>/<venture>-operations
gh label create "route:pm" -c "e99695" -R <org>/<venture>-operations
```

**Priority Labels (4):**
```bash
gh label create "prio:P0" -c "d93f0b" -R <org>/<venture>-operations
gh label create "prio:P1" -c "fbca04" -R <org>/<venture>-operations
gh label create "prio:P2" -c "0e8a16" -R <org>/<venture>-operations
gh label create "prio:P3" -c "c5def5" -R <org>/<venture>-operations
```

**Type Labels (4):**
```bash
gh label create "type:bug" -c "d73a4a" -R <org>/<venture>-operations
gh label create "type:feature" -c "a2eeef" -R <org>/<venture>-operations
gh label create "type:docs" -c "0075ca" -R <org>/<venture>-operations
gh label create "type:refactor" -c "bfd4f2" -R <org>/<venture>-operations
```

### 1.3 Add Documentation

Clone the operations repo and add documentation:

```bash
cd /tmp
git clone https://github.com/<org>/<venture>-operations.git
cd <venture>-operations

# Copy standard docs
mkdir -p docs
cp /tmp/crane-operations/docs/CRANE_RELAY_API.md docs/
```

Create a `README.md`:

```markdown
# <Venture Name> Operations

Issue tracking and operations repository for <Venture Name>.

## Documentation

- [Crane Relay API](docs/CRANE_RELAY_API.md) - HTTP API for creating and managing GitHub issues

## Labels

This repository uses the standardized Crane label system. See labels tab for details.

## Command Center

View all venture operations at: https://crane-command.vercel.app

**Password:** crane2026
```

Commit and push:

```bash
git add .
git commit -m "docs: add initial documentation

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
git push -u origin main
```

## Step 2: Configure Crane Relay

The Crane Relay already supports multi-repo operations. No code changes needed.

### 2.1 Verify Access

Ensure the `GITHUB_TOKEN` in Crane Relay has access to the new repository:

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Verify the token has `repo` scope for the organization
3. If using a GitHub App, ensure the app is installed on the new organization

### 2.2 Test Relay Access

Test creating an issue in the new repo:

```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/directive' \
  -H 'Authorization: Bearer <RELAY_TOKEN>' \
  -H 'Content-Type: application/json' \
  -d '{
    "to": "dev",
    "title": "Test issue for new venture",
    "body": "Testing Crane Relay integration",
    "labels": ["type:docs", "status:ready"],
    "repo": "<org>/<venture>-operations"
  }'
```

**Expected response:**
```json
{
  "success": true,
  "issue": 1,
  "url": "https://github.com/<org>/<venture>-operations/issues/1",
  "repo": "<org>/<venture>-operations"
}
```

## Step 3: Add Venture to Command Center

Update Crane Command Center to include the new venture.

### 3.1 Clone Command Center

```bash
cd /tmp
git clone https://github.com/venturecrane/crane-command.git
cd crane-command
npm install
```

### 3.2 Update Venture Configuration

Edit `src/types/github.ts`:

```typescript
// Add new venture ID to union type
export type VentureFilter =
  | 'all'
  | 'venture-crane'
  | 'silicon-crane'
  | 'dfg'
  | 'acme-crane'; // Add this line
```

Edit `src/app/api/github/route.ts`:

```typescript
const VENTURES: VentureConfig[] = [
  {
    id: 'venture-crane',
    name: 'Venture Crane',
    owner: 'venturecrane',
    repo: 'crane-operations',
    color: '#3b82f6',
  },
  {
    id: 'silicon-crane',
    name: 'Silicon Crane',
    owner: 'siliconcrane',
    repo: 'sc-operations',
    color: '#8b5cf6',
  },
  {
    id: 'dfg',
    name: 'Durgan Field Guide',
    owner: 'durganfieldguide',
    repo: 'dfg-console',
    color: '#10b981',
  },
  // Add new venture here
  {
    id: 'acme-crane',
    name: 'ACME Crane',
    owner: 'acmecrane',
    repo: 'acme-operations',
    color: '#f59e0b', // Choose a unique color
  },
];
```

Edit `src/app/page.tsx`:

```typescript
const VENTURE_FILTERS: Array<{ id: VentureFilter; label: string }> = [
  { id: 'all', label: 'All Ventures' },
  { id: 'venture-crane', label: 'Venture Crane' },
  { id: 'silicon-crane', label: 'Silicon Crane' },
  { id: 'dfg', label: 'Durgan Field Guide' },
  { id: 'acme-crane', label: 'ACME Crane' }, // Add this line
];
```

Edit `src/components/features/work-queue-card.tsx`:

Update the venture badge styling to include the new venture color:

```typescript
{/* Venture badge */}
<span
  className={cn(
    'px-2 py-0.5 rounded text-xs font-medium',
    card.venture === 'venture-crane'
      ? 'bg-blue-100 text-blue-700 dark:bg-blue-900/30 dark:text-blue-400'
      : card.venture === 'silicon-crane'
        ? 'bg-purple-100 text-purple-700 dark:bg-purple-900/30 dark:text-purple-400'
        : card.venture === 'dfg'
          ? 'bg-green-100 text-green-700 dark:bg-green-900/30 dark:text-green-400'
          : card.venture === 'acme-crane'
            ? 'bg-amber-100 text-amber-700 dark:bg-amber-900/30 dark:text-amber-400'
            : 'bg-gray-100 text-gray-700 dark:bg-gray-900/30 dark:text-gray-400'
  )}
>
  {card.venture === 'venture-crane' ? 'VC'
    : card.venture === 'silicon-crane' ? 'SC'
    : card.venture === 'dfg' ? 'DFG'
    : card.venture === 'acme-crane' ? 'AC' // Add abbreviation
    : 'Unknown'}
</span>
```

### 3.3 Test Locally

```bash
npm run dev
```

Visit http://localhost:3000 and verify:
- [ ] New venture appears in filter dropdown
- [ ] Issues from new venture display correctly
- [ ] Venture badge shows correct color and abbreviation
- [ ] Filtering by new venture works

### 3.4 Deploy to Vercel

```bash
git add .
git commit -m "feat: add <Venture Name> to Command Center

Add ACME Crane venture to multi-venture dashboard with:
- New venture filter option
- Unique color scheme (amber)
- Issue fetching from acmecrane/acme-operations

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

git push origin main
```

Vercel will automatically deploy the changes.

### 3.5 Verify Production

Visit https://crane-command.vercel.app and verify:
- [ ] New venture appears in dropdown
- [ ] Can view issues from new venture
- [ ] Filtering works correctly

## Step 4: Document and Communicate

### 4.1 Update Documentation

Add the new venture to relevant documentation:

**Update this file** (`docs/NEW_VENTURE_PLAYBOOK.md`):
```markdown
**Existing ventures:**
- Venture Crane (`venturecrane/crane-operations`)
- Silicon Crane (`siliconcrane/sc-operations`)
- Durgan Field Guide (`durganfieldguide/dfg-console`)
- ACME Crane (`acmecrane/acme-operations`)  # Add this
```

**Update Crane Relay docs** (`docs/CRANE_RELAY_API.md`):
```markdown
**Supported repos:**
- `venturecrane/crane-operations` (Venture Crane)
- `siliconcrane/sc-operations` (Silicon Crane)
- `durganfieldguide/dfg-console` (DFG, default)
- `acmecrane/acme-operations` (ACME Crane)  # Add this
```

### 4.2 Notify Team

Send notification to team:

```
New venture added to Crane system:

**ACME Crane** is now live in the Command Center.

- Operations repo: https://github.com/acmecrane/acme-operations
- Command Center: https://crane-command.vercel.app
- Relay API: Use `"repo": "acmecrane/acme-operations"` in API calls

Documentation: https://github.com/venturecrane/crane-operations/blob/main/docs/NEW_VENTURE_PLAYBOOK.md
```

## Checklist

Complete setup checklist:

- [ ] Created `<org>/<venture>-operations` repository
- [ ] Added all standardized labels (15 total)
- [ ] Added README.md and documentation
- [ ] Tested Crane Relay with new repo
- [ ] Updated Command Center types
- [ ] Updated Command Center venture list
- [ ] Updated Command Center filters
- [ ] Updated venture badge styling
- [ ] Tested Command Center locally
- [ ] Deployed Command Center to production
- [ ] Verified production deployment
- [ ] Updated documentation
- [ ] Notified team

## Common Issues

### Issue: Relay returns 401 Unauthorized

**Cause:** GitHub token doesn't have access to new organization

**Fix:**
1. Verify token scopes at https://github.com/settings/tokens
2. Ensure `repo` scope is enabled
3. If using GitHub App, install it on the organization

### Issue: Command Center shows no issues

**Cause:** GitHub API rate limiting or token permissions

**Fix:**
1. Check browser console for API errors
2. Verify `GITHUB_TOKEN` in Vercel environment has access
3. Check GitHub API rate limits

### Issue: Venture badge shows wrong color

**Cause:** Missing or incorrect styling in work-queue-card.tsx

**Fix:**
1. Verify ternary logic in venture badge includes new venture
2. Ensure Tailwind classes are correct
3. Run `npm run build` to check for TypeScript errors

### Issue: Filter dropdown missing new venture

**Cause:** VENTURE_FILTERS array not updated in page.tsx

**Fix:**
1. Add new venture to VENTURE_FILTERS array
2. Ensure label matches venture name
3. Restart dev server if running locally

## Advanced Configuration

### Custom Label Rules

If the new venture needs custom label automation rules, update `LABEL_RULES_JSON` in Crane Relay:

```json
{
  "qa.result_submitted": {
    "PASS": {
      "add": ["status:ready-to-merge"],
      "remove": ["status:needs-qa"]
    },
    "FAIL": {
      "add": ["status:blocked"],
      "remove": ["status:needs-qa"]
    }
  }
}
```

### Custom Components

If the new venture has specific component labels (like `component:api`, `component:worker`), add them to the relay's suggested commands logic in `index.ts`:

```typescript
function getSuggestedCommands(labels: string[]): string[] {
  const commands: string[] = [];

  for (const label of labels) {
    if (label === 'component:acme-api') {
      commands.push('```bash\ncd workers/acme-api\nnpx wrangler deploy\n```');
    }
    // ... more component commands
  }

  return commands;
}
```

## Reference Architecture

```
Multi-Venture System Architecture:

┌─────────────────────────────────────────────────────────────┐
│                    Crane Command Center                      │
│              (https://crane-command.vercel.app)              │
│                                                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │  VC Queue  │  │  SC Queue  │  │  AC Queue  │  ...       │
│  └────────────┘  └────────────┘  └────────────┘            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ GitHub REST API
                            │
                ┌───────────▼──────────────┐
                │    Crane Relay Worker    │
                │  (crane-relay.workers)   │
                │                          │
                │  Multi-repo support:     │
                │  - V1 endpoints          │
                │  - V2 events             │
                └────┬──────────┬──────────┘
                     │          │
          ┌──────────┴──┐   ┌──┴───────────┐   ┌──────────────┐
          │             │   │              │   │              │
    ┌─────▼─────┐ ┌────▼────┐ ┌───▼──────┐ ┌──▼──────┐
    │ VC Ops    │ │ SC Ops  │ │ DFG Ops  │ │ AC Ops  │ ...
    │ (issues)  │ │(issues) │ │(issues)  │ │(issues) │
    └───────────┘ └─────────┘ └──────────┘ └─────────┘
```

## Support

For questions or issues with venture setup:
- GitHub: https://github.com/venturecrane/crane-operations/issues
- Documentation: https://github.com/venturecrane/crane-operations

---

Last updated: 2026-01-13
