---
name: pica8-fleet-audit
description: Read-only inventory and health sweep of a PICOS fleet through AmpCon — every switch and its state, switches stuck in the parking lot, image currency against the model catalogue, group membership, licence expiry, and the operation log. No write operations.
api: Pica8 AmpCon Network Controller API
spec: openapi/pica8-ampcon-openapi.yml
operations:
  - createToken
  - getAllSwitches
  - getDeployedSwitches
  - getParkingLotSwitches
  - getImportedSwitches
  - getSwitchModels
  - getSwitchGroups
  - auditSwitchLicense
  - auditGroupLicenses
  - getSwitchLogs
---

# Audit a PICOS fleet, without changing anything

Every operation in this skill is a read, with one deliberate exception noted below. It is the safe skill to hand an
agent.

## 0. Token

`createToken` — `POST /token`. Superadmin only; AmpCon has no read-only API credential.

## 1. Inventory

- `getAllSwitches` — `GET /api/switch/all_switch_list` — the whole fleet. Each record carries `sn`, `hwid`,
  `host_name`, `mgt_ip`, `tmp_ip`, `status`, `step`, `topology` and `version`.
- `getDeployedSwitches` — `GET /api/switch/deployed_switch_list` — those that finished deployment.
- `getImportedSwitches` — `GET /api/switch/import` — those adopted rather than zero-touch provisioned.

None of these endpoints paginates. There is no `limit`, `offset` or cursor anywhere in the AmpCon contract, so each
call returns the complete array — size the request accordingly on a large fleet.

## 2. Find what is stuck

`getParkingLotSwitches` — `GET /api/switch/parkinglot`.

A switch in the parking lot has registered with AmpCon but has no generated configuration to deploy. `register_count`
and `history_time` show how many times it has tried. A high `register_count` is a switch that has been looping —
that is the number worth surfacing.

## 3. Image currency

`getSwitchModels` — `GET /api/settings/switch_model` — gives `up_to_date_version` per model. Join it to each
switch's `version` from step 1 to produce the drift list: switch serial, model, running version, current version.

## 4. Groups

`getSwitchGroups` — `GET /api/switch/groups` — group name, member serial numbers, and `action_array`, the set of
group-scoped operations permitted: `audit`, `action`, `upgrading`, `retrieve_config`. A group whose `action_array`
omits `audit` cannot be licence-audited as a group.

## 5. Licences

`auditSwitchLicense` (`POST /api/switch/license_audit`, `{"sn": ...}`) and `auditGroupLicenses`
(`POST /api/switch/groups/license_audit`, `{"group": ...}`) check whether a licence has expired or expires within
30 days.

These are POSTs and they are the one non-GET in this skill, but they are audits, not changes. Do **not** confuse
them with `applySwitchLicense` / `applyGroupLicenses` (`license_action`), which push new licences onto hardware —
those are writes and are out of scope here.

The group form runs in the background and answers immediately with
`{"status": 200, "msg": "license audit successful in the background"}` — the response does not mean the audit is
finished.

## 6. Operation log

`getSwitchLogs` — `GET /api/switch/log` — AmpCon's own audit trail: `sn`, `history_time`, `msg`, `type`
(severity) and `status` (read/unread). It is the only audit surface the API exposes, and entries carry no request
identifier, so correlate by serial number and timestamp.

## Do not call

`getSystemConfig` (`GET /api/settings/system_config`) is a read, but it returns the licence portal password and the
switch SSH operation password in plaintext. Keep it out of any agent-reachable tool set.
