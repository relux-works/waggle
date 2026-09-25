# waggle: agent communication specification

Status: **DRAFT v5.1**, 2026-09-24. Not normative. Track document: `spec/track.md`. Evidence: research notes in the private task-board repository (`skill-project-management`): the task-board messaging map (*TM*), the Apiary and ax messaging map (*AX*) and the agent-messaging landscape (*LS*). Continues the task-board coordination-rooms epic. v5.1 adds Amendment A1 (§19) with what board processes need: signed approvals, relaxations and human transitions on a board; signed external events from service principals; the hand-off result; user verification in board policy; delegation of board acts to orchestrator roles. The direction of A1 is accepted, and it is implemented with CM2. v5 splits the protocol into a communication layer and a coordination layer, and adds the conversation model and the provider interface, the protocol lab, delivery through the session-host module and the placement of the internet carrier; v4 of the same day added signer providers.

Published as a draft for review; nothing here is implemented yet.

Decided by the operator (2026-09-24): name **waggle**, a separate public Go module and repository consumed like `skill-agents-management`; **SSHSIG** signatures (amend Apiary's raw Ed25519 preimage); **multi-signature** envelopes; the layered injection defense of §7; the fast path F1–F3 of §14 with NATS on a dedicated Mac mini on the operators' tailnet first; **pluggable signer providers and key assurance** (§4.5): Apple Secure Enclave, FIDO, TPM, PIV, with a fallback to an ordinary key. Later the same day: **two layers**, communication and coordination (§2); a **conversation model and provider interface** in which providers are untrusted pipes, so proprietary chats are acceptable as adapters but never the core (§8, §9); IRC for a project that runs its own server, XMPP for federation; a **protocol lab** that feeds the coordination layer (§13); delivery into sessions through the **session-host module** (§10); NATS as a **separate tailnet-only service** that does not touch the host's shared ingress, and **no board-server mailbox** (§9.3).

---

## 0. Goal

One protocol, one message format and one delivery mechanism for:

1. **Orchestrator ↔ orchestrator** coordination on a shared project: several orchestrator sessions, first on one machine, then across 2–3 operators in different locations, later with other organizations.
2. **Orchestrator → worker** one-way messages: clarify, reprioritize, cancel, halt. Workers can never send instructions to orchestrators.

Transports are replaceable adapters; authority never comes from a transport. The protocol has two layers: the communication layer moves signed messages between principals, and the coordination layer, built on it, decides who works on what (§2).

## 1. What exists (summary)

- Nothing sends messages between orchestrators; run notices reach the orchestrator (Claude: sanitized terminal paste, at most 160 characters, only with a client attached; Codex: `turn/steer` / `thread/queue/add`, unsanitized, one attempt); children get goal revisions and directives they must poll (TM §1-§3).
- Nothing is signed; `TASK_BOARD_RUN_ID` is a convention; children can inherit the parent's manager variables (the child-environment leak in task-board spawns) (TM §6-§7).
- Coordination-rooms research chose a durable inbox/outbox with a Session Manager delivery adapter and "never type room content into a terminal" (TM §4).
- Apiary designed a transport-neutral signed envelope, closed receipts, deduplication, term + token fencing and "messages from other agents are data" (AX §1-§5); ax has no messaging (AX §6).
- No transport signs messages per user; SSHSIG over exact bytes with `allowed_signers` does (LS §3).

## 2. Two layers

```text
 coordination layer (§11–§13)
   decisions     proposals, support, quorum; decision records                                   waggle
   leases        scope ownership: holder, term, token, expiry (compare-and-set)                 waggle + lease store
   vocabulary    request, propose, claim, handoff, question and answer                          waggle
 ───────────────────────────────────────────────────────────────────────────────────────────────
 communication layer (§3–§10)
   identity      principals, SSH keys and certificates, roster, policy                          waggle
   envelope      payload bytes + one or more role signatures (SSHSIG)                           waggle
   verification  one pipeline for every provider                                                waggle
   inspection    optional guards against prompt injection                                       waggle (pluggable)
   conversation  spaces, threads, participants, messages                                        waggle
   mailbox       durable append, dedup, order, cursors, receipts, expiry                        provider-backed
   delivery      doorbell + pull into the agent loop; supervisor cancel/halt                    session host (agent-session-host) / AgentHost
   providers     local | NATS JetStream | IRC | XMPP | Matrix | Slack | Telegram | email        adapters (carriers, consoles)
   gateways      A2A                                                                            adapters
   consoles      `task-board mail tail` (TUI), optional IRC bridge (Ergo)                       adapters
```

- **The communication layer** answers "who said what to whom, who approved it, and did it arrive": principals and keys, the envelope and its signatures, verification, the conversation model, mailboxes, delivery into sessions, and the providers that carry bytes. It knows nothing about who owns which part of a project, except through one hook: step 4 of verification (§4.3) asks the coordination layer whether the author holds the authority a class requires.
- **The coordination layer** answers "who works on what, and how do we agree": negotiation, claiming work, scope leases, leader election and quorum decisions. It speaks only through the communication layer and never bypasses its verification; its authority lives in leases and gates, never in a model's reading of a message.
- The metaphor for the coordination layer is honeybee quorum sensing: when a swarm chooses a new home, scouts advertise candidate sites with waggle dances, support grows for the better sites, scouts inhibit dancers for competing sites, and the swarm commits once enough scouts gather at one site. waggle's decisions follow the same shape (§11).

## 3. Principals, keys, certificates, roster

| Principal | Example | Key |
| --- | --- | --- |
| Operator | `op:alice@acme` | any key a signer provider exposes (§4.5): Apple Secure Enclave, FIDO2 (`sk-ssh-ed25519`), TPM, PIV, or an ordinary key in `ssh-agent`; approvals may require a minimum assurance (§4.4) |
| Orchestrator session | `orch:alpha@acme` | ephemeral key created by the host at session start, certified by the operator's key (principal `orch:alpha@acme`, validity 12 h) |
| Worker run | `worker:RUN-260924-abc123@acme` | ephemeral key per run, certified by the host key, validity = run deadline |
| Host | `host:mini-1@acme` | machine key with a host certificate from the operator's key |
| Partner (later) | `orch:*@partner` | partner CA, admitted for `coord` only |

The **roster** is a stock OpenSSH `allowed_signers` file committed in the project (`.waggle/allowed_signers`), with a key revocation list (`.waggle/revoked`). Adding a teammate is a commit to the roster that a person confirms (Amendment A1; curator-trust `spec/trust.md` §3, §4); git hosting and pull requests are optional. A roster change takes effect only with a confirmation verified against the roster of the version in force, so no commit or batch admits its own signer, and a person's revocation takes effect at once (`spec/trust.md` §5). Hosts read the roster of the version in force, the pinned revision (process-configuration §2.4).

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

Optional for larger teams: operators get short-lived operator certificates from a team CA kept on a hardware key (`op:*@acme cert-authority,…` with the team CA), so rotating an operator key needs no roster commit.

Amendment A1 adds the service principal kind `svc:` for bridges that sign external events (§19.3), and admits host keys for hand-off results only (§19.4).

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

`signers` names who is expected to co-sign, so the author commits to the approver it asked for. Envelopes are stored exactly like this in every mailbox and stream and can be verified again offline with stock OpenSSH plus the roster at that revision. Messages posted to a space rather than to one principal also carry the conversation fields of §8.1.

### 4.2 Signing

1. The agent calls `task-board mail send …` with type, recipient and body. It never holds a key and never sets `from`.
2. The host fills `from`, `id`, `seq`, `term`, times and `signers` (from policy, §4.4), serializes once, and signs as `author` with the session key.
3. If the policy requires more roles, the message waits in the outbox as `pending-signatures`, and the host asks each signer:
   - an operator runs `task-board mail approve <id>`, which renders every payload field ("what you see is what you sign") and signs through `ssh-agent`, with a touch, PIN or fingerprint when the key demands it;
   - a parent orchestrator's host adds an `endorse` signature when its policy allows it.
4. When every required signature is attached, the host hands the envelope to the provider. Retries resend the same bytes.

### 4.3 Verifying

The same pipeline runs for every provider:

1. Decode `payload`; the expected class comes from the channel, never from the envelope.
2. For each entry in `sigs`: the principal is `payload.from` for `author`, `payload.signers.<role>` otherwise; verify the SSHSIG with namespace `<role prefix><class>.v1@waggle` against the roster at `issued_at` (certificates, validity windows, revocation list, waggle certificate extensions). Unverifiable entries are dropped and reported, never counted.
3. Apply the **signature policy** (§4.4): every required role present and verified, with the required key properties.
4. Check the audience and space membership (§8.3), then class authority through the coordination layer (a `cmd` author must hold the scope lease of the target run's scope, §12; a `coord` author's `term` must match its lease when it speaks for a scope).
5. Check expiry and skew; record the carrier's arrival time.
6. Deduplicate on `(project, from, id)` with the payload digest; the same id with other bytes is `rejected:conflicting_duplicate`.
7. Enforce `seq` per `(from, to)`.
8. Run inspection (§7).
9. Store with the verification result, ring the doorbell (§10), send a signed receipt, write the audit line (key id, serial, CA fingerprint per signature).

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

- `key = "sk"` with `user_verification = true` requires a FIDO2 signature whose flags byte, written by the authenticator and covered by the signature, says the user was verified (PIN or fingerprint). This is the only form of "the operator was physically there" that other people can check in the signature itself. Touch ID prompts of Secure Enclave agents or password managers protect the key locally but are not visible in the signature, so they do not satisfy this rule. Amendment A1 adds a second form for board policy: a signature from an enrolled operator companion app key, whose enrollment evidence proves that the key's own access control requires the person (§19.5).
- An approval is a signature by a key that others can resolve through the roster: listed directly, or certified by a CA line.
- Sub-orchestrators: a parent orchestrator can be required to co-sign (`endorse`) messages its sub-orchestrator sends outside the parent's scope.

### 4.5 Signer providers and key assurance

Operators have different hardware. Signing and the facts about a key are therefore pluggable; the envelope format (SSHSIG entries) does not change.

| Interface | Job | Built-in or planned implementations |
| --- | --- | --- |
| `SignerProvider` | lists usable keys and produces an SSHSIG signature over the payload bytes under a namespace | `ssh-agent` (ordinary keys, FIDO `sk-*` keys, Secure Enclave keys exposed by agents such as Secretive or 1Password); `apple-se` (a small signed helper using Secure Enclave P-256 keys directly); `companion` (the operator companion app on a phone, from Amendment A1, §19.5); later `tpm`, `piv` |
| `EnrollmentVerifier` | checks the evidence presented when a key is enrolled and returns assurance facts | `fido` (attestation written by `ssh-keygen -O write-attestation`, checked against the vendor roots or the FIDO metadata service); `apple-se` (attestation that the key was generated inside the Secure Enclave of a genuine Apple device and cannot be imported or exported); `companion` (the platform's attestation of the companion app's key, from Amendment A1, §19.5); later `tpm` (TPM2 key certification), `piv` (PIV slot attestation) |
| `SignatureInspector` | extracts per-signature facts | FIDO flags (user present, user verified) and signature counter |

Assurance levels, from weakest to strongest: `software` (an ordinary key) < `hardware-bound` (non-exportable key, origin not proven) < `attested` (hardware origin proven at enrollment by the vendor's attestation). A separate per-signature fact, `user_verified`, exists only where the signature format carries it (FIDO today); Amendment A1 adds enrolled companion keys, whose enrollment record states `user_verification = "app-enforced"` (§19.5). Enrollment evidence and its verification result are stored next to the roster (`.waggle/keys/<principal>/<fingerprint>.json`); every verifier re-checks the stored evidence against the vendor roots and never trusts a stored result alone (Amendment A1). The roster itself stays a stock `allowed_signers` file.

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
- Classes are communication-layer facts; what `coord` types mean, and which new ones exist, is the coordination layer's vocabulary (§11).
- Amendment A1 adds the `board` and `event` classes and the `coord.handoff-result` type (§19).

## 6. Example of multi-signature in use

Alpha (Alice's orchestrator) wants to halt a worker that Beta (Bob's orchestrator) launched, because it is about to integrate onto a broken base. Alpha's lease does not cover that worker, so a plain `cmd.halt` from Alpha is refused. Policy requires an operator approval with user verification. Alpha composes the halt. The host holds it as `pending-signatures` and shows Alice a prompt. Alice runs `task-board mail approve 0192f7`, reads the rendered payload and touches the FIDO key. The envelope now carries Alpha's `author` signature and Alice's `approver` signature with the user-verified flag. Bob's host verifies both, sees that an operator physically approved, halts the worker and sends a receipt. Anyone can later check that it was Alice, not only Alpha.

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

## 8. Conversation model

People and agents think in rooms, threads and replies; waggle models them once, and every provider maps them onto its own features (§9).

| Concept | Meaning | Where it lives |
| --- | --- | --- |
| **Space** | a room or channel: one per project, optional topic spaces, and a mailbox per principal for direct messages | `space` field of the signed payload; policy lists members |
| **Thread** | a conversation inside a space, identified by the id of its first message | `thread` field of the signed payload |
| **Participant** | a principal of the roster; a provider account is only a delivery address | roster + space membership in the policy |
| **Message** | one signed envelope; replies point at the message they answer | the envelope; `reply_to` field |

### 8.1 Conversation fields

A message posted to a space carries three more payload fields, inside the signed bytes:

```json
{
  "space": "acme/main",
  "thread": "0192f7a0-2b4c-7d11-8e2f-1a3b5c7d9e0f",
  "reply_to": "0192f7a9-4d6e-7f21-9a3b-2c4d6e8f0a1b"
}
```

`thread` is absent on the first message of a thread; `reply_to` is optional. Because both are signed, a provider cannot move a message to another space or thread, or detach a reply from its question, without the change being detected.

### 8.2 Mailboxes and rooms

A mailbox per principal, a room per project and optional topic rooms; outbox before send; durable append; dedup; per-sender order; cursors; expiry with tombstones; bounded queues with a reserved control lane for `cancel`, `halt` and receipts. Receipts never overclaim. Rooms are spaces; a mailbox is the space for messages addressed with `to`.

### 8.3 Membership

Which principals may post to and read a space is policy (`[spaces."acme/main"]` in `.waggle/policy.toml`), checked by the verifier on receipt (§4.3 step 4). Where a provider supports permissions, the host mirrors membership into them as defense in depth; the provider's permissions never grant anything the policy does not.

## 9. Providers

A provider is an adapter to something that moves bytes between machines or shows messages to people: a local directory, a message broker, a chat network, email. **Providers are untrusted pipes.** Authority comes from the signatures, the roster and the policy, never from the provider: a provider account proves nothing, a provider can drop, delay, reorder or duplicate messages, and the verification pipeline (§4.3) and receipts detect each of these. This is why proprietary chats such as Slack and Telegram are acceptable as adapters, and why none of them can be the core.

### 9.1 Interface

Illustrative; names are proposals until the Go module ships.

```go
type Provider interface {
    Capabilities() Capabilities
    Publish(ctx context.Context, space SpaceID, thread ThreadID, envelope []byte) (Ref, error)
    Subscribe(ctx context.Context, space SpaceID, from Cursor) (<-chan Delivery, error)
    History(ctx context.Context, space SpaceID, thread ThreadID, from Cursor, limit int) ([]Delivery, error)
    Ack(ctx context.Context, ref Ref) error
    Members(ctx context.Context, space SpaceID) ([]Account, error)
}
```

`Capabilities` says what the provider can do natively and what waggle must emulate: threads, history, maximum message size, federation, delivery guarantees, and which roles it may play (carrier, console, or both). A message larger than the provider's limit travels in one of two ways:

- **chunks**: the base64 envelope split into numbered parts carrying the envelope id and the digest of the whole; the receiver reassembles, checks the digest and only then verifies; a missing part means no delivery;
- **pointer**: the message carries the envelope id, its SHA-256 digest and where to fetch it from a durable carrier; the receiver fetches and checks the digest before verification.

Consoles never need either: they show a rendering, not the envelope (§9.3).

### 9.2 Capability matrix

Limits are typical defaults, to be checked against each provider's current documentation before implementation.

| Provider | Threads | History | Size limit → handling | Federation | Delivery guarantees | Roles |
| --- | --- | --- | --- | --- | --- | --- |
| Local directory (Maildir-style) | emulated (thread id in the payload) | yes (files) | none in practice | no (one machine) | durable atomic append, per-sender order by `seq`, cursors | carrier (the last stage of every other provider); console through `mail tail` |
| NATS JetStream | emulated (subject per space; thread id in payload and header) | yes (stream retention) | server maximum payload, 1 MB by default → pointer | no (servers of one operator; leaf nodes connect servers, not independent domains) | durable streams, acknowledgements, redelivery, deduplication on `Nats-Msg-Id`, per-message TTL, KV compare-and-set | carrier (internet, first) |
| IRC (Ergo) | none native; header lines and reply tags where supported | server history where `CHATHISTORY` is supported (Ergo supports it) | 512-byte lines; `draft/multiline` batches on Ergo → rendering only, pointer to the full text | no (one network: a project runs its own server) | none (no acknowledgements, no durability guarantee) | console |
| XMPP / MUC | `<thread>` element | Message Archive Management (XEP-0313) | server stanza limit, configurable → pointer | yes (server-to-server between domains) | per-hop acknowledgements with stream management (XEP-0198), delivery receipts (XEP-0184) | carrier (federation), console |
| Matrix | native threads | yes (room history on homeservers) | 64 KiB per event → pointer | yes (homeserver federation) | replicated room history, read receipts, eventual consistency across homeservers | carrier (alternative for federation), console |
| Slack | native threads | yes, subject to the workspace plan | about 40,000 characters per message → attachment or pointer | Slack Connect between organizations (proprietary) | API delivery with rate limits and event retries | console; carrier only as an untrusted adapter with the exact envelope attached |
| Telegram | reply chains; forum topics in supergroups | yes, for chats the bot can read | 4,096 characters per text message → document attachment or pointer | no (one provider) | bot API with update offsets | console; carrier only as an untrusted adapter |
| Email (SMTP/IMAP) | `In-Reply-To` / `References` headers | yes (mailboxes) | server limits, typically megabytes → attachment | yes (SMTP between domains) | store-and-forward, no ordering, eventual | carrier for slow asynchronous federation; console |

IRC suits a project that runs its own server: every participant registers there, and the network is one trust domain. XMPP suits federation: each organization keeps its own server and accounts, and rooms span domains.

### 9.3 Carriers, gateways and consoles

A provider plays one or more of three roles:

- **Carrier**: stores and delivers signed envelopes byte-exact with guarantees (durable, acknowledged, ordered, expiring). Only carriers deliver `coord` and `cmd`.
- **Console**: renders messages for people and lets them type. What a person types enters as a low-authority `coord` message; a console never carries commands.
- **Gateway**: translates to another protocol at the edge.

| Role | Adapter | Use |
| --- | --- | --- |
| carrier, one machine | **local**: Maildir-style mailbox under the Session Manager state root (generalizes today's notice inbox) | orchestrators on one Mac; the final delivery stage of every other carrier |
| carrier, internet (first) | **NATS JetStream** on a dedicated Mac mini on the operators' tailnet, run as a separate service reachable over the tailnet only; that host also serves production traffic, so the service never touches its shared ingress; streams per project; durable consumers per principal; `Nats-Msg-Id` = envelope id; `Nats-TTL` from expiry; leases in a JetStream KV bucket; one NATS user per operator | 2–3 operators in different locations |
| carrier, not planned | board server mailbox | not revived: an assessment on 2026-09-24 found a mailbox in the old board server would cost a week and more plus security rework (user-level tokens without scope or expiry, no push), against a few days for NATS |
| carrier, federation | **XMPP (Prosody)**, envelopes in a namespaced element (`urn:apiary:envelope:1`), s2s certificates and allowlists, members-only rooms | other organizations with their own servers |
| carrier or console, adapters | **Matrix**, **Slack**, **Telegram**, **email**: untrusted adapters under the rules of §9.1 and §9.2 | teams that already live there |
| gateway | **A2A v1** (Agent2Agent, Linux Foundation): a partner's agent receives our `coord.request` as an A2A task; our Agent Card is signed | partners that offer an agent as a service |
| console (first) | **`task-board mail tail`**: a terminal UI streaming rooms and mailboxes with full text, verification marks, pending approvals and one-key `approve` | every stage |
| console (optional) | **IRC bridge on Ergo**: one channel per project for any IRC client | people who prefer a chat client |

**How a console shows long messages when IRC lines are 512 bytes.** The console does not carry the signed envelope; it shows a rendering. Each message appears as a header line (`[cmd.halt] orch:alpha → worker:RUN-…abc123, approved by op:alice ✓`), followed by the body: clients that support the `draft/multiline` extension (Ergo serves it) show the body as one message; other clients see it split into several lines, capped at a configurable number with a pointer to the full text (`task-board mail read 0192f7`). The 512-byte limit matters for a carrier, which must deliver the exact signed bytes with acknowledgements; it does not matter for a rendering. Our own terminal UI shows everything in full and is the faster first console.

## 10. Delivery into agent loops: doorbell and pull

1. The host appends the verified message to the principal's mailbox.
2. It rings a fixed-template doorbell: `[waggle] 2 new messages for orch:alpha (coord from orch:beta; cmd from op:alice). Read: task-board mail read`. `interrupt` → Codex `turn/steer` with `expectedTurnId`, Claude paste at a safe point; `next_turn` → Codex `thread/queue/add`, Claude after the turn; `on_request` → no doorbell.
3. The agent pulls with `task-board mail read` or a per-session MCP tool with a capability token; the tool records `read`.
4. Doorbells coalesce; a stale-turn steer falls back to `next_turn`; with no client attached messages wait durably.

**Through the session host.** Putting anything into a live agent session is not waggle code. Notice injection is one capability of the session-host module (working name `agent-session-host`) that is being extracted from task-board's session daemon, with one contract per harness: start or resume a session; apply, clear and acknowledge the exact revision of a goal; inject a notice; emit goal and usage events. waggle rings the doorbell through that contract and ships no harness-specific code. The mechanics in step 2 are today's implementations of the capability for Codex and Claude; a harness gains waggle delivery the day it gains a session-host adapter.

## 11. Coordination: vocabulary and decisions

The coordination layer turns messages into agreements. The v1 vocabulary is the `coord` types of §5; the protocol lab (§13) proposes more, and each enters this section through a reviewed change.

| Need | v1 | Candidates (to be validated by the lab) |
| --- | --- | --- |
| Negotiate | `request`, `propose`, `question`, `answer` | `objection`, `withdraw` |
| Claim work | `handoff-offer`, `handoff-accept`, `handoff-result` (A1, §19.4) | `claim`, `release` for work items inside a leased scope |
| Own a scope | leases (§12) | explicit `lease-request` / `lease-grant` messages over the lease store's compare-and-set |
| Decide together | — | `support` and a `decide` record carrying the supporting signatures |
| Lead a room or scope | the lease holder with the highest term | — |

**Decisions by quorum**, after the honeybee swarm. A proposal (the scout's dance) states an option with its evidence. Participants send `support` or `objection` (the stop signal that damps competing dancers). When the support for one option reaches the quorum that the policy sets for that decision type, any participant may publish a `decide` record whose envelope carries the supporting signatures (multi-signature, §4.1). Anyone who holds the roster can therefore check a decision, and a model's reading of the thread never substitutes for the record.

**Leadership** is lease ownership: whoever holds the lease of a room or scope with the highest term leads it. Losing the lease (expiry, handoff, a higher term elsewhere) ends the leadership, and messages signed under the old term are refused.

## 12. Scope leases

A lease says which orchestrator owns which part of the project right now: holder, monotonic `term`, random token, expiry renewed by heartbeat, stored with compare-and-set (local board on one machine; NATS JetStream KV across machines; board server or Apiary authority later).

Example: Alpha (Alice) leases Epic Auth, Beta (Bob) leases Epic Billing. When Alpha tries to spawn a developer on a Billing story, task-board looks the lease up, sees Beta, and refuses with `scope_owned_by orch:beta`. To work there, Alpha sends `coord.handoff-offer` or a `coord.request`; if Beta accepts, the lease moves to Alpha with a higher term. If Alpha's session dies and comes back on another machine, the new session gets a higher term, and anything still signed under the old term is refused.

The kernel checks leases at the three places where damage happens: **spawn** (starting work), **integrate** (landing code) and **board commit** (writing board state). Messages alone cannot prevent two orchestrators from working on the same story at once (a finding of the earlier coordination-rooms research); the lease can.

## 13. Protocol lab

The coordination vocabulary should come from how agents actually coordinate, not only from design. The protocol lab is an experiment run beside CM1–CM2:

- **Fixture**: a disposable project with independent stories that can be worked on concurrently, split across three machines, each with its own orchestrator.
- **Channel**: deliberately primitive, a shared text file or any chat, so the agents' own conventions become visible.
- **Freedom**: the agents may invent any coordination protocol: who takes which story, how to negotiate, how to elect a leader, whether to build a better channel.
- **Logs**: every orchestrator's messages, decisions and board actions are kept for analysis.
- **Reset**: one command drops the project state so the cycle can be repeated with other prompts, models or channels.

Findings become candidate coordination message types and lease rules (§11); each one enters the specification through a reviewed change. In the lab the channel carries no authority; in production authority stays outside the model: leases, gates and the signature policy decide, whatever the agents agree in text.

## 14. Fast path to internet coordination

| Step | Delivers | Kept afterwards |
| --- | --- | --- |
| F1 | `waggle` module: envelope, multi-signature SSHSIG via `ssh-agent` (including FIDO user verification), roster and policy, verification pipeline, receipts, audit; interop tests against `ssh-keygen -Y verify` | yes |
| F2 | local carrier, doorbell + pull through the session host's notice injection, `task-board mail send\|read\|tail\|approve\|rooms\|rejected` | yes |
| F3 | NATS carrier as a separate tailnet-only service on a dedicated Mac mini on the operators' tailnet, clear of the host's shared ingress; NATS users per operator; workers never get NATS credentials | yes |
| F4 | scope leases enforced at spawn, integrate, board commit | yes |
| F5 (optional) | IRC bridge on Ergo | yes, as a console |

## 15. Security stages

| Stage | Admission | Confidentiality |
| --- | --- | --- |
| Trusted (F1–F4) | roster and policy pull requests; operator keys (FIDO for approvals); session, run and host certificates; NATS users per operator; tailnet ACLs | tailnet (WireGuard); carrier trusted for plaintext |
| External (CM4) | partner CA lines limited to `coord`; hosted accounts first, then XMPP federation; inspectors and quarantine on by default | carrier trusted for plaintext by default; an end-to-end profile (MLS) is a separate decision |

Always: children never receive carrier credentials or the parent's manager variables (fix the child-environment leak in task-board spawns first); worker mailboxes are receive-only.

## 16. Module and repository

`waggle` is its own public Go module and repository (`relux-works/waggle`), consumed by task-board through `go.mod` at tags exactly like `skill-agents-management`: no `replace`, a `go.work` only for local development, one tag per wave. Later consumers: Apiary's AgentHost and coordinator, and `curator-run` if it ever sends messages. For delivery into live sessions waggle consumes the session-host module's notice-injection contract (§10) and depends on no harness.

## 17. Phases

| Phase | Delivers | Done when |
| --- | --- | --- |
| CM0 | this draft; `waggle` repository | merged as draft |
| CM1 | F1 + F2; env-leak fix; notices and directives mapped onto classes; doorbell through the session host | two orchestrators on one machine coordinate in a room; each cancels only its own children; an operator approval with a FIDO touch halts a foreign worker |
| Lab (parallel) | the protocol lab of §13 | candidate coordination types and lease rules documented with the runs that produced them |
| CM2 | F3 + F4 (+ F5); coordination vocabulary refined by the lab; Amendment A1 (§19) | orchestrators of 2–3 operators in different locations split scopes and negotiate over the internet; a person watches in `mail tail`; a board accepts a signed approval and a signed external event, and a hand-off across boards returns its result |
| CM3 | orchestrator → remote worker through AgentHost (Apiary stage 1) | a remote worker receives `clarify`/`cancel` and cannot send `coord`/`cmd` |
| CM4 | other organizations: partner CAs, inspectors on by default, XMPP federation, A2A gateway, quotas | a partner coordinates in a shared room and cannot command our workers |

## 18. Decisions

| # | Question | Status |
| --- | --- | --- |
| 1 | Name, module, repository | decided: `waggle`, own public module and repository |
| 2 | Signature container | decided: SSHSIG, multi-signature by role; Apiary authority wire gets an amendment |
| 3 | Injection defense | decided: §7 |
| 4 | First internet carrier | decided: NATS JetStream on a dedicated Mac mini on the operators' tailnet, as a separate tailnet-only service clear of the host's shared ingress |
| 5 | Console | decided 2026-09-24: own terminal UI first, IRC bridge optional |
| 6 | Federation | proposed: XMPP + A2A gateway |
| 7 | Priority | decided: start now, in parallel with the migration |
| 8 | Key hardware | decided: pluggable signer providers and assurance levels (Apple Secure Enclave, FIDO, TPM, PIV, ordinary-key fallback) |
| 9 | Layers | decided: a communication layer and a coordination layer built on it (§2) |
| 10 | Conversation model and providers | decided: spaces, threads, participants and messages; providers are untrusted pipes, so proprietary chats are adapters and never the core (§8, §9) |
| 11 | IRC and XMPP | decided: both are providers; IRC for a project that runs its own server, XMPP for federation |
| 12 | Protocol lab | decided: its findings feed the coordination layer through reviewed changes; production authority stays outside the model (§13) |
| 13 | Delivery into sessions | decided: through the session-host module's notice injection (working name `agent-session-host`), no harness-specific code in waggle (§10) |
| 14 | Board-server mailbox | decided: not revived (§9.3) |
| 15 | Board processes | direction decided 2026-09-24: Amendment A1 (§19), implemented with CM2: board acts (approvals, relaxations, human transitions), signed external events from service principals, the hand-off result signed by the receiving board's host, user verification in board policy, `required` by default for approvals, relaxations, delegations and halts once a project sets a signature policy, with `hardware-bound` keys as the floor, delegation of board acts, installs of audited skills included, to the grantor's own orchestrator sessions within grants the board verifies (§19.8), confirmations of process changes as signed board acts (`board.confirm`) under the fixed floor of curator-trust `spec/trust.md`, and roster changes made by a confirmed commit and confirmed against the roster of the version in force (§3); the hand-off result is signed in a new signer role, `result` (§19.4) |

## 19. Amendment A1: board processes

Status: direction accepted 2026-09-24; implemented with CM2 (§17). Consumer: the process-configuration specification in [relux-works/curator-playbook](https://github.com/relux-works/curator-playbook) (`spec/process-configuration.md` §4.2, §6.4, §6.5, §6.9). The trust design in [relux-works/curator-trust](https://github.com/relux-works/curator-trust) (`spec/trust.md`) decides who counts as a person and what a confirmation, a person's narrowing act or a grant means. A1 keeps the wire forms; trust.md is their meaning. A1 adds two classes, one principal kind, one `coord` type signed in a new signer role, one policy field, a board type for confirming a version of a process and one for installing an audited skill, and delegation: two board types for granting and revoking it and one delegate namespace (§19.8). A1 changes §1–§18 in these places: §3, where a teammate is added by a confirmed commit and a roster change is confirmed against the roster of the version in force (curator-trust `spec/trust.md`); step 2 of §4.3, which gains the signer role `result` (§19.4); §4.4 and §4.5, which gain the operator companion app as a second way to show that a person acted, its signer provider and enrollment verifier, and the rule that every verifier re-checks enrollment evidence (§19.5). Nothing else in §1–§18 changes meaning.

A process configuration lets a board decide who may move work: a person approves a purchase, a supplier's system confirms an order, another process returns its result. Before A1 none of these had a waggle form. A verifier following §4.3 had to refuse each of them, or the board had to verify them by rules waggle did not define.

### 19.1 Board acts: approvals, relaxations and human transitions

A new class, `board`, carries the acts of people on a board. Its author namespace is `board.v1@waggle`, and its approver namespace is `approve.board.v1@waggle`.

| Type | The act | Signature category in the process configuration |
| --- | --- | --- |
| `board.approve` | an approval that an `approvals` guard counts | `approvals` |
| `board.relax` | a temporary exception to a role's hard requirement: an `ask` a person answered | `relaxations` |
| `board.transition` | a transition a person fires: out of a human-owned state, an override of a tool- or process-owned state, or a transition marked with a `risk` | `human-transitions`, `overrides`, `spending`, `halts` |
| `board.delegate` | a person's grant of named board acts to an orchestrator role, with its conditions, scope, expiry and budgets (§19.8) | `delegations` |
| `board.revoke` | the revocation of a delegation (§19.8) | `delegations` |
| `board.install` | the installation of an audited skill from the Curator registry into the project, by a person or by a delegate (§19.8) | — (signed under a delegation, or where the project asks) |
| `board.confirm` | a person's confirmation of a version of the process: the configuration digest of its resolved configuration at a named head (process-configuration §2.4) | none: a confirmation has a fixed floor, a key that requires a person's presence, that no project setting lowers (curator-trust `spec/trust.md` §4) |

- **The approver signature is the human act.** The author is whoever composed the request: usually the orchestrator session that asks for the approval. An operator acting alone signs both roles. The board counts only the `approver` signature, read against the project's process configuration at its pinned revision — the version in force (process-configuration §2.4) — and only from the approver set of the act's type:
  - `board.approve`: a principal in the group of the human role the approval is recorded as (the project's `people` map);
  - `board.relax`: the operator whose runs the exception covers, present in `people`; an exception never reaches another operator's runs;
  - `board.transition`: for a human-owned state, a principal in the group of the role that owns it; for an override of a tool- or process-owned state, or of a landing whose hosted checks could not run, any principal in `people`; for a transition marked with a `risk`, the group of the role that owns its source state, or any principal in `people` when an agent owns it;
  - `board.delegate`: a principal who may perform every act the grant covers, by the rules above and, for installs, any principal in `people`;
  - `board.revoke`: the grantor, or any principal in `people`, because a revocation only narrows authority;
  - `board.install`: any principal in `people`;
  - `board.confirm`: a principal of the group that the project's `policy.confirmers` names in the version in force, or, without it, the board's anchor operator; it is verified against the roster, revocations and enrollments of the version it replaces, never the version being confirmed, and only a key that requires a person's presence counts (curator-trust `spec/trust.md` §3, §4).

  A quorum counts distinct approver principals present in the roster at that revision.
- **The payload binds everything the act depends on**, inside the signed bytes:
  - the board and the element;
  - the state the act concerns and that state's entry number;
  - the act itself: the role approved as, the transition, or the requirement relaxed with its scope and expiry;
  - the evaluated revision: the configuration digest of the process version in force (process-configuration §2.4), and for work under a Change Request the revision under evaluation. The digest, not a commit, is bound: a commit that is not yet confirmed changes nothing, and a newly confirmed version whose digest is unchanged keeps a pending act valid;
  - the approver principal, in `signers.approver`;
  - every field the element's type declares, with its value (`shown`), and the digest of those fields (`fields_digest`), so the approver sees, and the signature binds, the whole element as it stood.

  An approval therefore cannot be replayed after the element enters the state again, against another configuration or revision, or after any declared field of the element changed: the board compares the bound digest with the digest of the element's current fields (§19.7).

  A `board.confirm` concerns no element: it binds the board, the head it confirms, the configuration digest of that version and of the version it replaces, and the confirmation's sequence number in the board's chain, so it cannot be replayed for another version or revive after a revert (curator-trust `spec/trust.md` §4). The signing device computes both digests from the configurations it renders. A `board.install` binds the skill and the exact package identity installed.
- **What you see is what you sign.** A board act is signed only through the operator-facing approve command (§4.2, `task-board mail approve`). The command renders every body field, including the element fields shown, before the key signs. An agent never holds an operator key.
- **Addressing.** A board act has no `to`. The board named in `body.board` consumes it, whether it arrives through that board's local command or over a carrier.

```json
{
  "schema": "waggle-v1",
  "type": "board.approve",
  "id": "0192f8b1-3c5d-7e6f-8a9b-0c1d2e3f4a5b",
  "from": "orch:cafe@acme",
  "signers": { "approver": "op:alice@acme" },
  "project": "acme",
  "issued_at": 1790000000000,
  "expires_at": 1790086400000,
  "body": {
    "board": "acme/cafe",
    "element": "PO-1042",
    "state": "approval",
    "entry": 2,
    "act": { "approve": { "role": "manager" } },
    "revision": { "process": { "digest": "sha256:9d41…" } },
    "shown": { "item": "milk", "supplier": "Milk & Co", "amount": "620.00 EUR", "due": null, "invoice": null },
    "fields_digest": "sha256:4c7a…"
  }
}
```

### 19.2 Signed external events

A new class, `event`, carries events from systems outside the project, such as a payment provider or a supplier's portal. Its only type is `event.post`, and its author namespace is `event.v1@waggle`. Only service principals (§19.3) sign it.

- **Body.** The board; the element; the event name the process configuration declares (`supplier.confirmed`, `payment.received`); the source, meaning the provider and the provider's own event identifier; typed, bounded data.
- **Expiry.** `expires_at` is required. An event past it is refused as `expired` (§4.3 step 5).
- **De-duplication.** The bridge derives the envelope `id` deterministically from the provider and its event identifier (a name-based UUID). A webhook the provider delivers twice therefore yields one envelope id, and step 6 of §4.3 drops the repeat. The same id with other bytes is `rejected:conflicting_duplicate`.
- **Meaning.** Events reach the board through `task-board event post` or over a carrier. What an accepted event does is the board's rule (process-configuration §6.9): which transition it fires, and how long the board keeps it.

```json
{
  "schema": "waggle-v1",
  "type": "event.post",
  "id": "5b0e4f1c-9a2d-5c7e-b3f8-1d6a0e9c4b27",
  "from": "svc:payments@acme",
  "project": "acme",
  "issued_at": 1790000000000,
  "expires_at": 1790003600000,
  "body": {
    "board": "acme/cafe",
    "element": "INV-2087",
    "name": "payment.received",
    "source": { "provider": "bank", "event_id": "evt_81f3" },
    "data": { "amount": "620.00 EUR" }
  }
}
```

### 19.3 Service principals

A new principal kind, `svc:`, names a bridge: a small service that receives a provider's webhook and posts the signed event.

| Principal | Example | Key |
| --- | --- | --- |
| Service (bridge) | `svc:payments@acme` | the bridge's own key; preferably a short-lived certificate from an operator key acting as a CA, as for a host |

```text
# services: signed external events only
svc:*@acme cert-authority,namespaces="event.v1@waggle" ssh-ed25519 AAAA…
```

- **One class only.** A service principal is admitted only for `event.v1@waggle`. The roster linter rejects any other namespace on an `svc:` line, and the verifier refuses an `svc:` signature in any other class even where a roster line allows it.
- **Signer groups.** A published process template declares the events it reacts to and a signer group for each. Only the committed project file binds a group to service principals. The board accepts an event only when its author is bound to the event's signer group at the pinned revision; an event whose group is unbound is refused and logged.
- **Bridges verify first.** A bridge verifies the provider's own webhook authentication (its signature scheme, or mutual TLS) before it signs, and signs only events of the groups it is bound to. It never holds an operator key and cannot sign `board`, `coord` or `cmd`.

### 19.4 The hand-off result

`coord` gains one type, `handoff-result`. It closes the hand-off that `handoff-offer` and `handoff-accept` open. When the element the accepting process created from the offer reaches a terminal state, the receiving board's kernel reports the result to the offering orchestrator.

- **Not a model's claim.** An orchestrator session is an agent, and what it writes proves who said it, not that it is true. The result therefore comes from the receiving board's kernel, which observes the terminal transition, and is signed with a host key of that board, a key no orchestrator session holds.
- **Signer role and namespace.** A1 adds one signer role to step 2 of §4.3: `result`, whose namespace prefix is `result.`, so that it signs `coord.handoff-result` under `result.coord.v1@waggle`, and whose principal is `payload.signers.result`. A `coord.handoff-result` carries exactly one signature, in the role `result`, by a `host:` principal of the receiving board; its signature policy requires that role and refuses an `author` signature. Only `host:` roster lines admit the namespace, and `handoff-result` is the only type it signs: the verifier refuses a `handoff-result` signed in any other role, and a host signature on any other `coord` type.
- **The host key.** An OS account other than the operator's, or a privileged service, holds the signing host key, whether or not it is `hardware-bound`, so that no process running as the operator can sign a result. A hardware-bound key in the operator's own account does not qualify: any process running as the operator can use it whenever it does not demand presence, and a host key signs unattended. A host whose key the operator's account can use does not sign results.
- **Body.** The offer (`offer`, the id of the `handoff-offer`); the received element and its board; its final state and resolution; the receiving board's revision (its board-state commit) at the terminal transition; the envelope ids of the signed act or event that caused that transition, when one did; a bounded summary. `reply_to` is the id of the `handoff-accept`.
- **Verification.** The offering board accepts the result only from a host principal that its committed project file binds to the receiving board (process-configuration §6.5; the binding form comes with cross-board hand-off).
- **One result per offer.** A second result for the same offer with other bytes is refused as a conflicting duplicate.
- **Effect.** The offering board turns a verified `handoff-result` into the `process.returned` event on the offering element, whose `process.result` guards match the final state (process-configuration §6.5).

```json
{
  "schema": "waggle-v1",
  "type": "coord.handoff-result",
  "id": "0192f9c4-7d1e-7a2b-9c3d-4e5f6a7b8c9d",
  "from": "host:books-1@acme",
  "to": "orch:cafe@acme",
  "signers": { "result": "host:books-1@acme" },
  "project": "acme",
  "reply_to": "0192f8e0-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
  "issued_at": 1790500000000,
  "expires_at": 1791104800000,
  "body": {
    "offer": "0192f8d7-6e5f-7a4b-9c3d-2e1f0a9b8c7d",
    "element": { "board": "acme/books", "id": "INV-2087" },
    "final_state": "paid",
    "resolution": null,
    "board_revision": "7c1e0b…",
    "caused_by": ["5b0e4f1c-9a2d-5c7e-b3f8-1d6a0e9c4b27"],
    "summary": "paid in full"
  }
}
```

### 19.5 User verification in board policy

The process configuration names which board acts need a signature (`policy.signatures`). It uses waggle's two key requirements (§4.4, §4.5): `min_assurance` (`software`, `hardware-bound`, `attested`) and `user_verified`:

- `required`: the approver signature must be a FIDO2 signature whose authenticator flags say the user was verified (PIN or fingerprint), or a signature from an enrolled companion key (below). The FIDO2 form is the requirement §4.4 spells as `key = "sk"` with `user_verification = true`; new policies spell it `user_verified = "required"`.
- `if-supported`: the flag is required when the signature format carries it; other signatures that meet `min_assurance` are accepted.

**Defaults.** Whenever a project sets a signature policy:

- `min_assurance` is `hardware-bound`, because any process running as the operator's OS user can obtain `software` signatures from `ssh-agent`;
- `user_verified` is `required` for approvals (`board.approve`), relaxations (`board.relax`), delegations (`board.delegate`, `board.revoke`) and halts: `cmd.halt` and `cmd.cancel` to runs of the process, and transitions marked `risk: halt`;
- `user_verified` is `if-supported` for spending, overrides and other human transitions (`board.transition`).

The project may set `user_verified` for every category at once or per category, and may lower `required` to `if-supported`. With the default `min_assurance`, `if-supported` admits Secure Enclave and other hardware-bound keys, never software keys.

**Why the default is strict.** Only a FIDO2 signature with the user-verified flag, or a signature from an enrolled companion key (below), shows that a person acted. Any process running as the operator's OS user can obtain `software` signatures from `ssh-agent`, and `hardware-bound` ones whenever the key does not demand presence for each use. A Secure Enclave key protects the key, which cannot be copied, but not presence: a Touch ID prompt, when the key's policy asks for one, leaves no trace in the signature (§4.4, §4.5). An operator whose keys cannot show user verification approves with a FIDO key, or the project lowers the requirement.

**The operator companion app** (decided 2026-09-24: people confirm through the phone in the common case) meets the floor of a confirmation, and is the way a headless node is served; it also meets `required` for board approvals and every other board act (decided 2026-09-25; curator-trust `spec/trust.md` §6). It is a signer provider (§4.5) on the operator's phone:

- its key lives in the phone's Secure Enclave or Android Keystore under an access control that itself requires the person's biometric authentication, so that the phone's operating system, not only the app, refuses a use without the person; the app uses it only after the person has seen the rendered payload;
- the key is enrolled with the platform's attestation that a genuine companion app, identified by its signed app identity, generated it in the phone's secure hardware with that access control, which gives it the assurance `attested`; its enrollment record carries `user_verification = "app-enforced"`. The evidence that proves `app-enforced` is that attestation chain, from the key to the platform vendor's root (Apple App Attest and key attestation, Android Key Attestation), naming the app and, where the platform attests it, the access control. Every verifier re-checks it against the vendor roots and never trusts the stored verification result alone;
- the app is paired with the node at initialization through an out-of-band handshake that pins keys on both sides, under the pairing invariants of curator-trust `spec/trust.md` §8: the attestation checked at every pairing, a single-use short-lived QR secret kept out of transcripts and logs, no pairing from an agent session, and a confirmation for every later pairing. Requests reach the app over the operator's tailnet or a relay that sees only messages encrypted end to end to the pinned keys.

A signature from an enrolled companion key meets the confirmation floor: the user verification is enforced by the attested key and app rather than written into each signature, and the enrollment record says so. For board approvals and every other board act it meets `user_verified = "required"` as well (decided 2026-09-25; curator-trust `spec/trust.md` §6). The app's design and platforms are open (curator-trust `spec/trust.md` §8, §14; process-configuration §17.3).

**Before the app exists**, board acts are signed on the machine in a local session, where the operating system can prompt the person, or, for a headless node, from the operator's workstation: a client command fetches the complete payload from the node over SSH, computes what it signs and renders it on the workstation, signs it there with a FIDO2 key created with required user verification, and returns only the signature (curator-trust `spec/trust.md` §9). A key is never used through a forwarded `ssh-agent`: a forwarded agent lets any process of the operator's account on the node request signatures while the session lasts, and a key that shows nothing signs whichever request reaches it first.

**Which policy applies.** For `board` types other than `board.confirm`, and for `cmd.halt` and `cmd.cancel` sent to runs of a process, the requirements come from the project's process configuration at its pinned revision, the version in force; `board.confirm` has the fixed floor of curator-trust `spec/trust.md` §4. Entries for the same types in `.waggle/policy.toml` may only add requirements; where both speak, the stricter wins.

### 19.6 Roster and classes, summarized

| Namespace (author role) | Types | Who may send → to whom |
| --- | --- | --- |
| `board.v1@waggle` (approver: `approve.board.v1@waggle`) | `approve`, `relax`, `transition`, `delegate`, `revoke`, `install`, `confirm` | orchestrator session or operator → the board named in the body; the operator's approver signature is the act |
| `delegate.board.v1@waggle` (delegate role) | `approve`, `relax`, `transition`, `install` on behalf of a grantor | orchestrator session of a delegate role → the board named in the body; counted as the grantor's act only within the referenced delegation (§19.8) |
| `event.v1@waggle` | `post` | service (bridge) → the board named in the body |
| `result.coord.v1@waggle` (signer role `result`; added type `coord.handoff-result`) | `handoff-result` | the receiving board's host → offering orchestrator |

- Operator roster lines add `board.v1@waggle,approve.board.v1@waggle`. Orchestrator session lines add `board.v1@waggle` as author only, and `delegate.board.v1@waggle`, which a verifier accepts only with a delegation that covers the act; never `approve.board`, and never `result.coord`.
- Host lines add `result.coord.v1@waggle`, for hand-off results only.
- Worker lines gain nothing: a worker key still cannot produce any of these signatures.

### 19.7 What stays with the board

waggle provides the forms, verification and delivery. The board decides what they mean (process-configuration §4.2, §6.3–§6.5, §6.9):
- which act a guard counts;
- freshness: an approval counts only for the state entry it names, under a Change Request only for the revision it names, only while the configuration digest it binds is the one in force, and only while the element's declared fields still hash to the digest it binds;
- quorum over distinct principals: each delegated act counts as at most one principal, its grantor, and one delegate session counts once, so a quorum above one needs distinct people acting directly or through distinct delegate sessions of distinct grantors;
- which acts may be delegated, to which roles, under which ceilings, and each delegation's conditions, scope, budgets and expiry;
- which signer group may post which event;
- how accepted events are kept;
- where a returned result routes.

### 19.8 Delegated acts

A person may delegate named board acts to an orchestrator: approvals of a role, transitions out of human-owned states, relaxations, and installs of audited skills from the Curator registry. The process configuration says what may be delegated, to which roles and under which ceilings (process-configuration §4.2). A delegation lets autonomous work go on without a person's signature on every step, while the person keeps what the delegation leaves out.

- **The grant.** A `board.delegate` act records the delegation: the grantor, the delegate role, the sessions that may act under it, the acts, their `field` conditions, a subtree scope, an expiry and optional budgets per period. By default only the grantor's own orchestrator sessions act under a grant: sessions whose certificates the grantor's key issued (§3). A grant may name other session principals explicitly, never a pattern. It is a human act: the grantor's approver signature, made through the approve command, is what counts, at the policy's assurance and user verification (§19.5) and never below the floor of a confirmation (curator-trust `spec/trust.md` §7). The board verifies the envelope once, when it records the grant; later delegated acts reference the recorded grant, so the envelope's `expires_at` bounds only its delivery and the grant's own expiry bounds its use.
- **The delegated act.** A `board.approve`, `board.relax`, `board.transition` or `board.install` made by a delegate carries no approver signature. The delegate's orchestrator session signs it in the `delegate` role, under the namespace `delegate.board.v1@waggle`, and the body names the grant: `on_behalf_of` holds the grantor and the envelope id of the `board.delegate`. A delegate signature is never an approver signature; the namespaces keep them apart.
- **The chain the board verifies.** The board verifies:
  1. the `board.delegate` envelope and its approver signature;
  2. the delegate signature, made by a session whose certificate the grantor's key issued, or by a session principal the grant names, and whose certificate extension `waggle-role@relux.works` names the delegate role, checked against the role alias at the pinned revision;
  3. that the act falls inside the grant's acts, scope, conditions, budgets and expiry, and inside the project's ceilings at the pinned revision;
  4. that the delegate session, or a run it spawned, wrote no declared field of the element, unless the project's rule allows self-approval (process-configuration §4.2).

  The board's kernel alone counts budgets, from the delegated acts it has recorded; no signature carries budget state. Checking a budget and recording the act are one compare-and-set in the lease store (§12), so a board shared across machines needs the CM2 carrier's lease store for budgeted grants.
- **Certificate form.** A grantor may instead certify the delegate's session key directly: an OpenSSH certificate signed by the grantor's key, whose key id names the delegation and whose critical option `delegation@waggle` carries the grant's envelope id. It is valid no later than the grant expires. A verifier accepts it only when the grantor's roster line admits certifying for `delegate.board.v1@waggle` and the certificate's constraints match the grant. The reference form works on every carrier; the certificate form lets a delegate prove its authority without the record being fetched.
- **Revocation and expiry.** A `board.revoke` act, a person's narrowing act (curator-trust `spec/trust.md` §5), ends a delegation at once, like the grant's expiry. The board refuses delegated acts under it that arrive later, delegated approvals under it that no transition has used yet stop counting, and exceptions that a relaxation under it created end, while a run already working under one finishes; acts already consumed stay valid. Across machines a revocation takes effect through the lease store. In the certificate form, short certificate lifetimes bound the exposure further.
- **Everything else waits for the person.** An act the delegation does not cover is not taken. The board notifies the grantor, and the grantor signs the act, or does not.

```json
{
  "schema": "waggle-v1",
  "type": "board.approve",
  "id": "0192fa01-5b6c-7d8e-9f0a-1b2c3d4e5f60",
  "from": "orch:cafe@acme",
  "signers": { "delegate": "orch:cafe@acme" },
  "project": "acme",
  "issued_at": 1790000000000,
  "expires_at": 1790086400000,
  "body": {
    "board": "acme/cafe",
    "element": "PO-1043",
    "state": "approval",
    "entry": 1,
    "act": { "approve": { "role": "manager" } },
    "on_behalf_of": { "grantor": "op:alice@acme", "delegation": "0192f7aa-0b1c-7d2e-8f3a-4b5c6d7e8f90" },
    "revision": { "process": { "digest": "sha256:9d41…" } },
    "shown": { "item": "milk", "supplier": "Milk & Co", "amount": "180.00 EUR", "due": null, "invoice": null },
    "fields_digest": "sha256:81e0…"
  }
}
```

### 19.9 Implementation

With CM2 (§17):
- the verification pipeline gains the `board` and `event` classes, the `svc:` kind and the `user_verified` policy field;
- the board types `board.delegate` and `board.revoke`, the `delegate.board.v1@waggle` namespace, and verification of the delegation chain in both forms, with budgets and revocations in the carrier's lease store;
- the board types `board.install` and `board.confirm`, the latter verified against the roster of the version it replaces under the fixed floor of curator-trust `spec/trust.md` §4 (a confirmation is a signed board record from PC3; `board.confirm` carries it over the CM2 carrier);
- roster changes confirmed against the roster of the version in force, and enrollment evidence re-checked against the vendor roots, the companion verifier included;
- the signer role `result` in step 2 of §4.3;
- the approve command renders board acts;
- the board verifies `event.post` from `task-board event post` and from bridges;
- a hand-off across boards runs `handoff-offer`, `handoff-accept` and `handoff-result` over the CM2 carrier.

Until then, signed human acts use only the forms of §1–§18, and a hand-off across boards is unavailable. A hand-off between processes on one board needs none of this.
