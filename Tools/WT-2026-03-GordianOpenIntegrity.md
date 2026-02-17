# WT-2026-03 — Gordian Open Integrity

### Stream44 Studio Workshop Tool

Companion to [WP-2026-03 — Gordian Open Integrity](../Patterns/WP-2026-03-GitRepository-GordianOpenIntegrity.md)

Authors: Christoph Dorn<br/>
Date: February 15, 2026<br/>
Status: DRAFT — Work in Progress

---

## Overview

This document describes the TypeScript reference implementation of the [Gordian Open Integrity Pattern](../Patterns/WP-2026-03-GitRepository-GordianOpenIntegrity.md). It maps each specification concept to its concrete capsule, function, and data structure.

The implementation uses the [Encapsulate](https://github.com/Stream44/encapsulate) capsule framework, where each module ("capsule") declares typed properties and dependency mappings. All capsules live under `caps/` in the [t44-blockchaincommons.com](https://github.com/Stream44/t44-blockchaincommons.com) package.

**Main Implementation:** [`caps/GordianOpenIntegrity.ts`](https://github.com/Stream44/t44-blockchaincommons.com/blob/main/caps/GordianOpenIntegrity.ts)

---

## 1. Architecture

### 1.1 Capsule Dependency Graph

```
GordianOpenIntegrity
├── xid                      (caps/xid.ts)
├── XidDocumentLedger        (caps/XidDocumentLedger.ts)
│   ├── xid
│   └── provenance-mark
├── GitRepositoryIdentifier  (caps/GitRepositoryIdentifier.ts)
├── GitRepositoryIntegrity   (caps/GitRepositoryIntegrity.ts)
│   ├── GitRepositoryIdentifier
│   ├── xid
│   ├── XidDocumentLedger
│   └── provenance-mark
├── provenance-mark          (caps/provenance-mark.ts)
└── lifehash                 (caps/lifehash.ts)
```

### 1.2 Capsule → Pattern Mapping

| Capsule | Pattern Section | Role |
|---------|----------------|------|
| `GordianOpenIntegrity` | WP-2026-03 §4, §5 | Top-level orchestrator: init, introduce, rotate, verify |
| `GitRepositoryIdentifier` | WP-2026-01 | Repository identifier creation and validation. See [WT-2026-01](./WT-2026-01-GitRepositoryIdentifier.md). |
| `GitRepositoryIntegrity` | WP-2026-02 | Multi-layer integrity validation. See [WT-2026-02](./WT-2026-02-GitRepositoryIntegrity.md). |
| `XidDocumentLedger` | WP-2026-03 §2, §4 (file writes) | Manages revision chains of XID Documents, writes `.o/*.yaml` and `.git/o/*-generator.yaml` |
| `xid` | WP-2026-03 §3 (envelope assertions) | XID Document CRUD, envelope serialization, assertion management |
| `provenance-mark` | WP-2026-03 §2 (mark field), §5 (monotonicity) | Provenance mark generation, validation, identifier extraction |
| `lifehash` | WP-2026-03 §4.1 Step 5 | LifeHash SVG generation from mark identifiers |

### 1.3 External Dependencies

| npm Package | Gordian Component |
|-------------|-------------------|
| `@bcts/xid` | XID Documents, keys, delegates, privileges |
| `@bcts/components` | `PrivateKeyBase`, `PublicKeys`, `PrivateKeys` |
| `@bcts/provenance-mark` | `ProvenanceMarkGenerator`, `ProvenanceMark`, validation |
| `@bcts/lifehash` | `LifeHash`, `Version` |
| `@bcts/envelope` | Gordian Envelope encode/decode |

---

## 2. Constants

Defined at the top of `caps/GordianOpenIntegrity.ts`:

```typescript
const PROVENANCE_FILE        = '.o/GordianOpenIntegrity.yaml'
const GENERATOR_FILE         = '.git/o/GordianOpenIntegrity-generator.yaml'
const ASSERTION_SIGNING_KEY  = 'GordianOpenIntegrity.SigningKey'
const ASSERTION_REPO_ID      = 'GordianOpenIntegrity.RepositoryIdentifier'
const ASSERTION_DOCUMENT     = 'GordianOpenIntegrity.Document'
const ASSERTION_DOCUMENTS    = 'GordianOpenIntegrity.Documents'
const CONTRACT               = 'Trust established using https://github.com/Stream44/t44-BlockchainCommons.com'
const INCEPTION_LIFEHASH_FILE = '.o/GordianOpenIntegrity-InceptionLifehash.svg'
const CURRENT_LIFEHASH_FILE  = '.o/GordianOpenIntegrity-CurrentLifehash.svg'
```

These correspond directly to the file paths and assertion predicates defined in Pattern §1 and §3.

---

## 3. XidDocumentLedger

The `XidDocumentLedger` capsule (`caps/XidDocumentLedger.ts`) is an implementation-level abstraction not present in the pattern. It manages the lifecycle of a single XID Document's provenance chain.

### 3.1 Data Structures

```typescript
interface Revision {
    seq: number          // Provenance mark sequence (0-indexed)
    label: string        // Human-readable label (e.g. "genesis", "link-ssh-key")
    date: Date           // Timestamp from the provenance mark
    document: any        // Cloned XIDDocument snapshot at this revision
    mark: any            // ProvenanceMark instance
}

interface Ledger {
    revisions: Revision[]
    xid: any                    // XID instance (stable across revisions)
    storeDir?: string           // Optional CLI-compatible mark storage directory
    documentPath?: string       // Absolute path to .o/*.yaml
    generatorPath?: string      // Absolute path to .git/o/*-generator.yaml
    assertions?: Array<{ predicate: string; object: string }>
    contract?: string
    inceptionMarkId?: string    // Bytewords identifier of inception mark
    inceptionMarkHex?: string   // Hex identifier of inception mark
    repositoryDid?: string
}
```

### 3.2 Methods

| Method | Pattern Operation | Description |
|--------|------------------|-------------|
| `createLedger({ document, documentPath, generatorPath, assertions, contract, repositoryDid })` | §4.1 Step 3, §4.2 Step 3 | Creates a new ledger from a XID Document. Writes the provenance YAML and generator state to disk. The first revision is labeled `"genesis"`. |
| `commit({ ledger, document, label, date })` | §4.1 Step 1.4, §4.2 Step 2 | Advances the provenance mark, creates a new revision, updates YAML and generator files on disk. Verifies XID stability before committing. |
| `verify({ ledger })` | §5.1, §5.2 | Checks: XID stability, genesis mark present, chain integrity (`precedes()`), sequence validity, date monotonicity. |
| `buildProvenanceYaml({ document, mark, assertions, ... })` | §2 | Serializes the XID Document → Envelope → UR string, builds the YAML+comments format per Pattern §2. |
| `parseProvenanceYaml({ yaml })` | §2.4 | Parses a `.o/*.yaml` file back into `{ urString, mark }`. |
| `getLatest`, `getGenesis`, `getRevision`, `getLabels`, `getMarks` | — | Query helpers for ledger state. |

### 3.3 File Write Behavior

When `documentPath` is set, `createLedger` and `commit` automatically write:

1. **Provenance YAML** to `documentPath` (e.g. `/repo/.o/GordianOpenIntegrity.yaml`)
2. **Generator JSON** to `generatorPath` (e.g. `/repo/.git/o/GordianOpenIntegrity-generator.yaml`)

The YAML is built by `buildProvenanceYaml` which:
1. Converts `XIDDocument → Envelope` (omitting private keys and generator via `XIDPrivateKeyOptions.Omit`, `XIDGeneratorOptions.Omit`)
2. Adds custom assertions (SSH key, Documents map) to the envelope
3. Extracts the UR string and mark identifier
4. Formats the YAML section + `---` separator + human-readable comments

---

## 4. GordianOpenIntegrity Capsule

The top-level capsule (`caps/GordianOpenIntegrity.ts`) orchestrates all operations from the pattern.

### 4.1 Author Methods (require private key material)

#### `createDocument(context)`

Creates a standalone XID Document for use with `introduceDocument`:

```typescript
context: {
    documentKeyPath: string       // Path to Ed25519 key for XID derivation (REQUIRED)
    provenanceKeyPath: string     // Path to Ed25519 key for provenance seed (REQUIRED)
    provenanceDate?: Date
}
```

Internally: derives `PrivateKeyBase` from `documentKeyPath` via SHA-256 hash, derives provenance seed from `provenanceKeyPath`, then creates XID Document with provenance mark.

#### `createRepository(context)`

Maps to: **Pattern §4.1 Steps 1–5**

```typescript
context: {
    repoDir: string
    authorName: string
    authorEmail: string
    firstTrustKeyPath: string     // Path to SSH Ed25519 private key (REQUIRED)
    provenanceKeyPath: string     // Path to Ed25519 key for provenance seed (REQUIRED)
    contract?: string
    blockchainCommonsCompatibility?: boolean
}
```

Sequence:
1. `repoId.createIdentifier(...)` → repository identifier commit + `.repo-identifier` (Pattern §4.1 Step 1)
2. Delegates to `createTrustRoot(...)` with the newly created identifier (Pattern §4.1 Steps 2–5)

Returns: `{ did, commitHash, mark, author }`

The `author` object is returned for use with subsequent operations (`rotateTrustSigningKey`, `introduceDocument`).

#### `createTrustRoot(context)`

Maps to: **Pattern §4.5 (Trust Root Reset)** and internally used by `createRepository`

```typescript
context: {
    repoDir: string
    authorName: string
    authorEmail: string
    firstTrustKeyPath: string     // Path to SSH Ed25519 private key (REQUIRED)
    provenanceKeyPath: string     // Path to Ed25519 key for provenance seed (REQUIRED)
    contract?: string
    existingIdentifier?: { commitHash: string; did: string; inceptionDate?: Date }
}
```

If `existingIdentifier` is not provided, reads the current identifier from `.repo-identifier`.

Sequence:
1. `createDocument({ documentKeyPath, provenanceKeyPath })` → XID Document (Pattern §4.1 Step 2)
2. `ledger.createLedger(...)` with `ASSERTION_SIGNING_KEY` and `ASSERTION_REPO_ID` assertions → writes `.o/GordianOpenIntegrity.yaml` and generator (Pattern §4.1 Step 3)
3. `ledger.commit({ label: 'link-ssh-key' })` → advances mark to seq 1
4. `git.createSignedCommit(...)` → commits the inception envelope (Pattern §4.1 Step 4)
5. `_writeLifehashes({ inception: true })` → writes both SVGs (Pattern §4.1 Step 5)

Returns: `{ did, commitHash, mark, author }`

#### `rotateTrustSigningKey(context)`

Maps to: **Pattern §4.3**

```typescript
context: {
    repoDir: string
    authorName: string
    authorEmail: string
    existingSigningKeyPath: string  // Path to current SSH key (REQUIRED)
    newSigningKeyPath: string       // Path to new SSH key (REQUIRED)
    author: any                     // From createRepository
}
```

Sequence:
1. Read new SSH key from `newSigningKeyPath` (Pattern §4.3 Step 1)
2. `xid.addKey(...)` + update `ASSERTION_SIGNING_KEY` + `ledger.commit({ label: 'add-rotated-key' })` (Pattern §4.3 Step 2)
3. `xid.removeInceptionKey(...)` + `ledger.commit({ label: 'remove-inception-key' })` (Pattern §4.3 Step 2)
4. `git.createSignedCommit(...)` signed with **existing** key (Pattern §4.3 Step 3)
5. `_writeLifehashes(...)` → updates current SVG (Pattern §4.3 Step 4)
6. Updates `author.sshKey` to the new key

Returns: `{ author }` with updated signing key.

#### `introduceDocument(context)`

Maps to: **Pattern §4.2**

```typescript
context: {
    repoDir: string
    authorName: string
    authorEmail: string
    trustKeyPath: string      // Path to current trust signing key (REQUIRED)
    provenanceKeyPath: string // Path to provenance key for encryption (REQUIRED)
    document: any             // From createDocument
    documentPath: string      // e.g. ".o/example.com/policy/v1.yaml"
    generatorPath: string     // e.g. ".git/o/example.com/policy/v1-generator.yaml"
    author: any               // From createRepository
    label?: string
    trustUpdateMessage?: string
}
```

Sequence:
1. Build Documents map, update `ASSERTION_DOCUMENTS` on `author.ledger.assertions` (Pattern §4.2 Step 2)
2. `ledger.commit({ label: 'introduce:<path>' })` on inception ledger (Pattern §4.2 Step 2)
3. `ledger.createLedger(...)` for the document with `ASSERTION_DOCUMENT` (Pattern §4.2 Step 1, Step 3)
4. `ledger.commit(...)` on document ledger (Pattern §4.2 Step 3)
5. `git.createSignedCommit(...)` with both files (Pattern §4.2 Step 4)
6. `_writeLifehashes(...)` → updates current SVG (Pattern §4.2 Step 5)

#### `commitToRepository(context)`

Creates a signed commit for non-provenance changes (Pattern §4.4).

```typescript
context: {
    repoDir: string
    authorName: string
    authorEmail: string
    signingKeyPath: string
    message: string
    files?: Array<{ path: string; content: string }>
}
```

### 4.2 Verifier Methods (no private key material needed)

#### `verify(context)`

Maps to: **Pattern §5.1**

```typescript
context: {
    repoDir: string
    mark?: string   // Published mark hex (optional)
}
```

Delegates to `integrity.verify(context)` on the `GitRepositoryIntegrity` capsule. See [WT-2026-02](./WT-2026-02-GitRepositoryIntegrity.md) for the full verification flow.

Key behavior:
- Validates published mark against the **latest** trust root only (supports trust root resets)
- Collects SSH keys from **all** provenance versions (supports key rotation)
- XID taken from latest entry

Returns the result object described in Pattern §5.1 Validity Criteria.

#### `verifyDocument(context)`

Maps to: **Pattern §5.2**

```typescript
context: {
    repoDir: string
    documentPath: string
    mark?: string
}
```

Additional steps beyond `verify`:
- Checks `ASSERTION_DOCUMENT` matches `documentPath` (Pattern §5.2 Step 2)
- Checks `ASSERTION_DOCUMENTS` map on inception envelope (Pattern §5.2 Step 3)
- Collects SSH keys from **both** document and inception histories for signature audit

### 4.3 Internal Helpers

#### `_collectProvenanceHistory({ repoDir, documentPath })`

Core helper that implements Pattern §5.1 Step 1:

```
git log --all --reverse --format=%H -- <documentPath>
```

For each commit hash:
1. `git show <hash>:<documentPath>` → raw YAML
2. `ledger.parseProvenanceYaml(...)` → `{ urString, mark }`
3. `xid.envelopeFromUrString(...)` → Envelope
4. `xid.fromEnvelope(...)` → XIDDocument
5. `xid.getEnvelopeAssertions({ predicate: ASSERTION_SIGNING_KEY })` → SSH keys
6. `xid.getProvenance(...)` → ProvenanceMark

Returns: `Array<{ xid, sshKeys, document, envelope, mark }>`

#### `_writeLifehashes({ repoDir, mark, inception? })`

1. `provenanceMark.getIdentifier(...)` → hex string
2. `lifehash.makeFromUtf8(...)` → LifeHash image
3. `lifehash.toSVG(...)` → SVG string
4. Write to `CURRENT_LIFEHASH_FILE` (always) and `INCEPTION_LIFEHASH_FILE` (only when `inception: true`)

---

## 5. Low-Level Mapped Capsules

`GordianOpenIntegrity` delegates git, key, and file operations to three low-level capsules:

| Mapping | Capsule | Purpose |
|---------|---------|---------|
| `git` | `git` | Git command execution (`createSignedCommit`, `run`, `ensureRepo`, `listCommits`, `auditSignatures`) |
| `key` | `key` | SSH key reading (`readSigningKey`), key derivation (`deriveKeyBase`, `deriveProvenanceSeed`), key generation (`generateSigningKey`) |
| `fs` | `fs` | File I/O (`readFile`, `writeFile`, `mkdir`, `join`) |

Repository identifier operations are delegated to the `GitRepositoryIdentifier` capsule (see [WT-2026-01](./WT-2026-01-GitRepositoryIdentifier.md)), and integrity validation is delegated to the `GitRepositoryIntegrity` capsule (see [WT-2026-02](./WT-2026-02-GitRepositoryIntegrity.md)).

### 5.1 Key Derivation

Two methods on the `key` capsule derive deterministic keys from Ed25519 key files:

| Method | Purpose |
|--------|---------|
| `key.deriveKeyBase({ keyPath })` | SHA-256 hash of key file data → `PrivateKeyBase.fromData()` for XID key material |
| `key.deriveProvenanceSeed({ keyPath })` | SHA-256 hash of key file data → `Uint8Array` for provenance mark seeding and generator encryption |

This approach ensures deterministic XID and provenance generation from the same key files.

---

## 6. Low-Level Capsules

### 6.1 `xid` Capsule (`caps/xid.ts`)

Wraps `@bcts/xid`. Key methods used by GordianOpenIntegrity:

| Method | Usage |
|--------|-------|
| `createDocument({ keyType, privateKeyBase, provenance })` | Create XID Document with key and provenance |
| `getXid({ document })` | Extract XID identifier |
| `getProvenance({ document })` | Extract `{ mark, generator }` |
| `advanceProvenance({ document, date })` | Generate next provenance mark |
| `cloneDocument({ document })` | Deep-copy for revision snapshots |
| `toEnvelope({ document, privateKeyOptions, generatorOptions })` | Serialize to Gordian Envelope |
| `fromEnvelope({ envelope })` | Deserialize from Envelope |
| `envelopeFromUrString({ urString })` | Decode UR string to Envelope |
| `addEnvelopeAssertion({ envelope, predicate, object })` | Add assertion to envelope |
| `getEnvelopeAssertions({ envelope, predicate })` | Query assertions by predicate |
| `addKey({ document, publicKeys, allowAll })` | Add key for rotation |
| `removeInceptionKey({ document })` | Remove inception key during rotation |
| `types()` | Access `PrivateKeyBase`, `XIDPrivateKeyOptions`, `XIDGeneratorOptions` |

### 6.2 `provenance-mark` Capsule (`caps/provenance-mark.ts`)

Wraps `@bcts/provenance-mark`:

| Method | Usage |
|--------|-------|
| `getSeq({ mark })` | Extract sequence number |
| `getDate({ mark })` | Extract timestamp |
| `getIdentifier({ mark })` | Hex identifier (first 4 bytes) |
| `getBytewordsIdentifier({ mark })` | Bytewords identifier (e.g. "🅑 WARM ARCH RAMP HORN") |
| `isGenesis({ mark })` | Check if sequence 0 |
| `precedes({ mark, next })` | Verify chain linkage |
| `isSequenceValid({ marks })` | Validate full mark sequence |
| `validate({ marks })` | Full validation report |
| `hasIssues({ report })` | Check report for problems |

### 6.3 `lifehash` Capsule (`caps/lifehash.ts`)

Wraps `@bcts/lifehash`:

| Method | Usage |
|--------|-------|
| `makeFromUtf8({ input })` | Generate LifeHash from UTF-8 string (version2) |
| `toSVG({ image })` | Convert LifeHash to SVG string |
| `toPPM({ image })` | Convert LifeHash to PPM format |

---

## 7. CLI

The CLI (`bin/oi`) boots the capsule framework and exposes two commands:

### 7.1 `init`

```bash
bunx @stream44.studio/t44-blockchaincommons.com init [GordianOpenIntegrity] --first-trust-key <path> --provenance-key <path>
```

Flow: `createRepository({ repoDir: cwd, firstTrustKeyPath, provenanceKeyPath })` → print DID + mark.

Author name/email default to `git config user.name` / `git config user.email`.

### 7.2 `validate`

```bash
bunx @stream44.studio/t44-blockchaincommons.com validate [GordianOpenIntegrity] [--mark <mark>]
```

Flow: `verify({ repoDir: cwd, mark })` → print result. Exit code 1 on failure.

If `--mark` is not provided, the action reads it from `.o/GordianOpenIntegrity.yaml`.

---

## 8. GitHub Action

`action.yml` runs as a composite action:

1. `oven-sh/setup-bun@v2`
2. `bun install --frozen-lockfile`
3. Read mark from `.o/GordianOpenIntegrity.yaml` if not provided
4. `bun bin/oi validate GordianOpenIntegrity --mark <mark>`

Requires `fetch-depth: 0` in the checkout step.

---

## 9. Validation Script

A standalone shell script [`bin/validate-git.sh`](https://github.com/Stream44/t44-blockchaincommons.com/blob/main/bin/validate-git.sh) implements all git-level verification checks from Pattern §5 that can be performed without the Gordian cryptographic stack. See the script header for a detailed breakdown of covered vs. uncovered checks.

The script is located in the reference implementation repository at `bin/validate-git.sh` and is tested as part of the integration test suite in `examples/03-GordianOpenIntegrity/main.test.ts`.

---

## 10. Test Suite

### 10.1 Unit Tests (`caps/GordianOpenIntegrity.test.ts`)

| Test Group | Coverage |
|-----------|----------|
| 1. createDocument | Standalone document creation |
| 2. createRepository | Repository identifier, inception envelope, file creation, DID, author object |
| 3. verify | Repository verification with/without mark, wrong mark detection |
| 4. rotateTrustSigningKey | Key rotation, SSH key replacement, verification after rotation |
| 5. introduceDocument | Document registration, Documents map assertion, commit |
| 6. commitToRepository | Regular signed commits, empty commits |
| 7. createTrustRoot | Trust root reset preserving DID, verification after reset, old mark invalidation |
| 8. Utility methods | `provenancePath`, `getMarkIdentifier`, `getLatestMark` |

### 10.2 Integration Tests

- `examples/03-GordianOpenIntegrity/main.test.ts` — Happy-path: repository → verify → introduce document → verify document
- `examples/04-GordianOpenIntegrityCli/main.test.ts` — CLI `init` and `validate` commands

### 10.3 Supporting Tests

- `caps/GitRepositoryIdentifier.test.ts` — Identifier creation, retrieval, validation
- `caps/GitRepositoryIntegrity.test.ts` — Multi-layer integrity validation (Layers 1–4)
- `caps/XidDocumentLedger.test.ts` — Ledger creation, commit, verify, YAML round-trip
- `caps/xid.test.ts` — XID Document CRUD, envelope serialization
- `caps/provenance-mark.test.ts` — Mark generation, validation, chain integrity
- `caps/lifehash.test.ts` — LifeHash generation, SVG output
- `caps/open-integrity-sh.test.ts` — SH tool cross-validation with GordianOpenIntegrity

---

## 11. Revision Labels

The implementation uses these labels for provenance mark revisions:

| Label | Operation | Ledger |
|-------|-----------|--------|
| `genesis` | Initial document creation | Both |
| `link-ssh-key` | SSH key binding | Inception |
| `add-rotated-key` | New key added during rotation | Inception |
| `remove-inception-key` | Old key removed during rotation | Inception |
| `introduce:<path>` | Document registered in inception | Inception |
| `<custom>` | User-provided label for document revisions | Document |