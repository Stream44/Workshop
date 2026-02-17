# WT-2026-02 — Git Repository Integrity

### Stream44 Studio Workshop Tool

Companion to [WP-2026-02 — Git Repository Integrity](../Patterns/WP-2026-02-GitRepository-Integrity.md)

Authors: Christoph Dorn<br/>
Date: February 15, 2026<br/>
Status: DRAFT — Work in Progress

---

## Overview

This document describes the TypeScript reference implementation of the [Git Repository Integrity Pattern](../Patterns/WP-2026-02-GitRepository-Integrity.md). It implements all four validation layers defined in the pattern.

The implementation is a standalone capsule that delegates to [GitRepositoryIdentifier](./WT-2026-01-GitRepositoryIdentifier.md) for Layer 2 and uses the Gordian cryptographic stack (XID, Envelopes, Provenance Marks) for Layer 3. It is used by [GordianOpenIntegrity](./WT-2026-03-GordianOpenIntegrity.md) for verification operations.

---

## 1. Capsule

**File:** `caps/GitRepositoryIntegrity.ts`<br/>
**Capsule name:** `t44/caps/providers/blockchaincommons.com/GitRepositoryIntegrity`

### 1.1 Mapped Dependencies

| Mapping | Capsule | Purpose |
|---------|---------|---------|
| `repoId` | `GitRepositoryIdentifier` | Layer 2 identifier validation |
| `xid` | `xid` | XID Document deserialization, assertion extraction |
| `ledger` | `XidDocumentLedger` | Provenance YAML parsing |
| `provenanceMark` | `provenance-mark` | Mark sequence validation |

### 1.2 Constants

```typescript
const EMPTY_TREE_HASH = '4b825dc642cb6eb9a060e54bf8d69288fbee4904'
const PROVENANCE_FILE = '.o/GordianOpenIntegrity.yaml'
const ASSERTION_SIGNING_KEY = 'GordianOpenIntegrity.SigningKey'
const ASSERTION_REPO_ID = 'GordianOpenIntegrity.RepositoryIdentifier'
const ASSERTION_DOCUMENT = 'GordianOpenIntegrity.Document'
const ASSERTION_DOCUMENTS = 'GordianOpenIntegrity.Documents'
```

---

## 2. Layer 1 — Commit Origin

### 2.1 `validateCommitSignatures(context)`

Maps to: **Pattern Layer 1.1**

```typescript
context: {
    repoDir: string
    allowedSigners?: Array<{ email: string; publicKey: string }>
}
```

Without `allowedSigners`: checks each commit's raw object for `gpgsig` field.
With `allowedSigners`: writes temporary allowed-signers file and runs `git verify-commit` for each commit.

### 2.2 `validateCommitSignoffs(context)`

Maps to: **Pattern Layer 1.2**

```typescript
context: { repoDir: string }
```

Checks every commit message for `Signed-off-by:` trailer.

### 2.3 `validateAuthorConsistency(context)`

Maps to: **Pattern Layer 1.3**

```typescript
context: { repoDir: string }
```

Groups commits by signing key fingerprint and verifies each key maps to exactly one author identity.

---

## 3. Layer 2 — Repository Identifier

### 3.1 `validateRepositoryIdentifier(context)`

Maps to: **Pattern Layer 2.1**

```typescript
context: { repoDir: string }
```

Delegates to `repoId.validateIdentifier({ repoDir })`. Returns the full validation result including `valid`, `did`, `isSigned`, `isEmpty`, `authorMatch`, `keyMatch`.

---

## 4. Layer 3 — Gordian Open Integrity

### 4.1 `verify(context)`

Maps to: **Pattern Layer 3.1 — Repository Verification**

```typescript
context: {
    repoDir: string
    mark?: string                    // Published mark hex (optional)
    strict?: {
        signersAllAuthorized?: boolean
        repoIdentifierIsInceptionCommit?: boolean
    }
}
```

Returns:
```typescript
{
    valid: boolean
    xid: string
    did: string
    marksMonotonic: boolean
    markMatchesLatest: boolean
    xidStable: boolean
    totalCommits: number
    validSignatures: number
    invalidSignatures: number
    provenanceVersions: number
    issues: string[]
}
```

Steps:
1. Collect provenance history from `git log -- .o/GordianOpenIntegrity.yaml`
2. Parse each version via `ledger.parseProvenanceYaml` → XID Document + Provenance Mark
3. Verify mark sequence monotonicity
4. Compare published mark against latest (if provided)
5. Collect SSH keys from `GordianOpenIntegrity.SigningKey` assertions
6. Audit all commit signatures against collected keys
7. Verify XID stability across versions
8. If `strict.signersAllAuthorized`: Layer 4.1 check
9. If `strict.repoIdentifierIsInceptionCommit`: Layer 4.2 check

### 4.2 `verifyDocument(context)`

Maps to: **Pattern Layer 3.2 — Document Verification**

```typescript
context: {
    repoDir: string
    documentPath: string
    mark?: string
    strict?: { ... }
}
```

Additional checks beyond `verify`:
- `GordianOpenIntegrity.Document` assertion matches `documentPath`
- `GordianOpenIntegrity.Documents` map on inception envelope contains correct XID
- Document XID stability across versions
- Combined SSH key collection from both document and inception histories

---

## 5. Layer 4 — XID Document Governance

Layer 4 checks are enabled via the `strict` parameter on `verify`, `verifyDocument`, and `validate`.

### 5.1 Signer Authorization (`strict.signersAllAuthorized`)

Verifies every commit is signed by a key present in at least one provenance version's `GordianOpenIntegrity.SigningKey` assertion.

### 5.2 Repository Identifier Binding (`strict.repoIdentifierIsInceptionCommit`)

Verifies the `GordianOpenIntegrity.RepositoryIdentifier` assertion in the inception envelope matches the repository's root commit (`git rev-list --max-parents=0 HEAD`).

---

## 6. Comprehensive Validation

### 6.1 `validate(context)`

Maps to: **Pattern — Comprehensive Validation**

```typescript
context: {
    repoDir: string
    strict?: { ... }
}
```

Runs all applicable layer checks and returns a structured report:

```typescript
{
    valid: boolean
    layers: {
        commitSignatures: { ... }
        commitSignoffs: { ... }
        authorConsistency: { ... }
        repositoryIdentifier: { ... }
        provenanceChain: { ... }          // if provenance exists
        xidStability: { ... }             // if provenance exists
        repoIdentifierIsInceptionCommit: { ... }  // strict only
        signerAuthorization: { ... }               // strict only
    }
    totalCommits: number
    signedCommits: number
    unsignedCommits: number
    issues: string[]
}
```

---

## 7. Internal Helpers

| Helper | Purpose |
|--------|---------|
| `_listCommits({ repoDir })` | `git log --all --format=...` — returns all commits with metadata |
| `_auditSignatures({ repoDir, allowedSigners })` | Writes temp allowed-signers file, runs `git verify-commit` per commit |
| `_collectProvenanceHistory({ repoDir, documentPath })` | Walks git log for provenance file, parses each version |
| `_git({ args, cwd, env })` | Spawns git subprocess |
| `_exec({ cmd, cwd, env })` | Spawns arbitrary subprocess |

---

## 8. Test Suite

**File:** `caps/GitRepositoryIntegrity.test.ts`

| Test Group | Coverage |
|-----------|----------|
| Layer 1 — Commit Signatures | Signed/unsigned commit detection, allowed signers verification |
| Layer 1 — Commit Sign-offs | Sign-off trailer presence |
| Layer 1 — Author Consistency | Same key → same author, multi-author detection |
| Layer 2 — Repository Identifier | Valid identifier, missing identifier |
| Layer 3 — Provenance Chain | Valid chain, wrong mark, no provenance |
| Layer 3 — XID Stability | Stable XID after key rotation |
| Layer 3 — Document Verification | Valid document, non-existent document, wrong mark |
| Layer 4 — signersAllAuthorized | Authorized signers, unauthorized signer detection |
| Layer 4 — repoIdentifierIsInceptionCommit | Identifier binding validation |
| Layer 4 — Combined strict | Both strict flags together |
| Comprehensive — validate | Full validation, strict mode, missing identifier, unsigned commits |

---

## 9. Cross-References

- **Pattern:** [WP-2026-02 — Git Repository Integrity](../Patterns/WP-2026-02-GitRepository-Integrity.md)
- **Depends on:** [WT-2026-01 — Git Repository Identifier](./WT-2026-01-GitRepositoryIdentifier.md) (Layer 2)
- **Used by:** [WT-2026-03 — Gordian Open Integrity](./WT-2026-03-GordianOpenIntegrity.md) (delegates all verification)
