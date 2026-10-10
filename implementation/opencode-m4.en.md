# M4 OpenCode: final controlled-model acceptance

[中文](opencode-m4.md) | English

On 2026-10-10, `Cloud 5b6b5ec` / `Desktop f8536f9d` / `Cluster a8146c7` passed both protocol API paths, complete Chromium flows and exact-resource verification. The production CLI in fixed `official/ora-space.opencode@0.6.4` reads LICENSE, writes files and commits Git through the platform model proxy. Only the upstream model is controlled; the agent is not Echo or simulated.

## Final direct evidence

Cluster's ignored `.acceptance/` retains safe states and IDs without keys, Cookies or tokens. Browser and host verification refer to the same Run; API results are independent supplements.

| Entry / protocol | Run | Revision | Final Git commit |
| --- | --- | --- | --- |
| Browser OpenAI / `openai-completions` | `ff53d668-4fde-4105-b903-61be7acf1f11` | `fcf6d81f-602e-4d87-9292-f39816555ad3` | `096fe77e4a3cb2262abecb3153bbd8b131c43a38` |
| Browser Anthropic / `anthropic-messages` | `e8617fbf-33e5-4f67-af41-d84f8e998478` | `cd3306ae-dc9e-4151-a604-1ed0601ec2b9` | `233e85021b43d411ad35e19b72833ff18950574d` |
| API / `openai-completions` | `72de9f9e-0289-4610-8ec4-1a85382c8973` | `e926d01a-f3d5-4515-8108-bc618811e7e7` | `bf1876c542e724ba8e84e5035311f07bf6ffcf7e` |
| API / `anthropic-messages` | `c428cd73-4eee-4fe3-ab92-25ad9c80f734` | `f7df02d0-b086-4455-a9a7-3b980c7951d9` | `8cae721e2b77e1918860396f08875e3493249744` |

Browser reports are `opencode-m4-receipt-final-browser-openai.json` and `opencode-m4-receipt-final-browser-anthropic.json`, with corresponding `verified-*` files. API is `opencode-m4-api-receipt-final.json`; all passed. `m4/tool-model` keeps its slash. Two nonempty replies reach idle; ending reaches Thread ended/public completed. Verifiers check bundle/JSONL size/digest, restored files, user Git identity, known credential/private-key markers, and container/volume removal. Both ended screenshots were inspected.

Login evidence is development login, not production external-identity acceptance. Live workload probes verified UID/GID 1000, no extra groups/capabilities, NoNewPrivs and no access to Node private keys/home. Guardian supervision remains independent; full production multiplayer isolation is not claimed.

## Merge compatibility and scope

Personal-model and prior-Revision inputs coexist. Published restore wire field 6 stays intact; model reference uses 7. Controller gates both capabilities independently; wire round-trip and missing-either-capability regressions passed. Both `0033` migrations retain complete filenames/SQL without rewriting applied ledgers. Final Cloud full race CI passed 472 tests, frontend 945, and Desktop three-platform/frontend/Tauri/E2E checks all passed. See [Cluster evidence](https://github.com/wanglongan587/cluster/blob/codex/m4-opencode-model-proxy/docs/opencode-m4-acceptance.md).

| Obligation | Direct evidence | Status |
| --- | --- | --- |
| Both page configuration/clone/Issue/multiturn/end paths | Final browser reports, production CLI, controlled model | Covered |
| Exact browser Run artifacts/Git/container and volume release | Final corresponding `verified-*` reports | Covered |
| Both API protocols and slash model IDs | `opencode-m4-api-receipt-final.json` | Covered |
| Bluezone with fresh page-entered Keys and real models | User configuration missing; controlled success is insufficient | Missing |
| Multi-identity browser permissions/concurrency/disable/revoke | Lower-layer rules tested; browser paths cover only the initiator | Partial |
| Full-stack authorization failure/crash/recovery | Scope/renewal/revoke/cancel tests do not replace fault drills | Partial |
| Master-key disaster recovery | Encryption/startup-denial tested; no full recovery drill | Partial |
| Unstarted empty JSONL full delivery/release | Node ending tests, no direct complete-delivery evidence | Partial |

Overall M4 stays Partial and personal-model ADRs approved until new page-entered Bluezone Keys pass one real model per protocol. Historic success, interruption recovery, startup failures and cumulative timeout reports remain distinct from final-source evidence.
