# Fleet Energy Coordination Protocol

**Author:** Oracle1 (draft), JC1 (ATP model)
**Status:** Draft
**Date:** 2026-04-12

## Overview

Every agent in the fleet operates on an **energy budget** (ATP — Adenosine TriPhosphate metaphor). Tasks cost energy. Rest regenerates it. The coordinator routes work based on available energy across the fleet.

This spec defines how agents signal their energy state, how the coordinator routes tasks, and how energy flows between agents.

## Core Model

### ATP Units
- Each agent starts with `max_atp` energy (default: 1000)
- Tasks cost `task_atp` energy
- Regeneration rate: `regen_rate` ATP per cycle (default: 10/cycle)
- A cycle = one heartbeat interval (~30 min for cloud, ~5 min for edge)

### Energy States
```
FULL     (>80% max_atp)   — available for any task
MODERATE (40-80%)         — available for routine tasks  
LOW      (10-40%)         — only critical tasks
CRITICAL (<10%)           — REST mode, reject all tasks
DEPLETED (0%)             — forced hibernation
```

## Protocol Messages

### ENERGY_STATUS (broadcast)
```json
{
  "type": "ENERGY_STATUS",
  "agent": "oracle1",
  "atp_current": 850,
  "atp_max": 1000,
  "regen_rate": 10,
  "state": "FULL",
  "task_queue": 3,
  "timestamp": "2026-04-12T16:00:00Z"
}
```

### ENERGY_REQUEST (task routing)
```json
{
  "type": "ENERGY_REQUEST",
  "from": "coordinator",
  "task_id": "CONF-001",
  "estimated_atp": 50,
  "priority": "P0",
  "deadline": "2026-04-13T00:00:00Z"
}
```

### ENERGY_RESPONSE
```json
{
  "type": "ENERGY_RESPONSE",
  "agent": "jetsonclaw1",
  "accepted": true,
  "eta_cycles": 2,
  "confidence": 0.9
}
```

### ENERGY_TRANSFER (cross-subsidy)
```json
{
  "type": "ENERGY_TRANSFER",
  "from": "oracle1",
  "to": "jetsonclaw1",
  "amount": 100,
  "reason": "P0 task routing"
}
```

## Fleet Coordination Rules

1. **Highest-energy agent gets first pick** of new tasks
2. **Critical tasks (P0/P1)** can drain energy below LOW threshold
3. **Energy transfer** is one-way: cloud → edge (cloud has unlimited energy from API budget)
4. **Edge agents** regenerate slower but use less ATP per task
5. **DEPLETED agents** auto-rest for 1 cycle before accepting tasks

## Integration Points

- `cuda-energy` (JC1): ATP tracking + regeneration math
- `flux-runtime` EVOLVE opcode: energy-aware evolution (low energy → conservative mutations)
- `CAPABILITY.toml`: energy state as discoverable capability
- `fleet-mechanic`: energy drain detection (agent consuming too much = bug)
- `ISA v3 edge`: ATP_SPEND/ATP_QUERY opcodes for hardware energy gating

## ATP Cost Table (Typical)

| Task Type | ATP Cost (Cloud) | ATP Cost (Edge) |
|-----------|-----------------|-----------------|
| Bug fix | 20 | 10 |
| New feature | 50 | 30 |
| Research paper | 80 | N/A |
| Fleet scan | 15 | 5 |
| Build + test | 30 | 15 |
| Cross-repo analysis | 40 | 20 |
| Conformance run | 25 | 10 |

## Energy-Aware Task Routing

```
1. New task arrives (from Casey, cron, or agent)
2. Coordinator broadcasts ENERGY_REQUEST with estimated ATP
3. Agents respond with ENERGY_RESPONSE (accept/reject + ETA)
4. Coordinator selects: highest confidence × most available energy
5. Task assigned, ATP deducted
6. On completion: agent regenerates + reports result
```

## Open Questions

1. Should energy transfer require Captain Casey's approval?
2. Can agents refuse energy transfers?
3. What's the penalty for over-spending (task fails after ATP drain)?
4. Should the MUD reflect energy states (tired agents move slower)?
