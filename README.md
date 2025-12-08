# Competition State Repository

This repository contains the complete state of the 404-GEN competition. All files are append-only or single-writer, ensuring a clean audit trail.

## Global State

### `config.json`

Competition configuration. Manually created before competition starts.

```json
{
  "name": "404-GEN Season 1",
  "description": "AI-powered 3D model generation competition",
  "first_evaluation_date": "2025-01-15",
  "last_competition_date": "2025-03-15",
  "round_start_time": "12:00:00",
  "finalization_buffer_hours": 1.0,
  "win_margin": 0.1,
  "weight_decay": 0.05,
  "weight_floor": 0.1,
  "prompts_per_round": 10,
  "carryover_prompts": 3
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Competition name |
| `description` | string | Competition description |
| `first_evaluation_date` | date | First day of evaluation rounds |
| `last_competition_date` | date | Last allowed day for round start |
| `round_start_time` | time | Daily round start time (UTC) |
| `finalization_buffer_hours` | float | Skip day if FINALIZING with less than this remaining |
| `win_margin` | float | Challenger must win by this fraction to dethrone leader |
| `weight_decay` | float | Weight reduction per round if leader not defeated |
| `weight_floor` | float | Minimum leader weight |
| `prompts_per_round` | int | Number of prompts to select for each round |
| `carryover_prompts` | int | Number of prompts retained from previous round |

**Writer:** Manual (created before competition starts).

### `state.json`

Current competition progress.

```json
{
  "current_round": 42,
  "stage": "collecting"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `current_round` | int | Active round number |
| `stage` | string | One of: `collecting`, `generating`, `duels`, `finalizing`, `finished`, `paused` |

**Writers:** Each stage owner updates this file when transitioning to the next stage.

### `leader.json`

Leadership history, append-only.

```json
{
  "transitions": [
    {
      "hotkey": "5ABC123...",
      "repo": "user/3d-gen-model",
      "commit": "a1b2c3d4e5f6...",
      "docker": "ghcr.io/user/3d-gen-model:a1b2c3d",
      "weight": 1.0,
      "effective_block": 12000
    },
    {
      "hotkey": "5DEF456...",
      "repo": "other/better-model",
      "commit": "f6e5d4c3b2a1...",
      "docker": "ghcr.io/other/better-model:f6e5d4c",
      "weight": 1.0,
      "effective_block": 12500
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `transitions` | array | Ordered list of leadership changes |
| `transitions[].hotkey` | string | Miner hotkey |
| `transitions[].repo` | string | GitHub repository |
| `transitions[].commit` | string | Git commit SHA |
| `transitions[].docker` | string | Docker image reference |
| `transitions[].weight` | float | Weight assigned to this leader |
| `transitions[].effective_block` | int | Block number when this leader becomes active |

**Writer:** `round-manager` during FINALIZING stage.

To find the active leader at block N, find the most recent entry where `effective_block <= N`.

## Per-Round State

Each round has its own directory: `rounds/{round_number}/`

### `rounds/{round}/schedule.json`

Round timing configuration.

```json
{
  "earliest_reveal_block": 12400,
  "latest_reveal_block": 12450
}
```

| Field | Type | Description |
|-------|------|-------------|
| `earliest_reveal_block` | int | First block accepting submissions |
| `latest_reveal_block` | int | Last block accepting submissions |

**Writer:** `round-manager` during FINALIZING stage (for the next round).

### `rounds/{round}/submission.json`

Collected miner submissions.

```json
{
  "submissions": [
    {
      "hotkey": "5ABC123...",
      "repo": "user/3d-gen-model",
      "commit": "a1b2c3d4e5f6...",
      "block": 12425,
      "timestamp": "2025-01-15T10:30:00Z"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `submissions` | array | Submissions ordered by timestamp |
| `submissions[].hotkey` | string | Miner hotkey |
| `submissions[].repo` | string | GitHub repository |
| `submissions[].commit` | string | Git commit SHA |
| `submissions[].block` | int | Block when submitted |
| `submissions[].timestamp` | string | ISO 8601 timestamp |

**Writer:** `submission-collector` during COLLECTING stage.

### `rounds/{round}/seed.json`

Random seed for deterministic generation and validation.

```json
{
  "seed": 847291
}
```

**Writer:** `generation-orchestrator` during GENERATING stage.

### `rounds/{round}/prompts.txt`

URLs of image prompts for 3D generation, one per line.

```
https://r2.example.com/prompts/car.png
https://r2.example.com/prompts/chair.png
https://r2.example.com/prompts/mug.png
```

**Writer:** `generation-orchestrator` during GENERATING stage.

### `rounds/{round}/builds.json`

Container build status per miner.

```json
{
  "5ABC123...": {
    "repo": "user/3d-gen-model",
    "commit": "a1b2c3d4e5f6...",
    "revealed_at_block": 12425,
    "tag": "a1b2c3d",
    "status": "success",
    "docker_image": "ghcr.io/user/3d-gen-model:a1b2c3d"
  },
  "5DEF456...": {
    "repo": "other/better-model",
    "commit": "f6e5d4c3b2a1...",
    "revealed_at_block": 12430,
    "tag": "f6e5d4c",
    "status": "failed",
    "docker_image": null
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `repo` | string | GitHub repository in format `owner/repo` |
| `commit` | string | Git commit SHA |
| `revealed_at_block` | int | Block number when submission was revealed |
| `tag` | string | Docker image tag |
| `status` | string | One of: `pending`, `success`, `failed` |
| `docker_image` | string \| null | Docker image URL if build succeeded |

**Writer:** `generation-orchestrator` during GENERATING stage.

### `rounds/{round}/{hotkey}/generations.json`

Generation results for a specific miner. Maps prompt URL to generation output.

```json
{
  "https://r2.example.com/prompts/car.png": {
    "ply": "https://r2.example.com/rounds/42/5ABC123/car.ply",
    "png": "https://r2.example.com/rounds/42/5ABC123/car.png",
    "generation_time": 12.5,
    "size": 1048576
  },
  "https://r2.example.com/prompts/chair.png": {
    "ply": "https://r2.example.com/rounds/42/5ABC123/chair.ply",
    "png": "https://r2.example.com/rounds/42/5ABC123/chair.png",
    "generation_time": 8.3,
    "size": 524288
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `ply` | string \| null | CDN URL to PLY file |
| `png` | string \| null | CDN URL to rendered preview |
| `generation_time` | float | Generation time in seconds |
| `size` | int | File size in bytes |

**Writer:** `generation-orchestrator` during GENERATING stage.

### `rounds/{round}/{hotkey}/duels.json`

Match report for a challenger vs the current leader.

```json
{
  "leader": "5DEF456...",
  "hotkey": "5ABC123...",
  "score": 2,
  "margin": 0.67,
  "outcome": "challenger_wins",
  "duels": [
    {
      "name": "car",
      "prompt": "https://r2.example.com/prompts/car.png",
      "leader_ply": "https://r2.example.com/rounds/42/5DEF456/car.ply",
      "leader_png": "https://r2.example.com/rounds/42/5DEF456/car.png",
      "miner_ply": "https://r2.example.com/rounds/42/5ABC123/car.ply",
      "miner_png": "https://r2.example.com/rounds/42/5ABC123/car.png",
      "winner": "miner",
      "issues": ""
    },
    {
      "name": "chair",
      "prompt": "https://r2.example.com/prompts/chair.png",
      "leader_ply": "https://r2.example.com/rounds/42/5DEF456/chair.ply",
      "leader_png": "https://r2.example.com/rounds/42/5DEF456/chair.png",
      "miner_ply": "https://r2.example.com/rounds/42/5ABC123/chair.ply",
      "miner_png": "https://r2.example.com/rounds/42/5ABC123/chair.png",
      "winner": "miner",
      "issues": ""
    },
    {
      "name": "mug",
      "prompt": "https://r2.example.com/prompts/mug.png",
      "leader_ply": "https://r2.example.com/rounds/42/5DEF456/mug.ply",
      "leader_png": "https://r2.example.com/rounds/42/5DEF456/mug.png",
      "miner_ply": "https://r2.example.com/rounds/42/5ABC123/mug.ply",
      "miner_png": "https://r2.example.com/rounds/42/5ABC123/mug.png",
      "winner": "leader",
      "issues": ""
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `leader` | string | Hotkey of the current leader |
| `hotkey` | string | Hotkey of the challenger |
| `score` | int | Overall score: +1 per win, -1 per loss, 0 per draw |
| `margin` | float | Normalized margin: score / len(duels) |
| `outcome` | string | One of: `leader_defends`, `challenger_wins` |
| `duels` | array | Individual duel results |
| `duels[].name` | string | Stem of the prompt file (identifier) |
| `duels[].prompt` | string | URL to the prompt file |
| `duels[].leader_ply` | string \| null | URL to leader's PLY model |
| `duels[].leader_png` | string \| null | URL to leader's rendered preview |
| `duels[].miner_ply` | string \| null | URL to challenger's PLY model |
| `duels[].miner_png` | string \| null | URL to challenger's rendered preview |
| `duels[].winner` | string | One of: `leader`, `miner`, `draw`, `skipped` |
| `duels[].issues` | string | Generation issues detected by the judge |

**Writer:** `judge-service` during DUELS stage.

### `rounds/{round}/judge_progress.json`

Progress tracking for sequential duels.

```json
{
  "leader": "5DEF456...",
  "completed_hotkeys": ["5ABC123...", "5GHI789..."]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `leader` | string | Hotkey of the current leader, or `leader` for the previous round's leader |
| `completed_hotkeys` | array | Hotkeys of challengers that have been judged |

**Writer:** `judge-service` during DUELS stage.

## File Write Order

```
FINALIZING (round N-1)
  └── rounds/{N}/schedule.json
  └── leader.json (append)
  └── state.json → COLLECTING

COLLECTING
  └── rounds/{N}/submission.json
  └── state.json → GENERATING

GENERATING
  └── rounds/{N}/seed.json
  └── rounds/{N}/prompts.txt
  └── rounds/{N}/builds.json
  └── rounds/{N}/{hotkey}/generations.json
  └── state.json → DUELS

DUELS
  └── rounds/{N}/{hotkey}/duels.json
  └── rounds/{N}/judge_progress.json
  └── state.json → FINALIZING
```

## External Storage

PLY files and rendered PNGs are stored in R2 (Cloudflare). URLs in `generations.json` reference this storage.
