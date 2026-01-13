# Crane Relay API

**Base URL:** `https://crane-relay.automation-ab6.workers.dev`

Crane Relay is a Cloudflare Worker that enables PM Team to create GitHub issues and manage issue lifecycle via HTTP POST. It eliminates copy-paste handoffs between Claude and GitHub.

## Key Features

- **Multi-repo support**: All V1 endpoints accept an optional `repo` parameter
- **Backwards compatible**: Defaults to `GITHUB_OWNER/GITHUB_REPO` from environment
- **V2 event tracking**: Rolling status comments with build provenance verification
- **Evidence uploads**: Attach screenshots and artifacts to QA events

## Authentication

All endpoints require authentication via Bearer token:

```bash
Authorization: Bearer <RELAY_TOKEN>
```

For V2 endpoints, use:

```bash
X-Relay-Key: <RELAY_SHARED_SECRET>
```

## V1 Endpoints

### Health Check

**GET** `/health`

Returns worker health status.

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2026-01-13T12:00:00.000Z"
}
```

---

### Create Directive

**POST** `/directive`

Create a GitHub issue from a directive.

**Headers:**
```
Authorization: Bearer <RELAY_TOKEN>
Content-Type: application/json
```

**Payload:**
```typescript
{
  to: 'dev' | 'qa' | 'pm';          // Required: Team to route to
  title: string;                     // Required: Issue title
  body: string;                      // Required: Issue body (markdown)
  labels: string[];                  // Required: Labels to add
  assignees?: string[];              // Optional: GitHub usernames
  repo?: string;                     // Optional: 'owner/repo' (defaults to env)
}
```

**Example:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/directive' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "to": "dev",
    "title": "Implement dark mode toggle",
    "body": "Add dark mode toggle to settings page...",
    "labels": ["type:feature", "prio:P1", "status:ready"],
    "assignees": ["scottdurgan"],
    "repo": "venturecrane/crane-operations"
  }'
```

**Response:**
```json
{
  "success": true,
  "issue": 42,
  "url": "https://github.com/venturecrane/crane-operations/issues/42",
  "repo": "venturecrane/crane-operations"
}
```

**Auto-added features:**
- Metadata header with routing and timestamp
- Planning requirement notice (for points >= 3 or prio:P0)
- Suggested commands based on component labels

---

### Add Comment

**POST** `/comment`

Add a comment to an existing GitHub issue.

**Headers:**
```
Authorization: Bearer <RELAY_TOKEN>
Content-Type: application/json
```

**Payload:**
```typescript
{
  issue: number;      // Required: Issue number
  body: string;       // Required: Comment body (markdown)
  repo?: string;      // Optional: 'owner/repo' (defaults to env)
}
```

**Example:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/comment' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 42,
    "body": "QA passed. Ready for production deploy.",
    "repo": "venturecrane/crane-operations"
  }'
```

**Response:**
```json
{
  "success": true,
  "issue": 42,
  "repo": "venturecrane/crane-operations"
}
```

---

### Close Issue

**POST** `/close`

Close a GitHub issue with an optional final comment.

**Headers:**
```
Authorization: Bearer <RELAY_TOKEN>
Content-Type: application/json
```

**Payload:**
```typescript
{
  issue: number;       // Required: Issue number
  comment?: string;    // Optional: Final comment before closing
  repo?: string;       // Optional: 'owner/repo' (defaults to env)
}
```

**Example:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/close' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 42,
    "comment": "Deployed to production. Closing.",
    "repo": "venturecrane/crane-operations"
  }'
```

**Response:**
```json
{
  "success": true,
  "issue": 42,
  "repo": "venturecrane/crane-operations"
}
```

---

### Update Labels

**POST** `/labels`

Add or remove labels on a GitHub issue.

**Headers:**
```
Authorization: Bearer <RELAY_TOKEN>
Content-Type: application/json
```

**Payload:**
```typescript
{
  issue: number;       // Required: Issue number
  add?: string[];      // Optional: Labels to add
  remove?: string[];   // Optional: Labels to remove
  repo?: string;       // Optional: 'owner/repo' (defaults to env)
}
```

**Example:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/labels' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 42,
    "add": ["status:in-progress"],
    "remove": ["status:ready"],
    "repo": "venturecrane/crane-operations"
  }'
```

**Response:**
```json
{
  "success": true,
  "issue": 42,
  "repo": "venturecrane/crane-operations",
  "labels": ["status:in-progress", "type:feature", "prio:P1"]
}
```

---

## V2 Endpoints

V2 endpoints provide event tracking with rolling status comments and build provenance verification.

### Post Event

**POST** `/v2/events`

Record a relay event (dev update, QA result, etc.) and update the rolling status comment.

**Headers:**
```
X-Relay-Key: <RELAY_SHARED_SECRET>
Content-Type: application/json
```

**Payload:**
```typescript
{
  event_id: string;                    // Required: Unique event ID (min 8 chars)
  repo: string;                        // Required: 'owner/repo'
  issue_number: number;                // Required: Issue number
  role: 'QA' | 'DEV' | 'PM' | 'MENTOR'; // Required: Reporter role
  agent: string;                       // Required: Agent name (min 2 chars)
  event_type: string;                  // Required: Event type (e.g., 'qa.result_submitted')

  // Optional fields
  summary?: string;                    // Summary text
  environment?: 'preview' | 'production' | 'dev';
  build?: {
    pr?: number;                       // PR number for provenance check
    commit_sha: string;                // Commit SHA (7-40 hex chars)
  };

  // QA/verdict fields
  overall_verdict?: 'PASS' | 'FAIL' | 'BLOCKED' | 'PASS_UNVERIFIED' | 'FAIL_UNCONFIRMED';
  scope_results?: Array<{
    id: string;
    status: 'PASS' | 'FAIL' | 'SKIPPED';
    notes?: string;
  }>;

  // Required for FAIL/BLOCKED
  severity?: 'P0' | 'P1' | 'P2' | 'P3';
  repro_steps?: string;                // Min 3 chars
  expected?: string;                   // Min 3 chars
  actual?: string;                     // Min 3 chars
  evidence_urls?: string[];

  artifacts?: Array<{
    type: string;
    label?: string;
    href: string;
  }>;
  details?: any;                       // Arbitrary JSON
}
```

**Example - Dev Update:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/events' \
  -H 'X-Relay-Key: <secret>' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_id": "evt_abc12345",
    "repo": "venturecrane/crane-operations",
    "issue_number": 42,
    "role": "DEV",
    "agent": "Claude Sonnet 4.5",
    "event_type": "dev.update",
    "summary": "Implemented dark mode toggle with system preference detection",
    "build": {
      "pr": 123,
      "commit_sha": "a1b2c3d"
    },
    "environment": "preview"
  }'
```

**Example - QA Result:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/events' \
  -H 'X-Relay-Key: <secret>' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_id": "evt_xyz98765",
    "repo": "venturecrane/crane-operations",
    "issue_number": 42,
    "role": "QA",
    "agent": "QA Bot",
    "event_type": "qa.result_submitted",
    "overall_verdict": "PASS",
    "build": {
      "pr": 123,
      "commit_sha": "a1b2c3d"
    },
    "environment": "preview",
    "scope_results": [
      { "id": "toggle_works", "status": "PASS" },
      { "id": "persists_preference", "status": "PASS" }
    ]
  }'
```

**Response:**
```json
{
  "ok": true,
  "event_id": "evt_abc12345",
  "stored": true,
  "rolling_comment_id": "123456789",
  "verdict": "PASS",
  "provenance_verified": true
}
```

**Features:**
- Idempotent: Same `event_id` with same payload returns success
- Provenance verification: Compares commit SHA with PR head
- Auto-downgrades PASS to PASS_UNVERIFIED if provenance fails
- Rolling comment: Upserts a single status comment per issue
- Label automation: Applies label transitions based on event type and verdict

---

### Upload Evidence

**POST** `/v2/evidence`

Upload a file (screenshot, log, etc.) as evidence for an event.

**Headers:**
```
X-Relay-Key: <RELAY_SHARED_SECRET>
Content-Type: multipart/form-data
```

**Form Fields:**
```
repo: string           // Required: 'owner/repo'
issue_number: number   // Required: Issue number
event_id: string       // Optional: Associated event ID
file: File             // Required: File to upload
```

**Example:**
```bash
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/evidence' \
  -H 'X-Relay-Key: <secret>' \
  -F 'repo=venturecrane/crane-operations' \
  -F 'issue_number=42' \
  -F 'event_id=evt_xyz98765' \
  -F 'file=@screenshot.png'
```

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "repo": "venturecrane/crane-operations",
  "issue_number": 42,
  "event_id": "evt_xyz98765",
  "filename": "screenshot.png",
  "content_type": "image/png",
  "size_bytes": 123456,
  "url": "https://crane-relay.automation-ab6.workers.dev/v2/evidence/550e8400-e29b-41d4-a716-446655440000"
}
```

---

### Get Evidence

**GET** `/v2/evidence/{id}`

Download an evidence file.

**Headers:**
```
X-Relay-Key: <RELAY_SHARED_SECRET>
```

**Example:**
```bash
curl 'https://crane-relay.automation-ab6.workers.dev/v2/evidence/550e8400-e29b-41d4-a716-446655440000' \
  -H 'X-Relay-Key: <secret>' \
  -o screenshot.png
```

**Response:**
- Status: 200
- Content-Type: (original file type)
- Content-Disposition: inline; filename="screenshot.png"
- Body: file bytes

---

## Multi-Repo Support

All V1 endpoints support the optional `repo` parameter for multi-venture operations:

```json
{
  "repo": "venturecrane/crane-operations"
}
```

If omitted, the relay defaults to `GITHUB_OWNER/GITHUB_REPO` from the environment (currently `durganfieldguide/dfg-console` for backwards compatibility).

**Supported repos:**
- `venturecrane/crane-operations` (Venture Crane)
- `siliconcrane/sc-operations` (Silicon Crane)
- `durganfieldguide/dfg-console` (DFG, default)

---

## Error Responses

All endpoints return JSON error responses with appropriate HTTP status codes:

**400 Bad Request:**
```json
{
  "error": "Missing required fields: title, body, to"
}
```

**401 Unauthorized:**
```json
{
  "error": "Unauthorized"
}
```

**404 Not Found:**
```json
{
  "error": "Not found"
}
```

**409 Conflict (V2 only):**
```json
{
  "error": "event_id already exists with different payload",
  "details": {
    "event_id": "evt_abc12345",
    "existing_hash": "...",
    "new_hash": "..."
  }
}
```

**500 Internal Server Error:**
```json
{
  "success": false,
  "error": "GitHub API failed"
}
```

---

## CORS

The relay allows requests from:
- `https://app.durganfieldguide.com`
- `https://durganfieldguide.com`
- `http://localhost:3000`

All responses include `Access-Control-Allow-Origin` header with the matched origin.

---

## Deployment

The relay is deployed to Cloudflare Workers:

```bash
cd /tmp/crane-relay
npx wrangler deploy
```

**Environment variables:**
```bash
# V1 (GitHub REST API)
GITHUB_TOKEN=<personal-access-token>
RELAY_TOKEN=<bearer-token-for-v1>
GITHUB_OWNER=durganfieldguide
GITHUB_REPO=dfg-console

# V2 (GitHub App)
RELAY_SHARED_SECRET=<secret-for-v2>
GH_APP_ID=<app-id>
GH_INSTALLATION_ID=<installation-id>
GH_PRIVATE_KEY_PEM=<rsa-private-key>
LABEL_RULES_JSON=<label-transition-rules>
```

---

## Rolling Status Comment

V2 events trigger an automatic "rolling status comment" that displays:

- **Current State**: Status, labels, owner
- **Build Provenance**: Environment, PR, commit, verification status
- **Latest Dev Update**: Summary from last `dev.update` event
- **Latest QA Result**: Verdict, scope results, evidence URLs
- **Recent Activity**: Last 5 events with timestamps

The comment is automatically created or updated on each event. It includes a marker (`<!-- RELAY_STATUS v2 -->`) for identification.

**Example rolling comment:**

```markdown
<!-- RELAY_STATUS v2 -->

## Relay Status — ISSUE #42

### Current State
- Status: `in-progress`
- Labels: `status:in-progress`, `type:feature`, `prio:P1`
- Owner: @scottdurgan

### Build Provenance
- Environment: `preview`
- PR: #123
- Commit: `a1b2c3d`
- Provenance: VERIFIED (matches PR head)

### Latest Dev Update
- Summary: Implemented dark mode toggle with system preference detection

### Latest QA Result
- Verdict: `PASS`
- Scope:
  - toggle_works — PASS
  - persists_preference — PASS
- Evidence:
  - https://crane-relay.automation-ab6.workers.dev/v2/evidence/550e8400-e29b-41d4-a716-446655440000

### Recent Activity
- 14:32Z — qa.result_submitted — QA Bot
- 14:15Z — dev.update — Claude Sonnet 4.5
- 13:45Z — dev.started — Claude Sonnet 4.5
```

---

## Label Automation

V2 events can trigger automatic label transitions based on `LABEL_RULES_JSON` configuration:

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

The relay applies these rules after storing each event.

---

## Examples

### Complete workflow using V1 endpoints

```bash
# 1. Create directive
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/directive' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "to": "dev",
    "title": "Add export button to dashboard",
    "body": "Users should be able to export dashboard data as CSV",
    "labels": ["type:feature", "status:ready"],
    "repo": "venturecrane/crane-operations"
  }'

# 2. Update labels when dev starts
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/labels' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 45,
    "add": ["status:in-progress"],
    "remove": ["status:ready"],
    "repo": "venturecrane/crane-operations"
  }'

# 3. Add comment when ready for QA
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/comment' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 45,
    "body": "Export functionality complete. Ready for QA in preview environment.",
    "repo": "venturecrane/crane-operations"
  }'

# 4. Close after merge
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/close' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "issue": 45,
    "comment": "Merged and deployed to production.",
    "repo": "venturecrane/crane-operations"
  }'
```

### Complete workflow using V2 endpoints

```bash
# 1. Dev starts work
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/events' \
  -H 'X-Relay-Key: <secret>' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_id": "evt_dev_start_001",
    "repo": "venturecrane/crane-operations",
    "issue_number": 45,
    "role": "DEV",
    "agent": "Claude Sonnet 4.5",
    "event_type": "dev.started",
    "summary": "Starting implementation of CSV export feature"
  }'

# 2. Dev completes work
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/events' \
  -H 'X-Relay-Key: <secret>' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_id": "evt_dev_complete_001",
    "repo": "venturecrane/crane-operations",
    "issue_number": 45,
    "role": "DEV",
    "agent": "Claude Sonnet 4.5",
    "event_type": "dev.update",
    "summary": "Implemented CSV export with proper encoding and date formatting",
    "build": {
      "pr": 156,
      "commit_sha": "f3e4d5c"
    },
    "environment": "preview"
  }'

# 3. QA uploads evidence
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/evidence' \
  -H 'X-Relay-Key: <secret>' \
  -F 'repo=venturecrane/crane-operations' \
  -F 'issue_number=45' \
  -F 'event_id=evt_qa_result_001' \
  -F 'file=@export-test.png'

# 4. QA submits result
curl -X POST 'https://crane-relay.automation-ab6.workers.dev/v2/events' \
  -H 'X-Relay-Key: <secret>' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_id": "evt_qa_result_001",
    "repo": "venturecrane/crane-operations",
    "issue_number": 45,
    "role": "QA",
    "agent": "QA Team",
    "event_type": "qa.result_submitted",
    "overall_verdict": "PASS",
    "build": {
      "pr": 156,
      "commit_sha": "f3e4d5c"
    },
    "environment": "preview",
    "scope_results": [
      { "id": "export_button_visible", "status": "PASS" },
      { "id": "csv_format_valid", "status": "PASS" },
      { "id": "filename_correct", "status": "PASS" }
    ],
    "evidence_urls": [
      "https://crane-relay.automation-ab6.workers.dev/v2/evidence/550e8400-e29b-41d4-a716-446655440000"
    ]
  }'
```

---

## Best Practices

1. **Use V2 for new features**: V2 provides better tracking and provenance verification
2. **Include build info**: Always include PR and commit SHA for provenance checks
3. **Upload evidence first**: Upload screenshots before submitting QA results
4. **Use unique event IDs**: Generate UUIDs or use `evt_<timestamp>_<random>` format
5. **Specify repo explicitly**: Always include `repo` parameter for multi-venture clarity
6. **Test locally first**: Use curl examples to verify payloads before integrating

---

## Support

For issues or questions:
- GitHub: https://github.com/venturecrane/crane-relay/issues
- Docs: https://github.com/venturecrane/crane-operations

Last updated: 2026-01-13
