# M4 OpenCode: controlled evidence and review repair boundaries

[中文](opencode-m4.md) | English

The following preserves historical 2026-10-10 results for `Cloud 5b6b5ec` / `Desktop f8536f9d` / `Cluster a8146c7`; those successes do not establish the subsequently reviewed failure paths or project configuration isolation. The production CLI in fixed `official/ora-space.opencode@0.6.4` performed LICENSE reads, file writes and Git commits through the platform proxy. Controlled upstream evidence does not establish real-provider acceptance.

## 2026-10-11 review follow-up

Review confirmed missing trusted DNS deployment behavior when system DNS returns reserved fake IPs, an omitted IPv6 benchmark range, insufficient safe transport classification, clone sharing the short local-command deadline with unbounded retries, project `.opencode` discovery, and ACP failures represented as completed turns. The subsequent HOME and public CA fixes require fresh-session verification.

Repairs add authenticated HTTPS DNS with whole-answer public-address checking, finite safe error classifications, a bounded network Git deadline, three durable clone attempts, and closed failed sessions with a persisted safe failure entry. Existing successful multiturn behavior remains; failed sessions attempt to deliver partial Revisions and release authorization. A committed user cancellation wins a racing model failure. Project configuration discovery is disabled without changing tracked `.opencode` files.

Per-comment reproductions, checks, commits and original-thread replies are recorded in the [review ledger](https://github.com/wanglongan587/cluster/blob/codex/m4-opencode-model-proxy/docs/m4-review-follow-up.en.md). One OpenAI response after a temporary hosts mapping is insufficient evidence for DNS, Anthropic, failure UI, cleanup or new-key acceptance. Overall M4 remains Partial and real-provider acceptance Missing.

## 2026-10-11 direct review-fix evidence

The independent `ora-m4-review` stack uses this round's Cloud/Node code, the fixed official 0.6.4
package and controlled HTTPS models. Its repository has tracked `.opencode/package.json` and a
sentinel; isolation is tested without deleting user files. `browser-final.json` and exact-Run
`verified-browser.json` passed development login, private connection setup, clone, two read/write
turns, ending, Revision restoration and container/volume release. Restored diffs contain only
`ora-m4-result.txt`; tracked `.opencode` stays unchanged and bundle/Node JSONL size/digests match.

| Protocol | This round's Run | Revision | Final Git commit |
| --- | --- | --- | --- |
| `openai-completions` | `3890efc9-63c2-48ff-bb04-c7efec7efb19` | `67233399-9f15-4248-8c9a-99c2c4cd8f3d` | `3348203ef5c71d58d0153bff14bf0b07f4fb23da` |
| `anthropic-messages` | `92efb86b-df38-40ff-abe7-c13a33f30aeb` | `f59f0c28-ce88-48d5-8b68-d17f8be48227` | `6e9ac4d9b39b593662cd1cb8eb865408cf79771f` |

`failure-browser.json` and `verified-failure.json` passed. A controlled provider rejects a synthetic
credential with HTTP 401; safe Gateway 502 and actual CLI ACP failure end the Node session. The page
shows a safe failure entry and disables appending; Thread is ended, public run failed and an unchanged
Revision registered. OpenAI Run `022547f5-d58f-407f-b870-a3fc4563c90e` and Anthropic Run
`09c01527-8ba4-4dea-b680-b830a60f6f14` have zero active grants, matching Node JSONL digests and removed
run containers/volumes. PostgreSQL/gRPC and Node plugin tests separately establish unknown-detail
sanitization, replay deduplication and committed-cancellation priority.

Actual Controller/Node clone operation `e56b5445-d95f-4121-b369-17c641df652b` stopped after exactly
three durable dispatches: failed, no retryAt, admission closed and manual retry rejected with 409;
the primary Workspace remains. A slow actual Node clone succeeds beyond the unchanged five-second
local-command budget. A network timeout still goes through guardian cleanup; Unknown is not retried.

Independent Echo `echo-browser.json` and `echo-resume-final.json` passed M2 multiturn/ending and M3's
three-run first output, restored unchanged and further-write paths, with object digests, Git author
and resource cleanup verified. Credential-free Windows/WSL HTTPS DNS probes verified a real upstream
certificate and received HTTP 401; this establishes connectivity only. No Pi Key was read this round;
Bluezone fresh-Key acceptance for both protocols remains Missing.

A first cold startup under concurrent heavy load exceeded the existing 30-second ready budget and
left AgentUnavailable; later fresh sessions passed. The ready budget, certificate verification and
hosts configuration were not relaxed. Full delivery of unstarted empty JSONL remains Partial and
complete descendant supervision stays tracked by Desktop #665. The review ledger owns current checks
and commit status.

## 2026-10-10 historical direct evidence

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

Personal-model and prior-Revision inputs coexist. Published restore wire field 6 stays intact; model reference uses 7. Controller gates both capabilities independently; wire round-trip and missing-either-capability regressions passed. Both `0033` migrations retain complete filenames/SQL without rewriting applied ledgers. The following is historical 2026-10-10 CI: Cloud full race passed 472 tests, frontend 945, and Desktop three-platform/frontend/Tauri/E2E checks passed. These are not this round's CI results. See [Cluster evidence](https://github.com/wanglongan587/cluster/blob/codex/m4-opencode-model-proxy/docs/opencode-m4-acceptance.md).

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
