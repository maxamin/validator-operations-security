# Monitoring, Hardening and an Operations Checklist

The previous chapters covered the dangerous moments: key generation, slashing protection, migrations and external dependencies. This chapter covers the steady state — the monitoring that tells you something is wrong, the hardening that makes compromise harder, and a checklist you can run before launch and on a schedule.

## What to monitor

Validator monitoring has two layers. **Infrastructure metrics** tell you whether machines and clients are healthy. **Duty metrics** tell you whether validators are doing their job. Duty metrics matter more, because a host can be perfectly healthy while its validators miss every attestation due to a bad fee recipient, a sync issue or a signer refusing requests.

| Signal | Source | Alert when |
|---|---|---|
| Attestation inclusion and correctness | Client validator monitor, beacon API, explorers | Effectiveness drops below baseline for 2+ epochs |
| Missed proposals | Beacon API, client logs | Any missed proposal |
| Signer refusals | Web3Signer logs and metrics | Any slashing-protection refusal (investigate immediately) |
| Peers and sync distance | Client Prometheus metrics | Low peer count, sync distance growing |
| Execution client health | EL metrics, `eth_syncing` | Not synced, engine API errors |
| Builder/relay | MEV-Boost logs | Relay errors, repeated fallbacks to local blocks |
| Balances | Beacon API | Balance decreasing over an epoch window |
| Slashing events | Beacon API, explorers | Any of your keys `slashed: true` |
| Credential changes and exits | Beacon API, EIP-7002 contract events | Any unplanned exit request |

A slashing-protection refusal is the single most important alert in this list. It means something tried to sign a conflicting message. Either two signers exist, or history was lost and restored. Treat it as an incident, not noise.

Clients expose Prometheus metrics; Lighthouse's `--validator-monitor-auto` and similar options in other clients produce per-validator duty metrics. Pair them with Grafana dashboards and an alert manager that pages a human. Public explorers such as beaconcha.in offer notifications and are useful as an independent second view, because a monitoring stack that shares a failure with your validators will go quiet exactly when you need it.

## Hardening

**Network.** Only the peer-to-peer ports need to be reachable from the internet. Keep beacon node HTTP APIs, the execution Engine API, the validator client keymanager API, signer APIs and metrics on private interfaces or behind authenticated proxies. See **blockchain-rpc-node-security** for node-level detail.

**Hosts.** Minimal operating systems, automatic security updates for the OS, SSH with keys only and no root login, a host firewall, full-disk encryption for machines holding keystores, and separate hosts (or at least separate users and containers) for signer and client.

**Accounts and access.** Separate cloud accounts for validator infrastructure and treasury. Hardware-backed MFA for everyone with access. Break-glass procedures written down. No shared admin accounts.

**Supply chain.** Verify client release signatures or checksums, pin container image digests, and stage upgrades on a testnet node before mainnet. Follow client release channels so hard-fork deadlines are never missed.

**Keymanager API.** Validator clients expose a standard keymanager API for importing and deleting keys and slashing history. It requires a bearer token; protect the token file and do not expose the port.

**Exits and credentials.** Pre-sign voluntary exit messages for emergencies and store them encrypted offline; they let you exit validators even if the signing infrastructure is lost. Keep withdrawal addresses in a multisig and monitor them.

## Incident playbooks

Write short playbooks before you need them:

- **Suspected key compromise:** stop signing everywhere, keep slashing history, decide whether to exit, and broadcast pre-signed exits if so. A stolen signing key cannot take funds, but it can slash you indefinitely until exit completes.
- **Signer refusal alert:** find every process that holds the key; stop all but one; compare databases.
- **Client bug or chain split:** follow client team announcements; do not switch clients mid-incident without exporting history.
- **Relay misbehaviour:** remove the relay from configuration and restart MEV-Boost.

For broader incident handling see **web3-incident-response**.

## Worked example

*Execution status: the commands, code and numbers in this example were not executed in this book's CI; they are illustrative. Verify locally on a devnet, testnet or fork before relying on them.*

Wire up a minimal alert for signer refusals on a devnet.

1. Run Web3Signer with metrics enabled (`--metrics-enabled --metrics-host=127.0.0.1`) alongside a Kurtosis devnet as in Chapter 3.
2. Scrape it with Prometheus and list the metric names exposed at `/metrics`. Identify the counter that tracks slashing-protection denials for your version.
3. Add an alert rule:

```yaml
groups:
- name: validator
  rules:
  - alert: SlashingProtectionRefusal
    expr: increase(<refusal_counter_name>[5m]) > 0
    labels: { severity: page }
    annotations:
      summary: "Signer refused a conflicting signature - possible duplicate signer"
```

4. Trigger it by sending a conflicting signing request as in Chapter 3. Confirm the alert fires and reaches your notification channel.
5. Add a second, independent check using the beacon API from a different host that queries your validators' liveness every epoch.

## Operations checklist

**Before launch**

- [ ] Keys generated offline with verified tools; mnemonic backed up offline.
- [ ] Withdrawal credentials are `0x01`/`0x02` and point at a multisig you control.
- [ ] One signer per key; slashing protection enabled and backed up with replication.
- [ ] Doppelganger protection enabled.
- [ ] Only P2P ports public; all APIs private.
- [ ] Minority clients chosen or split across clients.
- [ ] Multiple relays pinned; `min-bid` set.
- [ ] Duty-level monitoring, external second view, and paging on signer refusals.
- [ ] Pre-signed exits stored offline.

**Every change**

- [ ] Migration runbook followed with sign-off at the "old signer verified stopped" step.
- [ ] Upgrades tested on a testnet node first.

**Quarterly**

- [ ] Restore drill of the slashing DB on a devnet.
- [ ] Review who has access to hosts, signer and withdrawal multisig.
- [ ] Re-check client distribution and relay list.

## Common pitfalls

- Monitoring hosts but not duties.
- Silencing signer refusal alerts because they were noisy after a restore.
- Monitoring that runs on the same host as the validator client.
- Pre-signed exits stored next to the keystores they are meant to back up.
- Upgrading clients on mainnet first because the fork date is close.

## Exercises

1. Build a Grafana dashboard for one devnet validator showing attestation inclusion, balance and signer decisions.
2. Run a tabletop for "signer refusal at 3 a.m." with a colleague and time how long it takes to identify the duplicate.
3. Generate and verify a pre-signed voluntary exit on a testnet, then broadcast it and follow the validator through the exit queue.

## Further reading

- Keymanager API specification: <https://github.com/ethereum/keymanager-APIs>
- Beacon API specification: <https://github.com/ethereum/beacon-APIs>
- beaconcha.in notifications and API documentation
- Related books: **blockchain-rpc-node-security**, **web3-incident-response**
