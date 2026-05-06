# Calibration Sync & Application Reference

This document is the single source of truth for how calibrations flow from the analysis machine to the Pi scanner. Consult it when a calibration isn't applying, when adding new endpoints, or when debugging sync state.

---

## Overview

Calibrations are data-driven adjustments to the option scanner's scoring config (`scanner_config.json`). They are produced by the analysis pipeline or authored by hand, versioned in this repo, sent to the Pi via HTTP, and applied to the live scanner. The system is designed so that:

- A pipeline run **never auto-stages or auto-sends** a calibration — every send requires explicit user action.
- Calibrations are **idempotent**: applying the same file twice produces the same config state.
- The Pi is the **source of truth** for what has been applied; the local send log is an audit trail only, not a status fallback.
- **Version filtering** ensures calculated calibrations generated against an old strategy version are never sent after a strategy change. Coded calibrations bypass this filter entirely.

---

## Two Calibration Types

| Type | `source` field | Produced by | Version filter |
|---|---|---|---|
| **Calculated** | `"generated"` | `--mode pipeline` / `--mode calibrate` | Must match current strategy version |
| **Coded** | `"coded"` | Manually authored and committed to this repo | Bypassed — always shown as pending until applied |

**Coded calibrations** encode expert knowledge: baseline values, emergency corrections, bootstrap configs. They are authored directly as JSON files in `calibrations/` with `"source": "coded"` and a `"strategy_version"` for documentation purposes only (not enforced). The `send-outstanding` flow sends them alongside calculated calibrations.

---

## Repos & Machines

| Repo / Service | Machine | Purpose |
|---|---|---|
| `option-analysis` | Analysis machine | Generates calibrations, stages them, sends to Pi |
| `options-calibrations` (this repo) | Both (cloned) | Immutable archive of every calibration ever generated or coded |
| `option-getter` | Pi (Raspberry Pi) | Receives and applies calibrations; owns the applied history DB |

Both machines must have `options-calibrations` cloned **as a sibling** to `option-analysis` / `option-getter`:

```
options/
  option-analysis/
  option-getter/
  options-calibrations/     ← this repo
    calibrations/
    docs/
```

---

## Calibration File Format

Files in `calibrations/` follow the naming convention:

```
{YYYYMMDD}_{HHMMSS}_{strategy_version}_{calibration_id}.json
```

Example: `20260414_161600_6d5938c5-f2eec90c_154ce65f.json`

- **`strategy_version`** — `{factors_hash[:8]}:{config_hash[:8]}`, auto-computed from MD5 of all scanner factor files + `scanner_config.json`. Colons are replaced with hyphens in filenames. For coded calibrations this is informational (e.g. `"bootstrap:00000000"` or the version at time of authoring).
- **`calibration_id`** — first 8 chars of SHA256 of the sorted JSON content. Used for deduplication everywhere.

Key fields inside each file:

```json
{
  "generated_at":      "2026-04-14T16:16:00",
  "strategy_version":  "6d5938c5:f2eec90c",
  "sample_size":       61,
  "confidence":        "medium",
  "source":            "generated",
  "adjustments": {
    "max_spread_pct": {
      "current": 0.08, "suggested": 0.10, "delta": 0.02,
      "direction": "raise", "reason": "...", "priority": "MEDIUM"
    }
  },
  "factor_weight_adjustments": {
    "liquidity": {
      "config_key": "fw_liquidity",
      "current": 1.0, "suggested": 1.15, "priority": "HIGH"
    }
  },
  "hold_adjustment": null,
  "staged_from_file": "20260414_161600_..."
}
```

**Valid `source` values:**
- `"generated"` — produced by `--mode pipeline` / `--mode calibrate`
- `"manual"` — staged from the archive picker (`--mode stage-calibration` interactive)
- `"coded"` — manually authored JSON committed directly to this repo; bypasses version filtering in all send/list operations

---

## Two-Slot Model

There are two distinct slots on the analysis machine. Mixing them up is the most common cause of "why didn't this send?".

| Slot | Path | Written by | Purpose |
|---|---|---|---|
| **Generated** | `option-analysis/cache/calibration_generated.json` | `--mode pipeline` / `--mode calibrate` | Pipeline output. Never auto-staged or sent. |
| **Staged** | `option-analysis/cache/calibration.json` | `--mode stage-calibration` only | The file that `send-calibration` reads. Explicit user action required to populate it. |

A pipeline run **cannot clobber** a manually staged calibration because it only writes to the generated slot. The staged slot is only touched by the staging commands.

---

## Full Workflow

### Standard (analysis machine → Pi)

```
1. Run pipeline
   python main.py --mode pipeline
   → writes cache/calibration_generated.json
   → archives to options-calibrations/calibrations/{ts}_{ver}_{id}.json
   → commits to this repo (or via iOS "Commit to Repo" button)

2. Stage the generated calibration
   python main.py --mode stage-calibration --generated
   → copies calibration_generated.json → cache/calibration.json
   → prompts to overwrite if calibration.json already exists

3. Send to Pi
   python main.py --mode send-calibration
   OR: iOS app → Scoring Analysis → Calibration → "Send to Pi"
   → POST /calibration/upload on option-getter
   → Pi records to database/calibrations.db
   → Pi applies (if auto_apply=true) or queues
   → Response includes current_config; analysis machine saves to cache/pi_scanner_config.json
   → Analysis machine appends to cache/calibration_send_log.jsonl (audit record)
   → calibration.json renamed to calibration_sent_{ts}_{id}.json (cleared from staged slot)
```

### Authoring a coded calibration

```
1. Create a JSON file in calibrations/ with the standard format plus "source": "coded"
   Use filename: {YYYYMMDD}_{HHMMSS}_coded_{calibration_id}.json
   Set strategy_version to "coded" or the version you're targeting (informational only)

2. Commit to this repo:
   git add calibrations/<file>
   git commit -m "Add coded calibration: <description>"
   git push
   OR: iOS app → "Commit to Repo"

3. On both machines, pull the repo:
   cd options-calibrations && git pull

4. Send via send-outstanding (coded calibrations always appear regardless of version):
   python main.py --mode send-outstanding
   OR: iOS app → Pending Calibrations → "Send All to Pi"
```

### Sending multiple outstanding calibrations

```
python main.py --mode list-archive          # see what's pending
python main.py --mode send-outstanding      # send all pending (calculated current version + all coded)
OR: iOS app → Scoring Analysis → Pending Calibrations → "Send All to Pi"
→ POST /calibration/send-outstanding on option-analysis server
→ Sends each pending file oldest-first via POST /calibration/upload
```

### Recovery (Pi reinstall / missing history)

```
# On option-getter machine:
python main.py --mode apply-latest-calibration
# OR (auto-approve HIGH priority only):
python main.py --mode apply-latest-calibration --yes
# OR (auto-approve everything):
python main.py --mode apply-latest-calibration --yes-all
```

Requires `options-calibrations` cloned on the Pi. Reads the newest file in `calibrations/` not already in the Pi's history DB.

---

## Applied History: `database/calibrations.db`

The Pi maintains a SQLite database at `option-getter/database/calibrations.db` (not in this repo).

**Schema:**

```sql
CREATE TABLE calibration_history (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    applied_at       TEXT NOT NULL,
    calibration_id   TEXT NOT NULL,
    generated_at     TEXT,
    strategy_version TEXT,
    sample_size      INTEGER,
    confidence       TEXT,
    auto_applied     INTEGER,   -- 0/1
    apply_mode       TEXT,      -- "auto" | "high_only" | "all"
    changes_count    INTEGER,
    changes_applied  TEXT       -- JSON array of change strings
);
CREATE UNIQUE INDEX idx_cal_id ON calibration_history(calibration_id);
```

The `UNIQUE INDEX` on `calibration_id` means re-sending the same calibration is a no-op — `INSERT OR IGNORE` is used on every write, so double-applies are always safe.

**The Pi is the source of truth.** `list-archive`, `send-outstanding`, and `GET /calibration/pending-archive` all query the Pi's `GET /calibration/applied-ids` endpoint to determine status. If the Pi is unreachable, these operations show `[?]` (CLI) or return `{"pi_reachable": false}` (HTTP) — they do not fall back to the local `calibration_send_log.jsonl`. The send log exists as an audit record only.

**Migration**: On first startup after the upgrade, any existing `cache/calibration_history.jsonl` records are auto-imported into the DB and the JSONL file is renamed to `calibration_history.jsonl.migrated` (kept as backup, not deleted).

---

## Version Filtering

Every calibration file carries a `strategy_version` string.

| Status | Condition |
|---|---|
| `[applied {date}]` | `calibration_id` is in Pi's `calibration_history` table |
| `[pending]` | Not applied, `strategy_version` matches current archive version |
| `[coded-pending]` | `source == "coded"`, not applied — version filter bypassed |
| `[old-version]` | Not applied, `strategy_version` is older than the current archive version |
| `[?]` | Pi unreachable — status unknown |

`send-outstanding` and `GET /calibration/pending-archive` surface `[pending]` and `[coded-pending]` entries. `[old-version]` entries are skipped for calculated calibrations. `[coded-pending]` entries are always included regardless of version.

**Current version** is determined by reading the `strategy_version` from the most recently dated non-coded file in `calibrations/`.

---

## Relevant Endpoints

### option-analysis (port 9200) — analysis machine

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/calibration` | API key | Return staged `cache/calibration.json` |
| `DELETE` | `/calibration` | API key | Delete staged calibration |
| `POST` | `/calibration/send` | API key | Push staged `calibration.json` to Pi |
| `POST` | `/calibration/commit-archive` | API key | `git add/commit/push` new files in this repo |
| `GET` | `/calibration/pending-archive` | API key | List archive entries not yet applied to Pi; queries Pi's `/calibration/applied-ids`. Returns `{"pi_reachable": false}` with empty list if Pi is unreachable. |
| `POST` | `/calibration/send-outstanding` | API key | Send all pending archive entries to Pi oldest-first; returns 503 if Pi is unreachable |

Key behaviors of `POST /calibration/send`:
- Reuses `cache/pi_scanner_config.json` if it was written within the last 5 minutes (pipeline just synced it), to avoid a redundant Pi network call.
- Returns 422 if Pi's config already matches all suggested values ("already applied" guard).
- Archives `calibration.json` to `calibration_sent_{ts}_{id}.json` on success (clears staged slot).
- Records to `cache/calibration_send_log.jsonl` on success (audit record).

### option-getter (port 8001) — Pi

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `POST` | `/calibration/upload` | Bearer `CALIBRATION_TOKEN` | Receive a calibration; save to inbox; optionally auto-apply; record to `calibrations.db`. Returns **409** if already applied or already in queue (queue mode). |
| `GET` | `/calibration/status` | Bearer `CALIBRATION_TOKEN` | Return pending calibration summary + last applied record |
| `GET` | `/calibration/applied-ids` | Bearer `CALIBRATION_TOKEN` | Return all applied calibration IDs with strategy version and timestamp |
| `GET` | `/calibration/current-config` | Bearer `CALIBRATION_TOKEN` | Return current `scanner_config.json` (used as calibration baseline) |
| `POST` | `/calibration/apply` | Bearer `CALIBRATION_TOKEN` | Manually apply pending calibration (review mode) |
| `POST` | `/calibration/apply-all` | Bearer `CALIBRATION_TOKEN` | Auto-apply all pending changes |

**`GET /calibration/applied-ids` response:**

```json
{
  "applied": [
    {
      "calibration_id":   "154ce65f",
      "strategy_version": "6d5938c5:f2eec90c",
      "applied_at":       "2026-04-14T16:20:00Z"
    }
  ],
  "count": 1
}
```

This is the authoritative source for "what has the Pi actually applied." All status checks query it directly.

---

## Config Files on the Analysis Machine

| File | Written by | Purpose |
|---|---|---|
| `cache/calibration_generated.json` | `--mode pipeline` / `--mode calibrate` | Pipeline output; never auto-staged |
| `cache/calibration.json` | `--mode stage-calibration` | Staged-for-send slot; read by `send-calibration` |
| `cache/calibration_send_log.jsonl` | `send-calibration` / `send-outstanding` | **Audit record only** — not used to determine Pi status |
| `cache/pi_scanner_config.json` | Pipeline + send response | Cached copy of Pi's config; used as calibration baseline and send guard |
| `cache/analysis_config.json` | `--mode configure` | Stores `option_getter_api_url` and `option_getter_api_token` |

---

## Debugging: Calibration Not Applied

Work through this checklist top-to-bottom.

### 1. Is the calibration in the Pi's history?

```bash
# On option-getter machine:
sqlite3 database/calibrations.db \
  "SELECT calibration_id, applied_at, changes_count FROM calibration_history ORDER BY applied_at DESC LIMIT 10;"
```

If the `calibration_id` is present → it was applied. The scanner config is already updated. Re-running the pipeline and comparing `current` vs. `suggested` should show no delta.

If absent → continue below.

### 2. Is the calibration in the Pi's inbox but not yet applied?

```bash
# On option-getter machine:
ls cache/calibration_inbox/
```

Files here were received but not yet applied (Pi is in queue mode — `calibration_auto_apply = false` in `scanner_config.json`). Apply them:

```bash
python main.py --mode apply-calibration
# or auto-approve HIGH priority only:
python main.py --mode apply-calibration --yes
```

### 3. Was the calibration sent at all?

Check the Pi's applied-ids endpoint directly — the send log is an audit record, not a status source:

```bash
curl -H "Authorization: Bearer <token>" http://<pi>:8001/calibration/applied-ids
```

If the calibration_id is absent from the Pi's applied history → the send never completed or the Pi hasn't applied it yet. Check:
- Was `cache/calibration.json` (staged slot) populated before the send? If not, run `--mode stage-calibration --generated` first.
- Did `--mode send-calibration` return an error? Re-run it and read the output.
- Is the Pi reachable? `curl -H "Authorization: Bearer <token>" http://<pi>:8001/health`

### 4. Is the calibration showing [old-version] in list-archive?

```bash
python main.py --mode list-archive
```

If a calculated calibration shows `[old-version]`, it was generated against an older strategy version. Options:
- If the old calibration is still relevant, manually stage and send it: `--mode stage-calibration --file <path>` → `--mode send-calibration`.
- If the strategy changed intentionally, run a fresh pipeline to generate a calibration for the new version.

Note: coded calibrations (`source: "coded"`) always show as `[coded-pending]` regardless of strategy version — they are not subject to version filtering.

### 5. Did the send guard block the send (422 "already applied")?

This means Pi's `scanner_config.json` already has every value the calibration suggested. This is the normal state after a successful apply+pipeline cycle. No action needed — the calibration is in effect.

To verify: compare `calibration.json`'s `adjustments[key].suggested` against `GET /calibration/current-config` from the Pi.

### 6. Pi unreachable from analysis machine

```bash
# Check Pi reachability:
curl -H "Authorization: Bearer <token>" http://<pi>:8001/calibration/applied-ids

# If unreachable, check analysis_config.json:
cat cache/analysis_config.json
```

If the URL is wrong, run `python main.py --mode configure` to update it. Ensure the Pi is on the VPN (Tailscale) if connecting remotely.

When the Pi is unreachable:
- `list-archive` shows `[?]` for all unapplied entries
- `send-outstanding` aborts immediately (does not fall back to send log)
- `GET /calibration/pending-archive` returns `{"pi_reachable": false}` with an empty list

---

## Auto-Apply Setting

The Pi's `calibration_auto_apply` flag in `scanner_config.json` (editable via `--mode edit-config` on option-getter) controls what happens when a calibration arrives via `POST /calibration/upload`:

| Setting | Behavior |
|---|---|
| `true` (default) | All changes applied immediately on receipt (not added to queue) |
| `false` | Calibration saved to queue; nothing applied until `apply-calibration` is run manually |

The iOS app's Pi Calibration Status section reflects which mode is active and shows pending queue items.

---

## Deduplication Logic

Deduplication happens at multiple layers:

1. **`calibration_id`** — computed as `SHA256(sorted JSON content)[:8]`. Same content = same ID everywhere.
2. **Pi DB** — `INSERT OR IGNORE` on `calibration_id` unique index. A re-sent file is silently ignored.
3. **Queue-mode dedup** — `POST /calibration/upload` returns **409** if the calibration is already applied or already in the queue (queue mode only; auto-apply mode is always safe because the DB dedup handles it).
4. **Send guard** — `POST /calibration/send` checks Pi's live config before sending; returns 422 if all suggested values already match.
5. **`send-outstanding`** — filters by `calibration_id not in applied_ids` before building the send list.

So it is always safe to re-run `--mode send-outstanding` or retry `--mode send-calibration`. The worst case is a 409 (duplicate) or 422 (already applied) response from the Pi.

---

## Adding New Calibration Adjustment Types

When adding a new field to the calibration output format:

1. **`analysis/calibrator.py`** — add the new key to the `output["adjustments"]` or `output["factor_weight_adjustments"]` dict.
2. **`option-getter/services/calibration_applier.py`** — add the apply logic for the new key.
3. **`option-getter/ticker_server/calibration_routes.py`** — if the new field needs to be tracked in `changes_applied`, update `_record_history()`.
4. **`option-analysis/server.py`** — if the new field affects the send guard, update `_ADJ_KEY_MAP`.
5. **This doc** — update the "Calibration File Format" section above.
