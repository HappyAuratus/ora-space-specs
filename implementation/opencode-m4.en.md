# M4 OpenCode: browser and API acceptance against controlled upstreams

English | [中文](opencode-m4.md)

On 2026-10-10, the independent M4 Compose stack passed the normal browser and API paths for
`openai-completions` and `anthropic-messages`. The production CLI from the pinned official
`official/ora-space.opencode@0.6.4` package used the platform proxy, read and modified the checkout,
and delivered a Revision. Every record below reports `mode=fixture` and `status=passed`, with exit code 0.
The user has not entered a new Bluezone Key through the page. Overall M4 acceptance remains `Partial`,
and the related ADRs remain `approved`.

The decisions and core obligations are tracked in [personal model access](../test-cases/cloud/model-access/personal-model-access.md),
[Thread input permissions](../test-cases/cloud/thread/personal-model-input.md),
[Node model sessions](../test-cases/node/agent-runtime/model-proxy.md), and
[model identities and deployment](../test-cases/cluster/sandbox-server/model-client-identity.md).

## Direct evidence

The files are retained under cluster's `.acceptance/`. This record includes safe status, protocol and
resource identifiers only; it does not copy credentials, response bodies or screenshot contents.
Each browser record and its subsequent verification address the same Run. API acceptance is additional
evidence and does not replace page interactions.

| Entry and protocol | Browser record and verification of its exact resources | Run | Revision |
|---|---|---|---|
| Browser `openai-completions` | `opencode-m4-browser.json`, `opencode-m4-browser-verification.json` | `e464a25d-f31f-4770-ada9-b00f32b0ad34` | `69dc202d-1bb2-472f-9672-04f4d0deb6aa` |
| Browser `anthropic-messages` | `opencode-m4-browser-anthropic.json`, `verified-opencode-m4-browser-anthropic.json` | `c696ad9b-3159-4cdc-a3c1-f99d8731ffa3` | `3204dcbb-1c64-40d6-9672-975374bf2b9c` |
| API `openai-completions` | The matching scenario in `opencode-m4-final.json` | `81c988f3-9ea6-48cb-883b-5e2b0c160bcb` | `365d9a75-de7e-4630-a28f-c49034a0ecfb` |
| API `anthropic-messages` | The matching scenario in `opencode-m4-final.json` | `c4f74e77-b648-4eeb-b53f-a685f6b9e65b` | `61da1046-390d-4a80-a3b4-4d7e27ce4300` |

Both protocols retain model ID `m4/tool-model`. Its slash passes unchanged through the personal default,
run binding and actual CLI request. The browser runs retain 10 and 9 checkpoint screenshots respectively;
these counts provide traceability and do not replace assertions or stored-object verification.

## Verified normal path

The browser signs in through the development provider, creates a collaboration space, fills in the model
connection and default, saves a Git author identity, selects the official plugin, creates a project and
clones GitHub's `main` branch, creates its associated Issue, mentions OpenCode, sends a follow-up turn and
ends the session. The password input clears after saving. Both turns return non-empty replies and reach
idle. The page then displays a saved changed Revision and the public run becomes completed.
This login evidence does not claim production acceptance of an external identity provider.

Subsequent verification reads the exact browser-created Run, checks bundle and JSONL object digests,
restores the result file, checks the Git author, checks for credential material in the artifacts, and
confirms run `done` plus removal of the run Workspace container and volume. The browser run Workspaces are
`e83c5481-a6d0-493e-bc5f-49339b989acc` and `e84ce9d5-0d2b-4c92-85e8-fcdc53084a21`.
The verification does not create a replacement run to obtain a successful result.

Final API acceptance separately verifies both protocols, production CLI repository tools, multiple turns,
ending, Revision delivery, Git identity and cleanup. The model service is controlled; the execution side
uses the pinned official CLI rather than Echo or an Agent simulator. Acceptance may serve the original
package through an HTTPS cache with normal CA verification and the exact official SHA-256, without changing
the package bytes or digest.

## Evidence status and remaining acceptance

| Obligation | Current direct evidence | Status |
|---|---|---|
| Page configuration, clone, Issue, multiple turns and ending for both protocols | Two real browser records, pinned production CLI, controlled upstream | Covered |
| Revision, Git author, artifact restoration and container/volume removal for browser-created runs | Verification of the exact Run in both browser records | Covered |
| Normal API paths for both protocols and a model ID containing a slash | `opencode-m4-final.json`, model ID `m4/tool-model` | Covered |
| Both Bluezone protocols with a new Key entered through the page | Not executed; fixture success cannot establish it | Missing |
| Browser permissions for multiple identities, concurrency, disablement and revocation | Core, HTTP and frontend rule tests exist; these browser records cover the initiating user's normal path | Partial |
| Authorization failure, process crashes and recovery across the stack | Lower-level scope, revocation, renewal and cancellation tests exist; the normal path does not prove recovery | Partial |
| Encryption master-volume disaster recovery | Encryption identity and startup refusal tests exist; a complete recovery exercise has not run | Partial |
| Upload, registration and release of an empty JSONL Revision when the initial grant already sees ending | Node ending coordination and empty-history sealing are tested; complete delivery has no direct evidence yet | Partial |

M4 can be recorded as complete only after a new Bluezone Key is configured through the page and both
protocols pass acceptance against the actual service. Agent privilege reduction and guardian ownership
remain separate work; this record does not claim completed production execution isolation for multiple users.
