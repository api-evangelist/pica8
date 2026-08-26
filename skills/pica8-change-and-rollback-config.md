---
name: pica8-change-and-rollback-config
description: Change configuration on a live PICOS switch through AmpCon the safe way — snapshot first, diff the change before applying it, push, verify, and roll back to the snapshot if the change is wrong.
api: Pica8 AmpCon Network Controller API
spec: openapi/pica8-ampcon-openapi.yml
operations:
  - createToken
  - getSwitchBySn
  - backupSwitchConfiguration
  - getBackupConfigsBySn
  - compareBackupWithRunningConfig
  - verifyGeneratedConfiguration
  - getConfigFiles
  - pushConfigFileToSwitch
  - rollbackBackupConfiguration
  - getSwitchLogs
---

# Change configuration safely, and be able to take it back

This is a **high-consequence** flow. `pushConfigFileToSwitch` changes the running configuration of production
network hardware. Do not run it without an operator's explicit approval on a concrete diff.

## 0. Token

`createToken` — `POST /token`. Superadmin only. See `authentication/pica8-authentication.yml`.

## 1. Snapshot FIRST

`backupSwitchConfiguration` — `POST /api/backup_config/<switch-sn>`.

Take the snapshot **before** the change, not after. Rollback restores from a snapshot, so a change made without one
has no undo. Confirm it landed with `getBackupConfigsBySn` (`GET /api/backup_config/<switch-sn>`), which lists
every snapshot AmpCon holds for that serial number.

Pica8 publishes **no retention period** for snapshots. Do not tell an operator how long they have to roll back —
that number is not documented. If they need a guarantee, tell them to keep their own copy.

## 2. Rehearse the change

Two read-only comparisons exist. Use the one that matches what you are about to do:

- `compareBackupWithRunningConfig` — `POST /api/compare_config` — diffs a stored snapshot against the running
  configuration. Use it to see what has drifted since the snapshot.
- `verifyGeneratedConfiguration` — `POST /api/templates/template_verify` — renders what a global config plus site
  templates would generate for the switch and diffs that against `running-config`. Use it when the change comes
  from templates.

Show the diff. Get approval on the diff, not on a description of it.

## 3. Push

`getConfigFiles` — `GET /api/config_files` — find the configuration file by `name`. The tree is keyed by `pid` and
`level`.

`pushConfigFileToSwitch` — `POST /api/config_files/push`.

Two things Pica8's own release notes record about this operation, worth knowing before you call it:

- The input parameters for this call were **corrected** in the 2022-10-27 revision of the API document. A client
  written against the earlier document sends the wrong body.
- Pushes containing `delete` commands were broken until AmpCon 1.14.1 (ticket 890), and S5810/S5860 switches did
  not automatically `save_config` after a push until the same release (ticket 884). Check the controller version.

## 4. Verify

Re-run `compareBackupWithRunningConfig` against the pre-change snapshot to confirm the delta is exactly what was
approved, and read `getSwitchLogs` (`GET /api/switch/log`) for the operation record. Log entries carry `sn`,
`history_time`, `msg`, `type` and `status` — they do not carry a request id, so correlate on serial number and
timestamp.

## 5. Roll back if it is wrong

`rollbackBackupConfiguration` — `POST /api/backup_config/rollback`.

This is the documented reversal path for a configuration change. It restores the switch from a snapshot taken in
step 1. Again: no stated window, so roll back promptly rather than assuming the snapshot will still be there.

## What has no undo

There is no restore path for deleted objects — templates, global configurations, configuration files, playbooks,
groups and jobs are gone once deleted. The only guard is referential: deleting a global configuration still used by
a deployment is refused with `ERROR:[config is in use!]`. Everything else deletes silently and permanently.
