# WP-2026-03 — Gordian Open Integrity

### Stream44 Studio Workshop Pattern

Authors: Christoph Dorn<br/>
Date: February 15, 2026<br/>
Status: DRAFT — Work in Progress

---

## Introduction

**Gordian Open Integrity** is a system for recording cryptographically verifiable decisions *about* a git repository *in* the git repository itself. It uses [XID Documents](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2024-010-xid.md), [Gordian Envelopes](https://datatracker.ietf.org/doc/draft-mcnally-envelope/), and [Provenance Marks](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2025-001-provenance-mark.md) to establish an open namespace for decisions of any kind, tied cryptographically to a [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md) commit.

The document graph formed by the inception envelope and its registered decision documents is **self-describing and portable**. While the envelopes live inside the git repository, they can also be extracted and distributed independently — for example, as a standalone governance model, a policy bundle, or a trust anchor — without requiring the recipient to clone the entire codebase. Each envelope carries its own provenance chain, XID identity, and document-path self-reference, making it verifiable in isolation. When combined with the repository, the full commit-signature audit provides an additional layer of assurance.

Given the latest provenance mark via a publishing channel, verifiers can confirm the integrity of all recorded decisions with complete confidence — including the repository code itself — enabling secure distribution via public peer-to-peer networks.

### Design Goals

1. **Self-contained** — All provenance data lives in the repository itself.
2. **Separable** — The document graph can be distributed and verified independently of the code.
3. **Cryptographically rigorous** — Every commit is SSH-signed; every document carries a provenance mark chain.
4. **Open namespace** — Implementers design their own URI layouts and "Gordian Envelope Spaces" under `.o/`.
5. **Minimal dependencies** — Only `git`, `ssh-keygen`, and the Gordian cryptographic stack are required.
6. **Verifiable by third parties** — A cloned repository plus a published provenance mark is sufficient for full verification.

### Terminology

| Term | Definition |
|------|-----------|
| **Repository Identifier** | A signed, empty commit whose hash forms the repository DID. Created per [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md). May be the first commit or attached later. |
| **Repository DID** | `did:repo:<identifier-commit-hash>` — a stable identifier for the repository derived from the repository identifier commit. |
| **Inception Envelope** | The XID Document at `.o/GordianOpenIntegrity.yaml` — the root of the provenance chain. |
| **Trust Root** | The combination of a repository identifier and an inception envelope that establishes the cryptographic foundation for all subsequent provenance. A trust root can be reset while preserving the repository identifier. |
| **Decision Document** | Any additional XID Document under `.o/<hostname>/`, registered in the inception envelope's `Documents` map. |
| **Provenance Mark** | A hash-chain entry ([BCR-2025-001](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2025-001-provenance-mark.md)) that seals a document revision into a verifiable sequence. |
| **Generator** | The secret state used to produce successive provenance marks. MUST NOT be committed to git. |
| **XID** | An eXtensible IDentifier ([BCR-2024-010](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2024-010-xid.md)) — a stable 32-byte identifier derived from the hash of the initial signing key. |
| **LifeHash** | A visual hash ([lifehash.info](https://lifehash.info)) of the provenance mark, stored as SVG for human recognition. |
| **Ricardian Contract** | A textual trust statement embedded in the provenance commit (e.g. `Trust established using https://github.com/Stream44/t44-BlockchainCommons.com`). |

### Normative References

- [BCR-2024-010: XID](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2024-010-xid.md)
- [BCR-2024-003: Gordian Envelope](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2024-003-envelope.md)
- [BCR-2025-001: Provenance Mark](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2025-001-provenance-mark.md)
- [draft-mcnally-envelope: Gordian Envelope IETF Draft](https://datatracker.ietf.org/doc/draft-mcnally-envelope/)

---

## 1. Repository File Layout

### 1.1 Committed Files (`.o/`)

```
.o/
├── GordianOpenIntegrity.yaml                   # Inception envelope (REQUIRED)
├── GordianOpenIntegrity-InceptionLifehash.svg  # LifeHash of inception mark
├── GordianOpenIntegrity-CurrentLifehash.svg    # LifeHash of current mark
└── <hostname>/<namespace>/                     # Decision document namespace
    └── <document>.yaml                         # Decision document envelope
```

All `.o/` files are tracked and committed.

### 1.2 Secret Files (`.git/o/`)

```
.git/o/
├── GordianOpenIntegrity-generator.yaml        # Inception provenance generator (SECRET)
└── <hostname>/<namespace>/
    └── <document>-generator.yaml              # Document provenance generator (SECRET)
```

Generator files are **never committed**. They contain the secret state needed to advance the provenance mark chain and MUST be backed up separately.

#### Generator File Format

Generator files are JSON with the following fields:

| Field | Encrypted | Description |
|-------|-----------|-------------|
| `res` | No | Resolution level (integer) |
| `nextSeq` | No | Next sequence number (integer) |
| `seed` | **Yes** | Generator seed (base64) |
| `chainID` | **Yes** | Chain identifier (base64) |
| `rngState` | **Yes** | RNG state (base64) |

When a provenance key is provided, sensitive fields (`seed`, `chainID`, `rngState`) are encrypted using **AES-256-GCM**. Each encrypted value is prefixed with the algorithm, a key fingerprint, and the ciphertext:

```
aes-256-gcm:<key-fingerprint-8hex>:<base64(iv || ciphertext || auth-tag)>
```

The key fingerprint is the first 8 hex characters of `SHA-256(encryptionKey)`. This ensures that both the provenance key AND the generator file are required to advance the provenance chain — neither is sufficient alone.

---

## 2. Provenance Document Format

All `.o/*.yaml` files use YAML with two document sections separated by `---`.

### 2.1 Machine-Readable Section (YAML)

```yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
$defs:
  envelope:
    $ref: "https://datatracker.ietf.org/doc/draft-mcnally-envelope/"
  mark:
    $ref: "https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2025-001-provenance-mark.md"
envelope: "ur:envelope/..."
mark: "a9ea4602"
```

| Field | Description |
|-------|-------------|
| `$schema` | JSON Schema draft identifier |
| `$defs` | Schema references for the `envelope` and `mark` fields |
| `envelope` | Gordian Envelope encoded as a [UR](https://developer.blockchaincommons.com/ur/) string |
| `mark` | Current provenance mark identifier — first 4 bytes as hex (e.g. `"a9ea4602"`) |

### 2.2 Human-Readable Section (after `---`)

```yaml
---
# Repository DID: did:repo:<hash>
# Current Mark: <hex> (<bytewords>)
# Inception Mark: <hex> (<bytewords>)
# <envelope in human-readable notation>
# <ricardian contract>
```

Lines are prefixed with `#`. This section is informational — verification uses only the machine-readable section.

### 2.3 Complete Example

```yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
$defs:
  envelope:
    $ref: "https://datatracker.ietf.org/doc/draft-mcnally-envelope/"
  mark:
    $ref: "https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2025-001-provenance-mark.md"
envelope: "ur:envelope/lftpsogdhkwzdtfthptokigtvwnnjsqzcxknsktpsogdhkwz..."
mark: "a9ea4602"
---
# Repository DID: did:repo:abc123def456
# Current Mark: a9ea4602 (� WARM ARCH RAMP HORN)
# Inception Mark: 7b3f1a08 (� IRON AQUA BUZZ DICE)
# XID(d4c3b2a1) [
#     "GordianOpenIntegrity.SigningKey": "ssh-ed25519 AAAA... signing_ed25519"
#     "GordianOpenIntegrity.RepositoryIdentifier": "did:repo:abc123def456"
#     'key': Bytes(78) [
#         'allow': 'All'
#     ]
#     'provenance': Bytes(115)
# ]
# Trust established using https://github.com/Stream44/t44-BlockchainCommons.com
```

### 2.4 Parsing Algorithm

```
1. Split on first "\n---" occurrence
2. YAML.parse(section_before_separator)
3. Extract: envelope (UR string), mark (hex)
4. Decode: Envelope.fromUrString(envelope)
5. Reconstruct: XIDDocument.fromEnvelope(decoded_envelope)
```

---

## 3. Envelope Assertions

Four predicates are defined on XID Document envelopes:

### 3.1 `"GordianOpenIntegrity.SigningKey"` — SSH Key Binding

| | |
|---|---|
| **Predicate** | `"GordianOpenIntegrity.SigningKey"` |
| **Object** | SSH public key string (e.g. `"ssh-ed25519 AAAA... signing_ed25519"`) |
| **Where** | Inception envelope only |
| **Purpose** | Binds the XID to the SSH key used for commit signing. Replaced on key rotation. |

### 3.2 `"GordianOpenIntegrity.RepositoryIdentifier"` — Repository DID Binding

| | |
|---|---|
| **Predicate** | `"GordianOpenIntegrity.RepositoryIdentifier"` |
| **Object** | Repository DID string (e.g. `"did:repo:abc123def456"`) |
| **Where** | Inception envelope only |
| **Purpose** | Binds the inception envelope to the repository identifier, enabling Layer 4 validation that the DID matches the actual identifier commit. |

### 3.3 `"GordianOpenIntegrity.Documents"` — Document Registry

| | |
|---|---|
| **Predicate** | `"GordianOpenIntegrity.Documents"` |
| **Object** | JSON map of document paths → XID identifiers: `'{"<path>":"XID(<hex>)"}'` |
| **Where** | Inception envelope (added/updated when documents are introduced) |
| **Purpose** | Registry for all decision documents. Enables discovery and XID verification. |

### 3.4 `"GordianOpenIntegrity.Document"` — Document Self-Reference

| | |
|---|---|
| **Predicate** | `"GordianOpenIntegrity.Document"` |
| **Object** | The document's own path (e.g. `".o/example.com/policy/v1.yaml"`) |
| **Where** | Each decision document envelope |
| **Purpose** | Identifies the repository file path, enabling verification when distributed separately. |

### 3.5 Envelope Structure

**Inception envelope:**

```
XID(<hex>) [
    "GordianOpenIntegrity.SigningKey": "<ssh-public-key>"
    "GordianOpenIntegrity.RepositoryIdentifier": "did:repo:<hash>"
    "GordianOpenIntegrity.Documents": '{"<path>":"XID(<hex>)"}'
    'key': Bytes(78) [
        'allow': 'All'
    ]
    'provenance': Bytes(115)
]
```

**Decision document envelope:**

```
XID(<hex>) [
    "GordianOpenIntegrity.Document": "<document-path>"
    'key': Bytes(78) [
        'allow': 'All'
    ]
    'provenance': Bytes(115)
]
```

The `'key'` assertion carries the Ed25519 public key with `allow: 'All'` permissions. The `'provenance'` assertion carries the binary-encoded provenance mark.

---

## 4. Operations

### 4.1 Repository Initialization

Initialization establishes the trust root. It creates a [repository identifier](./WP-2026-01-GitRepository-Identifier.md) followed by the inception envelope.

#### Prerequisites

- An **Ed25519 key** — the *signing key*, used for signing git commits and deriving XID key material (generate with `ssh-keygen -t ed25519 -f <path> -N "" -C signing_ed25519`)
- An **Ed25519 key** — the *provenance key*, a dedicated key used exclusively to initialize the provenance mark generator. This key MUST be kept secure, stored separately from the signing key, and never reused for any other purpose. If compromised, an attacker could forge provenance marks.
- Author name and email for git commits

#### Step 1 — Create Repository Identifier

Create a [repository identifier](./WP-2026-01-GitRepository-Identifier.md) using the signing key. This produces a signed, empty commit and a `.repo-identifier` file. The commit hash becomes the **repository DID**: `did:repo:<identifier-commit-hash>`.

#### Step 2 — Create XID Document

Using the Gordian stack:

1. Derive a `PrivateKeyBase` from the signing key (SHA-256 hash of key file data)
2. Derive a provenance seed from the provenance key (SHA-256 hash of key file data)
3. Create a XID Document with:
   - One inception key (`allow: 'All'`)
   - A provenance mark generator (initialized from the provenance seed)
   - A genesis provenance mark (sequence 0)

#### Step 3 — Create Ledger and Write Provenance Files

Create a ledger for the XID Document with the following envelope assertions:
- `"GordianOpenIntegrity.SigningKey"`: the SSH public key string
- `"GordianOpenIntegrity.RepositoryIdentifier"`: the repository DID

Advance the provenance mark to sequence 1 (label: `"link-ssh-key"`).

Write the provenance document (§2) to `.o/GordianOpenIntegrity.yaml`.

Write the generator state to `.git/o/GordianOpenIntegrity-generator.yaml` (never committed). Sensitive fields (`seed`, `chainID`, `rngState`) are encrypted using the provenance seed (see §1.2).

#### Step 4 — Commit Inception Envelope

Create a signed commit containing `.o/GordianOpenIntegrity.yaml`:

```
[GordianOpenIntegrity] Establish inception Gordian Envelope at: .o/GordianOpenIntegrity.yaml

Trust established using https://github.com/Stream44/t44-BlockchainCommons.com
```

The commit MUST be SSH-signed with the signing key.

#### Step 5 — Generate LifeHash SVGs

```
identifier = ProvenanceMark.identifier(mark)    # hex string, e.g. "a9ea4602"
lifehash   = LifeHash.makeFromUtf8(identifier, version2)
svg        = lifehash.toSVG()
```

Write to both:
- `.o/GordianOpenIntegrity-InceptionLifehash.svg`
- `.o/GordianOpenIntegrity-CurrentLifehash.svg`

At inception, both files are identical.

#### Expected Git Log

```
commit <hash3> (HEAD -> main)
    [GordianOpenIntegrity] Establish inception Gordian Envelope at: .o/GordianOpenIntegrity.yaml
    Trust established using https://github.com/Stream44/t44-BlockchainCommons.com
    Signed-off-by: <Author> <<email>>

commit <hash2>
    [RepositoryIdentifier] Track <hash1-short>
    Signed-off-by: <Author> <<email>>

commit <hash1>
    [RepositoryIdentifier] Establish signed repository identifier.
    Signed-off-by: <Author> <<email>>
```

---

### 4.2 Introducing a Decision Document

Decision documents add new XID Documents for specific purposes (policies, governance, configurations).

#### Step 1 — Create Document XID

Using the Gordian stack, create a separate XID Document with its own provenance chain (same as §4.1 Step 1 but for the document). Add the self-reference assertion: `"GordianOpenIntegrity.Document": "<document-path>"`

#### Step 2 — Register in Inception Envelope

1. Update the `"GordianOpenIntegrity.Documents"` assertion with: `{ "<document-path>": "XID(<hex>)" }`
2. Advance the inception provenance mark (label: `"introduce:<document-path>"`)
3. Re-serialize the inception envelope to `.o/GordianOpenIntegrity.yaml`

#### Step 3 — Write Document Files

Write the document envelope to `.o/<hostname>/<namespace>/<document>.yaml` and its generator to `.git/o/<hostname>/<namespace>/<document>-generator.yaml`.

#### Step 4 — Commit

```bash
git add .o/GordianOpenIntegrity.yaml .o/<hostname>/<namespace>/<document>.yaml
git -c gpg.format=ssh \
    -c user.signingkey=<private-key-path> \
    commit --gpg-sign --signoff \
    -m "[GordianOpenIntegrity] Introduce new Gordian Envelope at: .o/<hostname>/<namespace>/<document>.yaml"
```

#### Step 5 — Update Current LifeHash

Regenerate `.o/GordianOpenIntegrity-CurrentLifehash.svg` from the new inception mark. The inception LifeHash is NOT updated.

#### Document Path Convention

Implementers can design their own layouts under `.o/<hostname>/`:

```
.o/example.com/policy/v1.yaml          → .git/o/example.com/policy/v1-generator.yaml
.o/myorg.dev/governance/charter.yaml    → .git/o/myorg.dev/governance/charter-generator.yaml
```

---

### 4.3 Key Rotation

Key rotation replaces the SSH signing key while maintaining provenance chain continuity.

#### Step 1 — Generate New SSH Key

```bash
ssh-keygen -t ed25519 -f <key-dir>/<key-name> -N "" -C <key-name>
```

#### Step 2 — Update XID Document

1. Derive a new `PrivateKeyBase` from the new key, add it to the XID Document (`allow: 'All'`)
2. Replace the `"GordianOpenIntegrity.SigningKey"` assertion with the new SSH public key
3. Advance provenance (label: `"add-rotated-key"`)
4. Remove the original inception key from the XID Document
5. Advance provenance (label: `"remove-inception-key"`)

#### Step 3 — Commit (signed with OLD key)

```bash
git add .o/GordianOpenIntegrity.yaml
git -c gpg.format=ssh \
    -c user.signingkey=<old-private-key-path> \
    commit --gpg-sign --signoff \
    -m "[GordianOpenIntegrity] Update GordianOpenIntegrity Gordian Envelope at: .o/GordianOpenIntegrity.yaml"
```

#### Step 4 — Update Current LifeHash

Regenerate `.o/GordianOpenIntegrity-CurrentLifehash.svg`. The inception LifeHash is NOT updated.

All future commits MUST be signed with the new SSH key.

---

### 4.4 Regular Commits

All non-provenance commits MUST be SSH-signed:

```bash
git -c gpg.format=ssh \
    -c user.signingkey=<private-key-path> \
    commit --gpg-sign --signoff \
    -m "<message>"
```

---

### 4.5 Trust Root Reset

A trust root reset creates a new inception envelope while preserving the existing repository identifier. This is a destructive operation that invalidates the previous provenance chain. It is useful when the provenance key is lost or compromised, or when the repository needs a fresh trust foundation.

#### Prerequisites

- An existing repository with a [repository identifier](./WP-2026-01-GitRepository-Identifier.md)
- A new **signing key** and **provenance key** (or the same signing key with a new provenance key)

#### Procedure

1. Read the existing repository identifier from `.repo-identifier`
2. Perform Steps 2–5 of §4.1 (Create XID Document, Create Ledger, Commit Inception Envelope, Generate LifeHash SVGs) using the existing repository DID instead of creating a new one

The previous provenance history remains in the git log but is no longer validated. Verification uses only the **latest** trust root (see §5.1).

#### Important Considerations

- The repository DID does **not** change
- The git history is **not** deleted
- Previous provenance marks become invalid for verification
- All future commits must be signed with the new signing key

---

## 5. Verification

Verification requires only the cloned repository and (optionally) a published provenance mark. No private key material is needed.

A reference shell script implementing the git-level checks is available at [`bin/validate-git.sh`](https://github.com/Stream44/t44-blockchaincommons.com/blob/main/bin/validate-git.sh) in the reference implementation repository.

### 5.1 Repository Verification

#### Step 1 — Collect Provenance History

```bash
git log --all --reverse --format=%H -- .o/GordianOpenIntegrity.yaml
```

For each commit hash, extract the file at that revision:

```bash
git show <hash>:.o/GordianOpenIntegrity.yaml
```

Parse each version per §2.4 to obtain the Gordian Envelope and provenance mark.

#### Step 2 — Verify Published Mark Against Latest Trust Root

Verification uses only the **latest** provenance version (the current trust root). This allows trust root resets (§4.5) without breaking verification.

If a published mark identifier is provided:

```
assert publishedMark == latestVersion.mark.identifier()
```

#### Step 3 — Collect SSH Keys From All Versions

Extract all SSH public keys from the `"GordianOpenIntegrity.SigningKey"` assertion across **all** provenance versions. This supports key rotation — commits signed with previously rotated keys remain valid.

```
for each version in fullHistory:
    collect version.sshKeys (deduplicated)
```

#### Step 4 — Audit Commit Signatures

Write all collected public keys to a temporary allowed-signers file:

```
xid-verifier@provenance namespaces="git" ssh-ed25519 AAAA...
```

Verify every commit:

```bash
git -c gpg.ssh.allowedSignersFile=<tmpfile> verify-commit <hash>
```

A signature is valid if the output contains `Good "git" signature`.

#### Step 5 — XID From Latest Trust Root

The XID is taken from the latest provenance version. Cross-version XID stability is not enforced at this level because trust root resets create a new XID.

#### Strict Mode (Optional)

Two additional checks can be enabled via a `strict` parameter:

- **`repoIdentifierIsInceptionCommit`** — Validates that the `GordianOpenIntegrity.RepositoryIdentifier` assertion in the inception envelope points to the repository's first commit.
- **`signersAllAuthorized`** — Validates that every commit is signed by a key present in at least one provenance version's `GordianOpenIntegrity.SigningKey` assertion.

#### Validity Criteria

| Check | Condition |
|-------|-----------|
| Provenance exists | ≥1 provenance version found |
| Mark matches (if provided) | Published mark = latest mark identifier |
| All signatures valid | Every commit signed by a key from any provenance version |
| Strict: identifier is inception | Repository identifier commit is the first commit (optional) |
| Strict: signers authorized | All commit signing keys appear in provenance history (optional) |

---

### 5.2 Document Verification

In addition to repository-level checks, document verification adds:

#### Step 1 — Collect Document Provenance

Same as §5.1 Steps 1–4 but for the document path.

#### Step 2 — Verify Self-Reference

```
docAssertion = envelope.assertion("GordianOpenIntegrity.Document")
assert docAssertion == documentPath
```

#### Step 3 — Verify Documents Map

From the latest inception envelope:

```
docsMap = JSON.parse(envelope.assertion("GordianOpenIntegrity.Documents"))
assert docsMap[documentPath] == documentXID
```

#### Step 4 — Verify Document XID Stability

Same as §5.1 Step 6 but for the document's XID.

#### Additional Validity Criteria

| Check | Condition |
|-------|-----------|
| Self-reference valid | Document envelope's `GordianOpenIntegrity.Document` matches its path |
| Registry valid | Inception `GordianOpenIntegrity.Documents` map contains document with correct XID |
| Document XID stable | Document XID unchanged across all its versions |

---

## 6. Commit Message Conventions

### 6.1 Repository Identifier Commits

See [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md) for the identifier commit message conventions.

### 6.2 Inception Envelope Commit

Single `-m` with the contract as a second paragraph:

```
-m "[GordianOpenIntegrity] Establish inception Gordian Envelope at: .o/GordianOpenIntegrity.yaml

Trust established using https://github.com/Stream44/t44-BlockchainCommons.com"
```

Flags: `--gpg-sign --signoff`

### 6.3 Document Introduction Commit

```
-m "[GordianOpenIntegrity] Introduce new Gordian Envelope at: .o/<path>"
```

Flags: `--gpg-sign --signoff`

### 6.4 Key Rotation Commit

```
-m "[GordianOpenIntegrity] Update GordianOpenIntegrity Gordian Envelope at: .o/GordianOpenIntegrity.yaml"
```

Flags: `--gpg-sign --signoff`

---

## 7. Verification Failure Modes

| Issue | Cause |
|-------|-------|
| `No provenance documents found in repository history` | `.o/GordianOpenIntegrity.yaml` was never committed |
| `Provenance mark sequence regressed: seq N <= M` | Marks not monotonically increasing |
| `Published mark "X" does not match latest provenance mark "Y"` | Provided mark doesn't match current state |
| `N commit(s) have invalid signatures` | Unsigned or unknown-key commits exist |
| `XID changed across provenance versions` | XID altered between revisions |
| `Document envelope missing GordianOpenIntegrity.Document assertion` | Decision document missing self-reference |
| `Document path assertion "X" does not match expected "Y"` | Self-reference points to wrong path |
| `Documents map XID mismatch for "X"` | Registry has wrong XID for document |
| `Inception envelope missing GordianOpenIntegrity.Documents assertion` | No documents registered |

---

## 8. Security Considerations

### 8.1 Provenance Key and Generator Secrecy

The **provenance key** (Ed25519) serves two purposes:
1. **Initialization** — Its key material seeds the provenance mark generator
2. **Encryption** — It encrypts sensitive fields in generator files at rest (AES-256-GCM)

This ensures that both the provenance key AND the generator file are required to produce subsequent marks — neither is sufficient alone.

The provenance key is distinct from the signing key and MUST be:
- Never reused for any other purpose
- Stored separately from the signing key
- Backed up securely — loss means inability to advance the provenance chain (though a trust root reset via §4.5 can re-establish provenance)

Generator state (`*-generator.yaml`) is equally sensitive. Compromise of both the provenance key and a generator file allows forging future marks. Generators MUST: never be committed, be stored with restrictive permissions, be backed up separately.

### 8.2 SSH Key Management

Keys MUST be Ed25519. During rotation, old keys remain valid for historical commit verification.

### 8.3 Repository Identifier Immutability

Force-pushing over the repository identifier commit changes the DID and invalidates all provenance. See [WP-2026-01-GitRepository-Identifier](./WP-2026-01-GitRepository-Identifier.md).

### 8.4 Provenance Mark Publishing

The published mark SHOULD be distributed independently of the repository (website, blockchain, signed announcement). A mark obtained from the repository itself only proves internal consistency.

### 8.5 SHA-1 Root of Trust

The empty repository identifier commit combined with SSH signature makes SHA-1 collision attacks infeasible for this use case.

### 8.6 Separable Distribution

When envelopes are distributed separately from the repository, verifiers can confirm provenance mark chains and XID stability but cannot audit commit signatures. Full verification requires the complete repository.

---

## 9. Implementation Requirements

### 9.1 Required Capabilities

| Category | Operations |
|----------|-----------|
| **Git** | `init`, `commit --allow-empty --gpg-sign`, `verify-commit`, `log`, `show`, `rev-list`, `rev-parse`, `add` |
| **SSH** | `ssh-keygen -t ed25519`, fingerprint extraction |
| **Gordian Stack** | XID Document (create, serialize, add key, add assertion, provenance mark embed), Gordian Envelope (encode/decode as UR), Provenance Mark (generate, validate sequence, extract identifier), LifeHash (generate SVG from UTF-8) |
| **File I/O** | Read/write `.o/` and `.git/o/` files |

### 9.2 Existing Libraries

| Component | Rust | TypeScript |
|-----------|------|------------|
| XID | [bc-xid](https://crates.io/crates/bc-xid) | [@bcts/xid](https://github.com/nicktomlin/nicktomlin.github.io) |
| Envelope | [bc-envelope](https://crates.io/crates/bc-envelope) | [@bcts/envelope](https://github.com/nicktomlin/nicktomlin.github.io) |
| Provenance Mark | [bc-provenance-mark](https://crates.io/crates/bc-provenance-mark) | [@bcts/provenance-mark](https://github.com/nicktomlin/nicktomlin.github.io) |
| LifeHash | [bc-lifehash](https://crates.io/crates/bc-lifehash) | [@bcts/lifehash](https://github.com/nicktomlin/nicktomlin.github.io) |
| Components | [bc-components](https://crates.io/crates/bc-components) | [@bcts/components](https://github.com/nicktomlin/nicktomlin.github.io) |

### 9.3 Reference Implementation & Tools

- **Implementation Guide**: See companion tool document [WT-2026-03 — Gordian Open Integrity](../Tools/WT-2026-03-GordianOpenIntegrity.md)
- **Git Validation Script**: [`bin/validate-git.sh`](https://github.com/Stream44/t44-blockchaincommons.com/blob/main/bin/validate-git.sh) — a standalone shell script that validates the git-level checks from §5 without requiring the Gordian stack
- **Reference Implementation**: [github.com/Stream44/t44-blockchaincommons.com](https://github.com/Stream44/t44-blockchaincommons.com)
- **Repository**: [github.com/Stream44/Workshop](https://github.com/Stream44/Workshop)
