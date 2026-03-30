# Synvya Docs

Cross-cutting architecture and specification documents for Synvya. Feature specs that span multiple repositories live here; single-repo implementation specs live in their respective repos and link back to this one.

## Architecture Documents

| Document | Description |
|---|---|
| [Auth & Real-Time Event Processing](architecture/auth-and-realtime.md) | Nostr key custody (Keycast), flexible authentication, 24/7 event processing, internal API contracts, deployment topology |

## Per-Repository Implementation Specs

Each component has its own implementation spec that references the architecture doc above for system context:

| Repository | Spec | Scope |
|---|---|---|
| [`Synvya/client`](https://github.com/Synvya/client) | [`docs/specs/auth-migration.md`](https://github.com/Synvya/client/blob/main/docs/specs/auth-migration.md) | Migrate client app auth from browser-local keypairs to Keycast. Login UI, signing migration, existing user migration. |
| [`Synvya/mcp-server`](https://github.com/Synvya/mcp-server) | [`docs/specs/thin-client-migration.md`](https://github.com/Synvya/mcp-server/blob/main/docs/specs/thin-client-migration.md) | Remove all Nostr code from MCP server. Call Event Processor API. Add `check_reservation` tool. |
| [`Synvya/event-processor`](https://github.com/Synvya/event-processor) | [`docs/specs/event-processor.md`](https://github.com/Synvya/event-processor/blob/main/docs/specs/event-processor.md) | Always-on relay listener, NIP-59 decryption via Keycast, reservation handlers, internal HTTP API. |
| [`Synvya/keycast`](https://github.com/Synvya/keycast) | Architecture doc, Section 5 | Fork of [divinevideo/keycast](https://github.com/divinevideo/keycast) with AWS KMS and SES customizations. |

## Implementation Order

```
1. Synvya/keycast            Deploy auth + signing infrastructure
     |
     v
2. Synvya/event-processor    Depends on Keycast for signing and decryption
     |
     +------------------+
     v                  v
3a. Synvya/mcp-server  3b. Synvya/client    Can be done in parallel
```
