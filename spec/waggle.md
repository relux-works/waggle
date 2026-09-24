# waggle: agent communication specification

Status: **DRAFT v5**, 2026-09-24. Not normative. Track document: `spec/track.md`. Evidence: research notes in the private task-board repository (`skill-project-management`): the task-board messaging map (*TM*), the Apiary and ax messaging map (*AX*) and the agent-messaging landscape (*LS*). Continues the task-board coordination-rooms epic. v5 splits the protocol into a communication layer and a coordination layer, adds the conversation model and the provider interface, the protocol lab, delivery through the session-host module and the placement of the internet carrier; v4 of the same day added signer providers.

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
- Classes are communication-layer facts; what `coord` types mean, and which new ones exist, is the coordination layer's vocabulary (§11).

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
| Claim work | `handoff-offer`, `handoff-accept` | `claim`, `release` for work items inside a leased scope |
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
| CM2 | F3 + F4 (+ F5); coordination vocabulary refined by the lab | orchestrators of 2–3 operators in different locations split scopes and negotiate over the internet; a person watches in `mail tail` |
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
