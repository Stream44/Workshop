# WT-2026-01 — Git Repository Identifier

### Stream44 Studio Workshop Tool

Companion to [WP-2026-01 — Git Repository Identifier](../Patterns/WP-2026-01-GitRepository-Identifier.md)

Authors: Christoph Dorn<br/>
Date: February 15, 2026<br/>
Status: DRAFT — Work in Progress

---

## Overview

This document describes the TypeScript reference implementation of the [Git Repository Identifier Pattern](../Patterns/WP-2026-01-GitRepository-Identifier.md). It maps each specification concept to its concrete capsule method.

The implementation is a standalone capsule with no external dependencies beyond `git` and `ssh-keygen`. It is used by both [GordianOpenIntegrity](./WT-2026-03-GordianOpenIntegrity.md) and [GitRepositoryIntegrity](./WT-2026-02-GitRepositoryIntegrity.md).

---

## 1. Capsule

**File:** `caps/GitRepositoryIdentifier.ts`<br/>
**Capsule name:** `t44/caps/providers/blockchaincommons.com/GitRepositoryIdentifier`

No mapped dependencies — all git operations are handled via internal helpers (`_exec`, `_git`, `_sshKeygen`, `_getFingerprint`, `_ensureGitRepo`).

---

## 2. Constants

```typescript
const IDENTIFIER_FILE = '.repo-identifier'
const IDENTIFIER_MESSAGE_PREFIX = '[RepositoryIdentifier]'
const DEFAULT_IDENTIFIER_MESSAGE = '[RepositoryIdentifier] Establish signed repository identifier.'
```

---

## 3. Methods

### 3.1 `createIdentifier(context)`

Maps to: **Pattern — Step 1 + Step 2**

```typescript
context: {
    repoDir: string
    signingKeyPath: string    // Path to SSH Ed25519 private key
    authorName: string
    authorEmail: string
    message?: string          // Override default identifier message
}
```

Returns:
```typescript
{
    commitHash: string        // SHA-1 hash of the identifier commit
    did: string               // "did:repo:<commitHash>"
    fingerprint: string       // SSH key fingerprint (SHA256:...)
    inceptionDate: Date       // Timestamp of the identifier commit
}
```

Sequence:
1. Ensure git repo exists (`git init` if needed)
2. Get SSH key fingerprint
3. Create signed empty commit with `--allow-empty --no-edit --gpg-sign` and `Signed-off-by` trailer. `GIT_COMMITTER_NAME` is set to the key fingerprint.
4. Write `did:repo:<hash>` to `.repo-identifier`
5. Create signed file commit with `--gpg-sign --signoff` tracking the identifier hash

### 3.2 `getIdentifier(context)`

Maps to: **Pattern — Retrieving the Current Identifier**

```typescript
context: { repoDir: string }
```

Returns: `{ commitHash, did }` — O(1) file read from `.repo-identifier`.

### 3.3 `getIdentifiers(context)`

Maps to: **Pattern — Listing All Identifiers**

```typescript
context: { repoDir: string }
```

Returns: `{ identifiers: Array<{ commitHash, did }>, count }` — queries `git log --follow -- .repo-identifier`.

### 3.4 `validateIdentifier(context)`

Maps to: **Pattern — Identifier Validation**

```typescript
context: {
    repoDir: string
    commitHash?: string    // Validate specific identifier (defaults to current)
}
```

Returns:
```typescript
{
    valid: boolean           // true if all four checks pass
    commitHash: string
    did: string
    isSigned: boolean        // gpgsig field present
    isEmpty: boolean         // no file changes in commit tree
    authorMatch: boolean     // same author name/email in both commits
    keyMatch: boolean        // same SSH signing key in both commits
    keyFingerprint: string
    authorName: string
    authorEmail: string
}
```

Validates four properties per the pattern:
1. **Signed** — identifier commit contains `gpgsig` field
2. **Empty** — identifier commit tree matches parent tree (or empty tree for root)
3. **Author match** — identifier and file commits share author name/email
4. **Key match** — both commits signed with the same SSH key (compared via signature block)

---

## 4. Test Suite

**File:** `caps/GitRepositoryIdentifier.test.ts`

| Test Group | Coverage |
|-----------|----------|
| 1. Create identifier on new repo | Inception commit, `.repo-identifier` file, DID format |
| 2. Create identifier on existing repo | Identifier after prior commits |
| 3. Multiple identifiers | Sequential identifiers, listing, validation |
| 4. Validation edge cases | Non-empty commit detection |

---

## 5. Cross-References

- **Pattern:** [WP-2026-01 — Git Repository Identifier](../Patterns/WP-2026-01-GitRepository-Identifier.md)
- **Used by:** [WT-2026-02 — Git Repository Integrity](./WT-2026-02-GitRepositoryIntegrity.md) (Layer 2 validation)
- **Used by:** [WT-2026-03 — Gordian Open Integrity](./WT-2026-03-GordianOpenIntegrity.md) (`createRepository` delegates to `createIdentifier`)
