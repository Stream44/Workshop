<table>
  <tr>
    <td><a href="https://Stream44.Studio"><img src=".o/stream44.studio/assets/Icon-v1.svg" width="42" height="42"></a></td>
    <td><strong><a href="https://Stream44.Studio">Stream44 Studio</a></strong><br/>Open Development Project</td>
    <td>Preview release for community feedback.<br/>Get in touch on <a href="https://discord.gg/9eBcQXEJAN">discord</a>.</td>
    <td>Designed by Hand<br/><b>AI assisted Code</a></td>
  </tr>
</table>

⚠️ **Disclaimer:** Under active development. Code has not been audited. APIs and interfaces are subject to change!

Stream44 Studio Workshop
===

This repository contains Pattern & Tool documentation for the [Stream44.Studio](https://Stream44.Studio) **Open Development Project**. Reach out at [discord.gg](https://discord.gg/9eBcQXEJAN) ([Stream44.Studio](https://Stream44.Studio) server) if you have any questions or post an issue on github.

Patterns
---

| ID | Name / Description | Status |
|----|----------------|--------|
| [WP-2026-01](./Patterns/WP-2026-01-GitRepository-Identifier.md) | **Git Repository Identifier** - Establishes a stable `did:repo:<hash>` identifier for git repositories via signed empty inception commits. | DRAFT |
| [WP-2026-02](./Patterns/WP-2026-02-GitRepository-Integrity.md) | **Git Repository Integrity** - Four-layer progressive validation model for git repository integrity: commit origin, repository identifier, Gordian Open Integrity provenance, and XID document governance. | DRAFT |
| [WP-2026-03](./Patterns/WP-2026-03-GitRepository-GordianOpenIntegrity.md) | **Gordian Open Integrity** - Cryptographic decision provenance system for git repositories using XID Documents, Gordian Envelopes, and Provenance Marks. Establishes a separable document graph for governance models that can be distributed and verified independently of code. | DRAFT |

Tools
---

| ID | Name / Description | Status |
|----|-------------|--------|
| [WT-2026-01](./Tools/WT-2026-01-GitRepositoryIdentifier.md) | **Git Repository Identifier** - TypeScript capsule for creating and validating `did:repo:` identifiers. | DRAFT |
| [WT-2026-02](./Tools/WT-2026-02-GitRepositoryIntegrity.md) | **Git Repository Integrity** - TypeScript capsule implementing multi-layer integrity validation. | DRAFT |
| [WT-2026-03](./Tools/WT-2026-03-GordianOpenIntegrity.md) | **Gordian Open Integrity**- TypeScript reference implementation using the Encapsulate capsule framework. Includes CLI tools, GitHub Actions, and a standalone git-level validation script. | DRAFT |

Provenance
===

[![Gordian Open Integrity](https://github.com/Stream44/Workshop/actions/workflows/gordian-open-integrity.yaml/badge.svg)](https://github.com/Stream44/Workshop/actions/workflows/gordian-open-integrity.yaml?query=branch%3Amain) [![DCO Signatures](https://github.com/Stream44/Workshop/actions/workflows/dco.yaml/badge.svg)](https://github.com/Stream44/Workshop/actions/workflows/dco.yaml?query=branch%3Amain)

Repository DID: `did:repo:00f3c248e2a9ddc8b1116e60003100a59dfd34e0`

<table>
  <tr>
    <td><strong>Inception Mark</strong></td>
    <td><img src=".o/GordianOpenIntegrity-InceptionLifehash.svg" width="64" height="64"></td>
    <td><strong>Current Mark</strong></td>
    <td><img src=".o/GordianOpenIntegrity-CurrentLifehash.svg" width="64" height="64"></td>
    <td>Trust established using<br/><a href="https://github.com/Stream44/t44-blockchaincommons.com">Stream44/t44-BlockchainCommons.com</a></td>
  </tr>
</table>

(c) 2026 [Christoph.diy](https://christoph.diy) • Code: [MIT](./LICENSE.txt) • Text: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) • Created with [Stream44.Studio](https://Stream44.Studio)
