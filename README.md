# Crane Operations

Issue tracking and operations repository for Venture Crane.

## Documentation

- [Crane Relay API](docs/CRANE_RELAY_API.md) - HTTP API for creating and managing GitHub issues

## Labels

This repository uses a standardized label system:

**Status (8):**
- `status:backlog` - Not yet ready for work
- `status:ready` - Ready to be picked up
- `status:in-progress` - Currently being worked on
- `status:needs-qa` - Waiting for QA validation
- `status:needs-pm` - Waiting for PM review
- `status:blocked` - Blocked by external dependency
- `status:ready-to-merge` - Approved and ready for merge
- `status:done` - Completed and merged

**Routing (3):**
- `route:dev` - Assigned to Dev Team
- `route:qa` - Assigned to QA Team
- `route:pm` - Assigned to PM Team

**Priority (4):**
- `prio:P0` - Critical (drop everything)
- `prio:P1` - High priority
- `prio:P2` - Medium priority
- `prio:P3` - Low priority

**Type (4):**
- `type:bug` - Something isn't working
- `type:feature` - New functionality
- `type:docs` - Documentation changes
- `type:refactor` - Code improvement without behavior change

## Workflows

Issues flow through the following states:

```
backlog → ready → in-progress → needs-qa → ready-to-merge → done
                      ↓
                  blocked (can return to in-progress)
```

## Command Center

View all Venture Crane operations at: https://crane-command.vercel.app

**Password:** crane2026

## Related Repositories

- [venturecrane/crane-relay](https://github.com/venturecrane/crane-relay) - GitHub issue automation worker
- [venturecrane/crane-command](https://github.com/venturecrane/crane-command) - Multi-venture command center
- [siliconcrane/sc-operations](https://github.com/siliconcrane/sc-operations) - Silicon Crane operations
- [durganfieldguide/dfg-console](https://github.com/durganfieldguide/dfg-console) - DFG operations
