# Track: waggle (agent communication)

Status: **DRAFT v3**, 2026-09-24. Specification: `spec/waggle.md` (draft v4). Continues the task-board coordination-rooms epic. Research: three notes in the private task-board repository (`skill-project-management`): the task-board messaging map, the Apiary and ax messaging map, and the agent-messaging landscape. Roadmap: parallel track CM0–CM4, high priority, starts now.

## Why

Orchestrators sharing a project cannot talk to each other, and an orchestrator reaches its children only through goals and directives the children must poll. What exists is run-shaped, unsigned, local to one machine and partly unsafe. The same channel will serve Apiary later.

## Design in one screen

- **Signatures travel inside every message.** A message is an envelope: the exact payload bytes, a namespace naming the message class, and an SSHSIG signature made with an SSH Ed25519 key through `ssh-agent`. The project's committed `allowed_signers` file says which key may sign which class for which principal. Every receiver verifies the same way on every transport; stored envelopes can be re-verified offline with stock `ssh-keygen`.
- **Several signatures on one message.** An envelope carries a list of signatures over the same bytes, each with a role: `author` (the orchestrator), `approver` (an operator), `endorser` (a parent orchestrator). The namespace binds class and role, so no signature can be reused in another role. A committed policy says which message types need which roles; an approval can require a FIDO key with user verification (PIN or fingerprint), which is the only proof of physical presence that other people can check. Example: halting another operator's worker needs the orchestrator's signature plus the operator's touch.
- **Any key hardware, one format.** Signer providers are plugins: Apple Secure Enclave (with an attestation that the key was generated in a genuine Apple device's enclave and never leaves it), FIDO keys (attestation plus a per-signature user-verified flag), later TPM and PIV, and an ordinary key as the fallback. Each enrolled key gets an assurance level (`software`, `hardware-bound`, `attested`), and the policy says which level each kind of approval needs. Platform SSO fits as the way a team signing service issues short-lived operator certificates only to attested devices.
- **Lessons from SSH certificate practice.** Short-lived certificates as the main control, revocation lists for emergencies, CA keys on hardware, automatic issuance at session start, principals and namespaces restrict each key, certificates can be inspected, every acceptance is logged with key id, serial and CA; OpenSSH certificates are single-level, so chains are expressed by listing CA keys in the roster and by co-signatures.
- **Sessions sign, agents compose.** The host creates a short-lived key per orchestrator session, certified by the operator's key; the agent writes content through `task-board mail send`, the host fills the sender and signs. A worker's certificate allows reports only, so a worker cannot produce a command or coordination message over any transport.
- **Delivery is a doorbell plus a pull.** A fixed-template doorbell goes into the agent loop; the agent pulls the content as tool output inside an untrusted-data frame.
- **Injection defense is layered.** Mandatory: authority outside the model (gates, scope leases, supervisor-executed cancel) and typed, framed messages. Optional: deterministic inspectors, a model-based guard, a quarantined extractor that turns free text into typed requests, quarantine with human release, rate limits. External senders get the optional layers by default.
- **Scope leases decide, messages negotiate.** Each orchestrator leases a scope (Epics, Stories) with a term and token in a compare-and-set store; the kernel checks the lease at spawn, integrate and board commit.
- **Transports are adapters.** Carrier = stores and delivers signed envelopes with guarantees. Console = a place people watch and type (their input enters as low-authority `coord`). Gateway = a translator at the edge.

## Fast path to internet coordination

| Step | Delivers | Thrown away later? |
| --- | --- | --- |
| F1 | `waggle` library: envelope, SSHSIG via `ssh-agent`, roster, verification pipeline, receipts | no |
| F2 | local carrier, doorbell + pull, `task-board mail` commands; two orchestrators on one machine | no: the last delivery stage of every carrier |
| F3 | NATS JetStream carrier on a dedicated Mac mini on the operators' tailnet; NATS user per operator; workers never get NATS credentials; leases in JetStream KV | no: the internet carrier and the scale path |
| F4 | scope leases enforced at write boundaries | no |
| F5 (optional) | IRC bridge on Ergo for people | no: the console |

Why NATS first for the internet: it already gives durable streams, acknowledgements, redelivery, deduplication, per-message TTL, compare-and-set KV and per-subject permissions, in one Go binary with a mature Go client, so nothing has to be built into the board server first. The team's tailnet already shares nodes between people, so the server can sit on a private address.

## Roles of IRC, XMPP and A2A

- **Console first: our own terminal UI** (`task-board mail tail`): full messages, verification marks, pending approvals, one-key approve.
- **IRC = optional console.** People open any IRC client and watch the orchestrators' room. A console shows a rendering, not the signed envelope: a header line plus the body, as one message for clients with the multiline extension or split into lines with a pointer to the full text for others. The 512-byte line limit only matters for a carrier, which must deliver the exact signed bytes with acknowledgements.
- **XMPP = federation carrier later.** When another organization joins with its own server: real inter-domain federation, invite-based accounts, members-only rooms, archives. Our signed envelopes ride inside XMPP messages.
- **A2A = partner gateway later.** An open protocol (Linux Foundation, v1.0 in 2026) for handing a task to another organization's agent over HTTP; used when a partner offers an agent service instead of joining rooms.

## Milestones

| # | Delivers | Done when |
| --- | --- | --- |
| CM0 | draft spec; `waggle` repository | merged as draft |
| CM1 | F1 + F2; child-environment leak in task-board spawns fixed; notices and directives mapped onto message classes | two orchestrators on one machine coordinate in a room; each cancels only its own children |
| CM2 | F3 + F4 (+ F5) | orchestrators of 2–3 operators in different locations split scopes and negotiate over the internet; a person can watch |
| CM3 | orchestrator → remote worker via AgentHost (Apiary stage 1) | a remote worker receives `clarify`/`cancel` and cannot send `coord`/`cmd` |
| CM4 | other organizations: partner CAs, inspectors on by default, XMPP federation, A2A gateway, quotas | a partner coordinates in a shared room and cannot command our workers |

## Priority in words

Start now, in parallel with the migration off agents-infra. CM0–CM1 need nothing from the migration except the `waggle` repository; CM1 touches tb-sessiond like milestone M3, so their pull requests are sequenced. CM2 follows CM1 directly: internet coordination is the goal. CM3 lands with Apiary stage 1; CM4 comes with the first external organization.

## Decisions

| # | Question | Status |
| --- | --- | --- |
| 1 | Name | decided: `waggle`, its own public Go module and repository, consumed like `skill-agents-management` |
| 2 | Signature container | decided: SSHSIG with multi-signature by role; Apiary's authority wire gets an amendment |
| 3 | First internet carrier | decided: NATS JetStream on a dedicated Mac mini on the operators' tailnet |
| 4 | Console | proposed: own terminal UI first; IRC bridge optional |
| 4b | Injection defense | decided: the layered model of the specification |
| 5 | Federation | proposed: XMPP + A2A gateway |
| 6 | Priority | decided: start CM0–CM1 now |
