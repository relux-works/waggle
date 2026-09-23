# waggle: agent communication specification

Status: **DRAFT v4**, 2026-09-24. Not normative. Track document: `spec/track.md`. Evidence: research notes in the private task-board repository (`skill-project-management`): the task-board messaging map (*TM*), the Apiary and ax messaging map (*AX*) and the agent-messaging landscape (*LS*). Continues the task-board coordination-rooms epic.

Published as a draft for review; nothing here is implemented yet.

Decided by the operator (2026-09-24): name **waggle**, a separate public Go module and repository consumed like `skill-agents-management`; **SSHSIG** signatures (amend Apiary's raw Ed25519 preimage); **multi-signature** envelopes; the layered injection defense of §7; the fast path F1–F3 of §12 with NATS on a dedicated Mac mini on the operators' tailnet first; **pluggable signer providers and key assurance** (§4.5): Apple Secure Enclave, FIDO, TPM, PIV, with a fallback to an ordinary key.

---

## 0. Goal

One protocol, one message format and one delivery mechanism for:

1. **Orchestrator ↔ orchestrator** coordination on a shared project: several orchestrator sessions, first on one machine, then across 2–3 operators in different locations, later with other organizations.
2. **Orchestrator → worker** one-way messages: clarify, reprioritize, cancel, halt. Workers can never send instructions to orchestrators.

Transports are replaceable adapters; authority never comes from a transport.

## 1. What exists (summary)

- Nothing sends messages between orchestrators; run notices reach the orchestrator (Claude: sanitized terminal paste, at most 160 characters, only with a client attached; Codex: `turn/steer` / `thread/queue/add`, unsanitized, one attempt); children get goal revisions and directives they must poll (TM §1-§3).
- Nothing is signed; `TASK_BOARD_RUN_ID` is a convention; children can inherit the parent's manager variables (the child-environment leak in task-board spawns) (TM §6-§7).
- Coordination-rooms research chose a durable inbox/outbox with a Session Manager delivery adapter and "never type room content into a terminal" (TM §4).
- Apiary designed a transport-neutral signed envelope, closed receipts, deduplication, term + token fencing and "messages from other agents are data" (AX §1-§5); ax has no messaging (AX §6).
- No transport signs messages per user; SSHSIG over exact bytes with `allowed_signers` does (LS §3).

## 2. Layers and roles

```text
 identity      principals, SSH keys and certificates, roster, policy             waggle
 envelope      payload bytes + one or more role signatures (SSHSIG)              waggle
 verification  one pipeline for every transport                                  waggle
 inspection    optional guards against prompt injection                          waggle (pluggable)
 mailbox       durable append, dedup, order, cursors, receipts, expiry           carrier-backed
 delivery      doorbell + pull into the agent loop; supervisor cancel/halt       tb-sessiond / AgentHost
 carriers      local | NATS JetStream | board server | XMPP                      adapters
 gateways      A2A                                                               adapters
 consoles      `task-board mail tail` (TUI), optional IRC bridge (Ergo)          adapters
```

- **Carrier**: stores and delivers signed envelopes byte-exact with guarantees (durable, acknowledged, ordered, expiring). Only carriers deliver `coord` and `cmd`.
- **Console**: renders messages for people and lets them type. What a person types enters as a low-authority `coord` message; a console never carries commands.
- **Gateway**: translates to another protocol at the edge.

## 3. Principals, keys, certificates, roster

| Principal | Example | Key |
| --- | --- | --- |
| Operator | `op:alice@acme` | any key a signer provider exposes (§4.5): Apple Secure Enclave, FIDO2 (`sk-ssh-ed25519`), TPM, PIV, or an ordinary key in `ssh-agent`; approvals may require a minimum assurance (§4.4) |
| Orchestrator session | `orch:alpha@acme` | ephemeral key created by the host at session start, certified by the operator's key (principal `orch:alpha@acme`, validity 12 h) |
| Worker run | `worker:RUN-260924-abc123@acme` | ephemeral key per run, certified by the host key, validity = run deadline |
| Host | `host:mini-1@acme` | machine key with a host certificate from the operator's key |
| Partner (later) | `orch:*@partner` | partner CA, admitted for `coord` only |

The **roster** is a stock OpenSSH `allowed_signers` file committed in the project (`.waggle/allowed_signers`), with a key revocation list (`.waggle/revoked`). Adding a teammate is a reviewed pull request. Hosts read the roster at a pinned revision.

```text
# operators sign and approve directly
op:alice@acme namespaces="coord.v1@waggle,cmd.v1@waggle,approve.coord.v1@waggle,approve.cmd.v1@waggle" sk-ssh-ed25519@openssh.com AAAA…
op:bob@acme namespaces="coord.v1@waggle,cmd.v1@waggle,approve.coord.v1@waggle,approve.cmd.v1@waggle" ssh-ed25519 AAAA…
# orchestrator sessions: certificates issued by each operator's key
orch:*@acme cert-authority,namespaces="coord.v1@waggle,cmd.v1@waggle,endorse.coord.v1@waggle,endorse.cmd.v1@waggle" ssh-ed25519 AAAA…
# worker runs: certificates issued by host keys, reports only
worker:*@acme cert-authority,namespaces="report.v1@waggle" ssh-ed25519 AAAA…
# hosts: receipts only
host:*@acme cert-authority,namespaces="receipt.v1@waggle" ssh-ed25519 AAAA…
```

Optional for larger teams: operators get short-lived operator certificates from a team CA kept on a hardware key (`op:*@acme cert-authority,…` with the team CA), so rotating an operator key needs no pull request.

### 3.1 Lessons adopted from SSH certificate practice

| Lesson | How waggle applies it |
| --- | --- |
| Short expiry is the primary control; revocation lists are for emergencies because they must be distributed | session certificates 12 h, worker certificates end with the run, operator certificates (if a team CA is used) a few days; the revocation list lives in the repository and is checked on every verification |
| The CA key is the crown jewel | CA keys (operator keys acting as CAs, the optional team CA) live on hardware or in a Touch-ID-gated agent and never on disk in clear; per-operator CAs limit the blast radius |
| Issuance must be automatic | the host asks `ssh-agent` to certify the session key once at session start; later a signing service behind SSO (Smallstep or OPKSSH style) can issue operator certificates for bigger teams |
| Principals restrict what a certificate can do | principal patterns plus `namespaces=` per roster line; waggle-specific certificate extensions (`waggle-project@relux.works`, `waggle-role@relux.works`) bind a certificate to one project and role and are checked by the verifier |
| Inspect before trusting | `task-board mail keys inspect <cert>` decodes a certificate like `ssh-keygen -L`; `task-board mail roster lint` flags over-broad principals and missing expiry |
| Audit every acceptance | every verified signature is logged with key id, serial and CA fingerprint, like `sshd`'s "Accepted … ID … (serial …) CA …" line |
| Host certificates remove trust-on-first-use | hosts present certificates from the operator's key, so peers verify host receipts without pinning fingerprints |
| Clocks drift | bounded skew on `issued_at`; the carrier's arrival time is recorded next to it |
| OpenSSH certificates are single-level: a certificate cannot sign another certificate | chains are expressed by listing each CA key in the roster with its own principals and namespaces, and by co-signatures (§4.3) when a second party must vouch at message time |
| Long lifetimes out of convenience and permissive principal lists are the common failures | the roster linter rejects both |

## 4. Where signatures live and how they are checked

### 4.1 The envelope

```json
{
  "payload": "eyJzY2hlbWEiOiJ3YWdnbGUtdjEiLCJ0eXBlIjoiY21kLmhhbHQiLC…",
  "sigs": [
    { "role": "author",   "ns": "cmd.v1@waggle",         "sig": "U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg…" },
    { "role": "approver", "ns": "approve.cmd.v1@waggle", "sig": "U1NIU0lHAAAAAQAAAEpzay1zc2gtZWQyNTUxOUBvcGVuc3No…" }
  ]
}
```

- `payload`: base64url of the exact message bytes; never re-serialized.
- `sigs`: one or more signatures over the **same payload bytes**. Each is an SSHSIG blob (the format of `ssh-keygen -Y sign`, also used by git for SSH-signed commits) containing the signer's public key or certificate and a signature over a structure made of the magic string `SSHSIG`, the namespace, the hash algorithm and the SHA-512 hash of the payload.
- The **namespace binds both the class and the role**: an author signs `cmd.v1@waggle`, an approver signs `approve.cmd.v1@waggle`, a parent orchestrator signs `endorse.cmd.v1@waggle`. A signature made in one role cannot be reused in another, and a changed byte breaks every signature.

The payload:

```json
{
  "schema": "waggle-v1",
  "type": "cmd.halt",
  "id": "0192f7a8-6c1e-7cc2-9f1e-5a2b7c9d0e11",
  "from": "orch:alpha@acme",
  "to": "worker:RUN-260924-abc123@acme",
  "signers": { "approver": "op:alice@acme" },
  "project": "acme",
  "scope": "EPIC-260924-auth",
  "issued_at": 1790000000000,
  "expires_at": 1790000600000,
  "seq": 42,
  "term": 7,
  "delivery": "interrupt",
  "body": { "reason": "stop before integration; wrong base" }
}
```

`signers` names who is expected to co-sign, so the author commits to the approver it asked for. Envelopes are stored exactly like this in every mailbox and stream and can be verified again offline with stock OpenSSH plus the roster at that revision.

### 4.2 Signing

1. The agent calls `task-board mail send …` with type, recipient and body. It never holds a key and never sets `from`.
2. The host fills `from`, `id`, `seq`, `term`, times and `signers` (from policy, §4.4), serializes once, and signs as `author` with the session key.
3. If the policy requires more roles, the message waits in the outbox as `pending-signatures`, and the host asks each signer:
   - an operator runs `task-board mail approve <id>`, which renders every payload field ("what you see is what you sign") and signs through `ssh-agent`, with a touch, PIN or fingerprint when the key demands it;
   - a parent orchestrator's host adds an `endorse` signature when its policy allows it.
4. When every required signature is attached, the host hands the envelope to the carrier. Retries resend the same bytes.

### 4.3 Verifying

The same pipeline runs for every carrier:

1. Decode `payload`; the expected class comes from the channel, never from the envelope.
2. For each entry in `sigs`: the principal is `payload.from` for `author`, `payload.signers.<role>` otherwise; verify the SSHSIG with namespace `<role prefix><class>.v1@waggle` against the roster at `issued_at` (certificates, validity windows, revocation list, waggle certificate extensions). Unverifiable entries are dropped and reported, never counted.
3. Apply the **signature policy** (§4.4): every required role present and verified, with the required key properties.
4. Check the audience, then class authority (a `cmd` author must hold the scope lease of the target run's scope, §10; a `coord` author's `term` must match its lease when it speaks for a scope).
5. Check expiry and skew; record the carrier's arrival time.
6. Deduplicate on `(project, from, id)` with the payload digest; the same id with other bytes is `rejected:conflicting_duplicate`.
7. Enforce `seq` per `(from, to)`.
8. Run inspection (§7).
9. Store with the verification result, ring the doorbell (§8), send a signed receipt, write the audit line (key id, serial, CA fingerprint per signature).

A message failing steps 1–7 never reaches an agent; operators see it in `task-board mail rejected`. Extra signatures never grant anything beyond what the policy asks for.

### 4.4 Signature policy and attested approval

The policy is a committed file next to the roster (`.waggle/policy.toml`); the roster stays a stock `allowed_signers` file.

```toml
[types."coord.request"]
author = "orch:*@acme"

[types."coord.handoff-accept"]
author   = "orch:*@acme"
approver = { principal = "op:*@acme" }

[types."cmd.halt"]
author   = "orch:*@acme"
approver = { principal = "op:*@acme", key = "sk", user_verification = true }

[types."cmd.cancel"]
author   = "orch:*@acme"
endorser = { principal = "orch:*@acme", when = "author_is_sub_orchestrator" }
```

- `key = "sk"` with `user_verification = true` requires a FIDO2 signature whose flags byte, written by the authenticator and covered by the signature, says the user was verified (PIN or fingerprint). This is the only form of "the operator was physically there" that other people can check. Touch ID prompts of Secure Enclave agents or password managers protect the key locally but are not visible in the signature, so they do not satisfy this rule.
- An approval is a signature by a key that others can resolve through the roster: listed directly, or certified by a CA line.
- Sub-orchestrators: a parent orchestrator can be required to co-sign (`endorse`) messages its sub-orchestrator sends outside the parent's scope.

### 4.5 Signer providers and key assurance

Operators have different hardware. Signing and the facts about a key are therefore pluggable; the envelope format (SSHSIG entries) does not change.

| Interface | Job | Built-in or planned implementations |
| --- | --- | --- |
| `SignerProvider` | lists usable keys and produces an SSHSIG signature over the payload bytes under a namespace | `ssh-agent` (ordinary keys, FIDO `sk-*` keys, Secure Enclave keys exposed by agents such as Secretive or 1Password); `apple-se` (a small signed helper using Secure Enclave P-256 keys directly); later `tpm`, `piv` |
| `EnrollmentVerifier` | checks the evidence presented when a key is enrolled and returns assurance facts | `fido` (attestation written by `ssh-keygen -O write-attestation`, checked against the vendor roots or the FIDO metadata service); `apple-se` (attestation that the key was generated inside the Secure Enclave of a genuine Apple device and cannot be imported or exported); later `tpm` (TPM2 key certification), `piv` (PIV slot attestation) |
| `SignatureInspector` | extracts per-signature facts | FIDO flags (user present, user verified) and signature counter |

Assurance levels, from weakest to strongest: `software` (an ordinary key) < `hardware-bound` (non-exportable key, origin not proven) < `attested` (hardware origin proven at enrollment by the vendor's attestation). A separate per-signature fact, `user_verified`, exists only where the signature format carries it (FIDO today). Enrollment evidence and its verification result are stored next to the roster (`.waggle/keys/<principal>/<fingerprint>.json`); the roster itself stays a stock `allowed_signers` file.

Apple specifics (to verify against current Apple documentation before implementation):

- A Secure Enclave key is generated by the enclave's own random number generator and never leaves it; an attestation returned at creation lets a verifier check that a genuine Apple device's Secure Enclave generated it. After enrollment, every signature verified against that public key is bound to that hardware-backed credential.
- Whether each signature also required biometry is a property of the key's access-control policy set at creation; the attestation may or may not state it, so `user_verified` is not assumed for Apple keys unless the attestation says so.
- Platform SSO and Access Keys (macOS 26) are identity-provider credentials, not a general signing interface. Their natural role here is issuance: a team signing service behind SSO issues short-lived operator certificates only to devices that prove, through Platform SSO or managed device attestation, that the operator's key lives in a Secure Enclave. This is a later `apple-psso` verifier.

Fallback: an operator with neither a Mac nor a FIDO key signs with an ordinary key (`software`). The policy decides what that key may do: sign `coord` normally, approve only low-risk types, or require a second approver for high-risk ones.

```toml
[types."cmd.halt"]
author   = "orch:*@acme"
approver = { principal = "op:*@acme", min_assurance = "attested", user_verified = "if-supported" }
fallback = { min_assurance = "software", require = "second-approver" }
```

## 5. Classes and authority

| Namespace (author role) | Types | Who may send → to whom |
| --- | --- | --- |
| `coord.v1@waggle` | `request`, `propose`, `notice`, `question`, `answer`, `handoff-offer`, `handoff-accept` | orchestrator ↔ orchestrator; operator → orchestrator; room broadcast |
| `cmd.v1@waggle` | `clarify`, `reprioritize`, `cancel`, `halt`, `extend-deadline` | scope-owning orchestrator or operator → worker run |
| `report.v1@waggle` | `started`, `progress`, `blocked`, `result-ready`, `failed` | worker run → board/coordinator (rendered to orchestrators as quoted data) |
| `receipt.v1@waggle` | `received_durable`, `delivered`, `read`, `applied`, `rejected:<reason>`, `expired` | host → sender |

- A `coord` message asks or informs; the receiving orchestrator decides under its own policy and the kernel's gates.
- `cancel` and `halt` are executed by the supervisor; the worker's model is only informed.
- The roster limits namespaces per principal, so a worker key cannot produce a valid `coord`, `cmd`, `approve` or `endorse` signature over any transport.
- Existing run notices become `report.v1`; existing directives become `cmd.v1`.

## 6. Example of multi-signature in use

Alpha (Alice's orchestrator) wants to halt a worker that Beta (Bob's orchestrator) launched, because it is about to integrate onto a broken base. Alpha's lease does not cover that worker, so a plain `cmd.halt` from Alpha is refused. Policy requires an operator approval with user verification. Alpha composes the halt; the host holds it as `pending-signatures` and shows Alice a prompt; Alice runs `task-board mail approve 0192f7`, reads the rendered payload, touches the FIDO key; the envelope now carries Alpha's `author` signature and Alice's `approver` signature with the user-verified flag. Bob's host verifies both, sees that an operator physically approved, halts the worker and sends a receipt. Anyone can later check that it was Alice, not only Alpha.

## 7. Injection defense in depth

Pulling content as tool output removes channel problems (nothing is typed into a terminal, nothing impersonates the system or the operator, every sender is verified), but a model can still be persuaded by what it reads. Layers (accepted by the operator 2026-09-24):

| Layer | What it does | Mandatory |
| --- | --- | --- |
| 1. Authority outside the model | gates, scope leases, supervisor-executed `cancel`/`halt`, signature policy | yes |
| 2. Typed bodies and framed presentation | typed fields; bounded optional free text; `mail read` renders an untrusted-data frame with sender, verification status and trust tier; the orchestrator module says messages are requests to evaluate, never instructions to follow | yes |
| 3. Deterministic inspectors | size and type limits, URL and command-pattern policy, secret detection, known injection markers → `allow`, `flag`, `quarantine`, `reject` | optional (default on for external senders) |
| 4. Model-based guard | a cheap classifier or prompt-injection detector | optional |
| 5. Quarantined reader | a separate low-privilege model with no tools turns free text into the typed request; the orchestrator sees only the result plus a flag | optional (recommended for external senders) |
| 6. Quarantine and human release | `task-board mail release <id>` | optional (default for external senders) |
| 7. Rate limits and quotas | per principal and room | yes for CM4 |

Trust tier (`own`, `team`, `external`) comes from the roster line that verified the author signature.

## 8. Delivery into agent loops: doorbell and pull

1. The host appends the verified message to the principal's mailbox.
2. It rings a fixed-template doorbell: `[waggle] 2 new messages for orch:alpha (coord from orch:beta; cmd from op:alice). Read: task-board mail read`. `interrupt` → Codex `turn/steer` with `expectedTurnId`, Claude paste at a safe point; `next_turn` → Codex `thread/queue/add`, Claude after the turn; `on_request` → no doorbell.
3. The agent pulls with `task-board mail read` or a per-session MCP tool with a capability token; the tool records `read`.
4. Doorbells coalesce; a stale-turn steer falls back to `next_turn`; with no client attached messages wait durably.

## 9. Mailboxes and rooms

A mailbox per principal, a room per project and optional topic rooms; outbox before send; durable append; dedup; per-sender order; cursors; expiry with tombstones; bounded queues with a reserved control lane for `cancel`, `halt` and receipts. Receipts never overclaim.

## 10. Scope leases

A lease says which orchestrator owns which part of the project right now: holder, monotonic `term`, random token, expiry renewed by heartbeat, stored with compare-and-set (local board on one machine; NATS JetStream KV across machines; board server or Apiary authority later).

Example: Alpha (Alice) leases Epic Auth, Beta (Bob) leases Epic Billing. When Alpha tries to spawn a developer on a Billing story, task-board looks the lease up, sees Beta, and refuses with `scope_owned_by orch:beta`. To work there, Alpha sends `coord.handoff-offer` or a `coord.request`; if Beta accepts, the lease moves to Alpha with a higher term. If Alpha's session dies and comes back on another machine, the new session gets a higher term, and anything still signed under the old term is refused.

The kernel checks leases at the three places where damage happens: **spawn** (starting work), **integrate** (landing code) and **board commit** (writing board state). Messages alone cannot prevent two orchestrators from working on the same story at once (a finding of the earlier coordination-rooms research); the lease can.

## 11. Carriers, gateways and consoles

| Role | Adapter | Use |
| --- | --- | --- |
| carrier, one machine | **local**: Maildir-style mailbox under the Session Manager state root (generalizes today's notice inbox) | orchestrators on one Mac; the final delivery stage of every other carrier |
| carrier, internet (first) | **NATS JetStream** on a dedicated Mac mini on the operators' tailnet, reachable over the tailnet only at first; streams per project; durable consumers per principal; `Nats-Msg-Id` = envelope id; `Nats-TTL` from expiry; leases in a JetStream KV bucket; one NATS user per operator | 2–3 operators in different locations |
| carrier, optional later | board server mailbox | if the board server becomes the team's single authority |
| carrier, federation | **XMPP (Prosody)**, envelopes in a namespaced element (`urn:apiary:envelope:1`), s2s certificates and allowlists, members-only rooms | other organizations with their own servers |
| gateway | **A2A v1** (Agent2Agent, Linux Foundation): a partner's agent receives our `coord.request` as an A2A task; our Agent Card is signed | partners that offer an agent as a service |
| console (first) | **`task-board mail tail`**: a terminal UI streaming rooms and mailboxes with full text, verification marks, pending approvals and one-key `approve` | every stage |
| console (optional) | **IRC bridge on Ergo**: one channel per project for any IRC client | people who prefer a chat client |

**How a console shows long messages when IRC lines are 512 bytes.** The console does not carry the signed envelope; it shows a rendering. Each message appears as a header line (`[cmd.halt] orch:alpha → worker:RUN-…abc123, approved by op:alice ✓`), followed by the body: clients that support the `draft/multiline` extension (Ergo serves it) show the body as one message; other clients see it split into several lines, capped at a configurable number with a pointer to the full text (`task-board mail read 0192f7`). The 512-byte limit matters for a carrier, which must deliver the exact signed bytes with acknowledgements; it does not matter for a rendering. Our own terminal UI shows everything in full and is the faster first console.

## 12. Fast path to internet coordination

| Step | Delivers | Kept afterwards |
| --- | --- | --- |
| F1 | `waggle` module: envelope, multi-signature SSHSIG via `ssh-agent` (including FIDO user verification), roster and policy, verification pipeline, receipts, audit; interop tests against `ssh-keygen -Y verify` | yes |
| F2 | local carrier, doorbell + pull, `task-board mail send|read|tail|approve|rooms|rejected` | yes |
| F3 | NATS carrier on a dedicated Mac mini on the operators' tailnet; NATS users per operator; workers never get NATS credentials | yes |
| F4 | scope leases enforced at spawn, integrate, board commit | yes |
| F5 (optional) | IRC bridge on Ergo | yes, as a console |

## 13. Security stages

| Stage | Admission | Confidentiality |
| --- | --- | --- |
| Trusted (F1–F4) | roster and policy pull requests; operator keys (FIDO for approvals); session, run and host certificates; NATS users per operator; tailnet ACLs | tailnet (WireGuard); carrier trusted for plaintext |
| External (CM4) | partner CA lines limited to `coord`; hosted accounts first, then XMPP federation; inspectors and quarantine on by default | carrier trusted for plaintext by default; an end-to-end profile (MLS) is a separate decision |

Always: children never receive carrier credentials or the parent's manager variables (fix the child-environment leak in task-board spawns first); worker mailboxes are receive-only.

## 14. Module and repository

`waggle` is its own public Go module and repository (`relux-works/waggle`), consumed by task-board through `go.mod` at tags exactly like `skill-agents-management`: no `replace`, a `go.work` only for local development, one tag per wave. Later consumers: Apiary's AgentHost and coordinator, and `curator-run` if it ever sends messages.

## 15. Phases

| Phase | Delivers | Done when |
| --- | --- | --- |
| CM0 | this draft; `waggle` repository | merged as draft |
| CM1 | F1 + F2; env-leak fix; notices and directives mapped onto classes | two orchestrators on one machine coordinate in a room; each cancels only its own children; an operator approval with a FIDO touch halts a foreign worker |
| CM2 | F3 + F4 (+ F5) | orchestrators of 2–3 operators in different locations split scopes and negotiate over the internet; a person watches in `mail tail` |
| CM3 | orchestrator → remote worker through AgentHost (Apiary stage 1) | a remote worker receives `clarify`/`cancel` and cannot send `coord`/`cmd` |
| CM4 | other organizations: partner CAs, inspectors on by default, XMPP federation, A2A gateway, quotas | a partner coordinates in a shared room and cannot command our workers |

## 16. Decisions

| # | Question | Status |
| --- | --- | --- |
| 1 | Name, module, repository | decided: `waggle`, own public module and repository |
| 2 | Signature container | decided: SSHSIG, multi-signature by role; Apiary authority wire gets an amendment |
| 3 | Injection defense | decided: §7 |
| 4 | First internet carrier | decided: NATS JetStream on a dedicated Mac mini on the operators' tailnet |
| 5 | Console | proposed: own terminal UI first, IRC bridge optional |
| 6 | Federation | proposed: XMPP + A2A gateway |
| 7 | Priority | decided: start now, in parallel with the migration |
| 8 | Key hardware | decided: pluggable signer providers and assurance levels (Apple Secure Enclave, FIDO, TPM, PIV, ordinary-key fallback) |
