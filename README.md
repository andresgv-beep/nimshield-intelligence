# NimShield Intelligence

Signed threat intelligence feed for [NimOS](https://github.com/andresgv-beep) — distributed defenses for self-hosted NAS systems.

NimShield Intelligence is the proactive layer of **NimShield**, the security module built into NimOS. While NimShield reactively detects attackers at the door (honeypots, brute force, scanning), this feed lets every NimOS instance know about threats that are *already* attacking others — and block them before they arrive.

The feed is **publicly readable but cryptographically impossible to forge**: every release is signed with an Ed25519 key whose private half never leaves an isolated machine. Any NimOS in the world can download and verify it without an account, a token, or any configuration.

## How it works

```
┌─────────────────────────────┐         ┌────────────────────────────────┐
│   THE FACTORY (isolated)    │         │   EACH NimOS (consumer)        │
│                             │  push   │                                │
│   1. fetch public sources   │ ──────► │   · embedded base feed         │
│   2. aggregate + dedup      │ GitHub  │   · refresh every ~2 days      │
│   3. build signed manifest  │ public  │   · verify Ed25519 signature   │
│   4. publish                │ ◄────── │   · validate per-file hashes   │
│                             │  pull   │   · observe → then block       │
│   PRIVATE KEY lives here    │         │   · keep N versions (rollback) │
└─────────────────────────────┘         └────────────────────────────────┘
        private key                            public key embedded
```

The signing key's private half lives **only** on a dedicated, isolated machine — never on a NAS, never in this repo, never exposed to the internet. If any NimOS instance is ever compromised, it still cannot publish a fake feed or sign anything: the blast radius is contained by design.

## Repository layout

```
nimshield-intelligence/
├── README.md
├── latest/                    # NimOS always fetches from here
│   ├── manifest.json          # signed index (versions + per-file SHA-256)
│   ├── manifest.json.sig      # Ed25519 signature of the manifest
│   ├── blocklist_ipv4.txt     # malicious IPv4 addresses / CIDRs
│   └── blocklist_ipv6.txt     # malicious IPv6 addresses / CIDRs
└── versions/                  # historical releases (for rollback)
    └── <feed_version>/
```

## The manifest

Only the manifest is signed. It indexes every file with its SHA-256 hash, so one signature protects the whole release: a valid signature proves the hashes are authentic, and the hashes prove each file is intact.

```jsonc
{
  "schema_version": 1,                    // manifest format version
  "feed_version": 42,                      // content version (monotonic)
  "generated_at": "2026-06-26T03:00:00Z",
  "files": [
    {
      "name": "blocklist_ipv4.txt",
      "type": "blocklist_ip",
      "sha256": "…",
      "entries": 14203,
      "action": "block"                    // block | observe | enrich
    }
  ]
}
```

The format is generic on purpose. Today it carries IP blocklists; tomorrow it can carry compromised domains, malicious ASNs, IOCs, file hashes, or detection rules — **without changing the protocol**, only the manifest contents.

## Verifying the feed

Every consumer verifies the signature before applying anything. The public key is embedded in NimOS. To verify manually with the `nimsign` tool:

```sh
nimsign verify latest/manifest.json latest/manifest.json.sig nimshield-public.key
```

A tampered manifest — even a single changed byte — fails verification and is rejected. NimOS then keeps its previous known-good feed.

### Public key

```
(to be published here once the production key is generated)
```

## Safety model

- **Signature** protects against a forged or tampered feed (an attacker).
- **`feed_version`** is monotonic — an older feed can never be replayed over a newer one.
- **`schema_version`** lets older NimOS versions read newer feeds gracefully, ignoring what they don't understand.
- **Rollback** (multiple retained versions) protects against a validly-signed but *broken* feed (human error).
- **Observation mode** logs matches without blocking when a feed is new, so false positives are caught before they bite.
- **Whitelist and reputation** always take priority over the feed inside NimShield.

## Sources

The feed aggregates reputable public threat sources. Entries are deduplicated, validated, and sanity-checked before publication. Conservative by default to minimize false positives.

## Disclaimer

This is community-maintained threat intelligence provided as-is, with no warranty. The blocklists aggregate publicly available data. Always keep your own whitelist for addresses you trust.

## License

See [LICENSE](LICENSE).
