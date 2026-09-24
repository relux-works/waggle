# waggle

A secure agent coordination and communication protocol: signed messages between agents. waggle defines one message envelope, one verification pipeline and one delivery model for agent orchestrators that share a project. Orchestrators coordinate with each other, first on one machine, then across a few operators over the internet, later with other organizations. Orchestrators also send one-way commands (clarify, reprioritize, cancel, halt) to their own workers; workers can never instruct orchestrators. Every message carries SSH signatures (SSHSIG) over its exact bytes, so any receiver on any transport can check who sent it and who approved it, and a stored message can be verified again offline with stock OpenSSH.

## Status

**Draft design.** Nothing is implemented yet. This repository holds the design under review; the Go module arrives with milestone CM1. Interface, field and command names in the specification are proposals until they ship.

## Layout

| Path | Contents |
| --- | --- |
| [`spec/waggle.md`](spec/waggle.md) | the specification (draft v5.1): two layers; principals and roster, envelope and multi-signature, verification pipeline, signature policy, signer providers and key assurance, message classes, injection defense, conversation model, providers and their capabilities, delivery through the session host; coordination vocabulary and quorum decisions, scope leases, the protocol lab; phases; Amendment A1 for board processes (board acts, signed external events, service principals, the hand-off result, user verification in board policy) |
| [`spec/track.md`](spec/track.md) | the delivery track: why, the design on one screen, the fast path to internet coordination, milestones CM0–CM4, decisions |
| [`spec/waggle.ru.md`](spec/waggle.ru.md), [`spec/track.ru.md`](spec/track.ru.md) | Russian translations; the English files are canonical |

## Design in brief

- **Signatures travel inside every message.** An envelope is the exact payload bytes plus one or more SSHSIG signatures, each with a role: `author`, `approver` or `endorser`. The signature namespace binds message class and role, so a signature made in one role or for one class cannot be reused for another.
- **The roster is a stock OpenSSH `allowed_signers` file** committed in the project, next to a revocation list and a policy file that says which message types need which signatures. Adding a teammate is a reviewed pull request.
- **Sessions sign, agents compose.** The host creates a short-lived certified key for each orchestrator session and signs what the agent writes; the agent never holds a key. A worker's certificate allows reports only, so a worker cannot produce a command or a coordination message over any transport.
- **Any key hardware, one format.** Signer providers are plugins: `ssh-agent`, Apple Secure Enclave, FIDO keys, later TPM and PIV, with an ordinary key as the fallback. Each enrolled key gets an assurance level (`software`, `hardware-bound`, `attested`), and the policy says which level an approval needs.
- **Delivery is a doorbell plus a pull, behind layered injection defense.** A fixed-template doorbell enters the agent loop and the agent pulls messages as untrusted data. Authority stays outside the model (gates, scope leases, supervisor-executed cancel and halt); inspectors, guards and quarantine are optional layers on top.
- **Two layers.** A communication layer moves signed messages between principals; a coordination layer built on it decides who works on what, with scope leases and quorum decisions.
- **Scope leases decide, messages negotiate; providers are untrusted pipes.** Orchestrators lease parts of a project with a term and a token in a compare-and-set store. Providers sit behind one interface: carriers deliver the signed bytes (a local mailbox, then NATS JetStream for the internet, XMPP for federation later), consoles render messages for people, gateways (A2A) translate at the edge, and chats such as Slack or Telegram can be adapters because authority comes from signatures, never from the provider.

## Consumers

- **task-board**: orchestrator and worker messaging through `task-board mail`. It will consume this module through `go.mod` at tags, the same way it consumes [skill-agents-management](https://github.com/relux-works/skill-agents-management).
- **Apiary** (later): the agent host and coordinator of the Curator agent runtime.
- **Process configuration** ([curator-playbook](https://github.com/relux-works/curator-playbook)): board approvals, signed external events and hand-off results across boards (Amendment A1).

## Links

- Specification: [`spec/waggle.md`](spec/waggle.md)
- Track and milestones: [`spec/track.md`](spec/track.md)
- In Russian: [`spec/waggle.ru.md`](spec/waggle.ru.md), [`spec/track.ru.md`](spec/track.ru.md)
- The SSHSIG format used by `ssh-keygen -Y sign` and `-Y verify`: [PROTOCOL.sshsig](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.sshsig)

## License

Licensed under the Apache License, Version 2.0; see [LICENSE](LICENSE).
