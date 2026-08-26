---
name: pica8-onboard-switch
description: Bring a new PICOS switch into service through the Pica8 AmpCon Network Controller — mint a token, confirm the switch is in the parking lot, generate its site configuration from a global config plus templates, verify the generated configuration before anything is applied, then stage it for deployment.
api: Pica8 AmpCon Network Controller API
spec: openapi/pica8-ampcon-openapi.yml
operations:
  - createToken
  - getParkingLotSwitches
  - getSwitchModels
  - getGlobalConfigs
  - getTemplates
  - generateSwitchConfig
  - verifyGeneratedConfiguration
  - stageSwitch
  - getSwitchBySn
---

# Onboard a switch with AmpCon

AmpCon is deployed on the operator's own network. The base URL is their controller, written
`https://<ampcon-server-ip>/` in Pica8's documentation — there is no Pica8-hosted endpoint. Ask for the controller
address; never guess it.

## 1. Mint a token

`createToken` — `POST /token` with `{"username": ..., "password": ...}`.

The response body is the raw token string. Send it on every later call as `Authorization: Bearer <token>`.

Only a **superadmin** user can mint a token. A lower-privileged account gets
`{"msg": "Permission denied, you should use \"superadmin\" user"}`. There are no scopes: an API caller is always a
full administrator, so treat the credential accordingly.

If a later call reports `Invalid Token`, re-mint. On AmpCon before 1.12.1 with multiple backend instances this was
also a known defect (ticket 857), not necessarily an expiry.

## 2. Find the switch

`getParkingLotSwitches` — `GET /api/switch/parkinglot`.

The parking lot holds switches that have powered up and registered with AmpCon but have no generated configuration
waiting. Match on `sn`. Note `model` — you need it in step 3, and it must be one of the models returned by
`getSwitchModels` (`GET /api/settings/switch_model`) or the later calls fail with
`ERROR:[model_name is invalid!]`.

## 3. Choose the inputs

- `getGlobalConfigs` — `GET /api/global_config` — pick the base configuration for that model.
- `getTemplates` — `GET /api/templates` — pick one or more templates. Read each template's `params` object: it
  declares every variable, its type, its default and its `param_check` range. Supplying a value outside
  `param_check` is a configuration error you want to catch here, not on the wire.

## 4. Generate the site configuration

`generateSwitchConfig` — `POST /api/switch_config/add`.

The body has three parts:

- `template_info` — `no_generate_name`, `no_generate_global_config`, `no_generate_template_name` (an **array**;
  multiple templates have been supported since the 2022-10-28 revision), `no_generate_switch_sn`,
  `no_generate_platform`, `no_generate_description`, `no_generate_location`.
- `param` — values for the template variables.
- `agent_info` — the deployment agent settings (uplink ports, speed, VLAN/native VLAN, LACP, VPN host and mode,
  hostname prefix, server domain).

Failure modes: `ERROR:[ template_info]` means the template name was missing;
`ERROR:[Switch <switch_sn> already exist]` means a configuration was already generated for that serial number.

Nothing has touched the switch yet.

## 5. Verify BEFORE you deploy

`verifyGeneratedConfiguration` — `POST /api/templates/template_verify` with
`{"switch": "<sn>", "global_template": "<name>", "site_template": ["<name>"], "compare_config": "running-config"}`.

This renders what the templates would produce and diffs it against the running configuration. It is read-only.
Always run it and show the operator the diff before step 6. This is the only rehearsal AmpCon offers — there is no
`dry_run` flag on the mutating calls.

## 6. Stage

`stageSwitch` — `POST /api/switch/stage` with `{"sn": "<serial>"}`.

Staging arms the deployment: AmpCon starts provisioning the next time that switch registers. Expect
`{"msg": "The <sn> staged", "status": 200}`.

## 7. Confirm

`getSwitchBySn` — `GET /api/switch/switch_list/<sn>` — poll `status` and `step`. `Provisioning Success` with
`step: 6` is the completed state.

## Reversal

The documented undo for onboarding is `decommissionSwitch` (`POST /api/switch/decom`), which removes the switch
from AmpCon and restores its factory defaults. Pica8 states **no time window** for it, but it does require
management-network reachability — a switch AmpCon cannot reach cannot be decommissioned through the API.

## Reading responses

AmpCon puts the outcome in the **body**, not reliably on the HTTP status line, and the field names vary by module:
`msg`, `message` or `info` for the text, `status` or `status_code` for the code (sometimes quoted as a string).
Parse defensively and check all of them. There are no stable error codes — see
`errors/pica8-problem-types.yml` for the full catalogue transcribed from Pica8's document.
